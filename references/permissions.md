# Дозволи у Ґрунт

Ґрунт використовує **Role-Based Access Control (RBAC)** — дозволи прив'язані до ролей, а ролі — до користувачів.

## Permissions у DocType JSON

Визначаються в масиві `permissions` у JSON файлі DocType. Кожен рядок — роль і **список дій**
`actions`:

```json
"permissions": [
  {"role": "Sales", "actions": ["read", "write", "create", "print", "share"], "match": "owner == user"},
  {"role": "Manager", "actions": ["read", "write", "create", "delete", "export", "print", "share", "report"]},
  {"role": "Viewer", "actions": ["read", "print"], "hidden_fields": ["salary"]},
  {"role": "System Manager", "actions": ["read", "write", "create", "delete", "export", "import", "print", "share", "report"]}
]
```

| Поле | Опис |
|------|------|
| `role` | Назва ролі (має існувати в системі) |
| `actions` | Дозволені дії — список з переліку нижче |
| `match` | Row-level фільтр: `<field> == user`, `<field> != 'значення'`, `<link_field>.<field> == user` (`user` — email поточного користувача) |
| `hidden_fields` | Поля, приховані від ролі: не читаються, не пишуться, не фільтруються/сортуються/шукаються |
| `readonly_fields` | Поля, які роль бачить, але не може змінити в наявному документі (новий документ значення приймає). Діє лише в рядках з `write` |

**Дії** (`actions`):

| Дія | Що дає |
|-----|--------|
| `read` | Читання документів (включає `select`) |
| `select` | Лише вибір у Link-полях (name, title, поля пошуку) |
| `write` | Редагування |
| `create` | Створення |
| `delete` | Видалення |
| `export` | Експорт у файл/CSV, DataExport |
| `import` | DataImport, шаблон імпорту |
| `print` | Друк і попередній перегляд |
| `share` | Поділитися документом (SharedWith, посилання) |
| `report` | Звіти, що мають `ref_doctype` на цей DocType; «Створити звіт» у списку |

Правила формату:

- **Старі булеві ключі** (`"read": true`, `"write": 1`, …) — помилка валідації: DocType з ними не
  завантажиться («list them as actions»). Завжди лише `actions`.
- Невідома дія, невідомий `match` або назва поля в `hidden_fields`/`readonly_fields`, якої немає
  в DocType, — теж помилка (одруківка не повинна мовчки відмовляти чи відкривати поле).
- `export`, `import`, `print`, `share`, `report` **не випливають** з `read`/`write` — видавай їх
  явно. Типова схема: `print` — кожній ролі з `read`; `export` — разом з `report`; `share` — з
  `write`; `import` — лише адміністративним ролям.
- Кілька рядків однієї ролі / кількох ролей користувача поєднуються через OR.

------|------|
| `role` | Назва ролі (має існувати в системі) |
| `read` | Читання документів |
| `write` | Редагування документів |
| `create` | Створення нових документів |
| `delete` | Видалення документів |

---

## Перевірка ролей у коді

### grunt.get_user() — поточний користувач

```python
# Перевірити наявність хоча б однієї ролі
grunt.get_user().has_role("Manager", "Director")  # True або False

# Перевірити дію над DocType / документом (та сама перевірка, що й у REST)
await grunt.has_permission("Invoice", "export")
await grunt.has_permission("Invoice", "write", doc_id="INV-0001")

# Отримати всі ролі
roles = grunt.get_user().role_names  # ["Sales", "Manager"]

# Перевірити суперадміна (обходить всі дозволи)
grunt.get_user().has_role("System Manager")  # True або False

# Email поточного користувача
grunt.get_user().email  # "ivan@company.com"
```

> `grunt.session` у фасаді немає. `grunt.get_user()` ніколи не повертає `None`: без користувача в контексті (гостьовий запит, задача поза `grunt.context`) він кидає 401, а в `grunt.system_context(...)` повертає SYSTEM-користувача з роллю System Manager.

### Захист у контролері

```python
from grunt.document.base import Document


class Invoice(Document):
    async def before_delete(self) -> None:
        if not grunt.get_user().has_role("Accountant", "Manager"):
            grunt.throw(
                "Тільки бухгалтер або менеджер може видаляти рахунки",
                title="Недостатньо прав",
            )

    async def validate(self) -> None:
        # Тільки менеджер може встановити знижку > 20%
        if self.discount and self.discount > 20:
            if not grunt.get_user().has_role("Manager"):
                grunt.throw("Знижку понад 20% може встановити лише менеджер")
```

### Захист у хуку

```python
from grunt.hooks import on_doc


@on_doc("Settings", "before_save")
async def protect_system_settings(doc, user, **kwargs):
    if not user.is_superadmin and "Manager" not in user.roles:
        grunt.throw("Системні налаштування доступні лише адміністраторам")
```

---

## Системні ролі

| Роль | Опис |
|------|------|
| `System Manager` | Повний доступ до налаштувань системи |
| `Administrator` | Суперадмін (обходить всі дозволи) |

Кастомні ролі створюються через DocType `Role` в системному додатку.

---

## Row-level security (дозволи на рівні рядка)

Для обмеження доступу до конкретних документів (наприклад, менеджер бачить лише свої замовлення) використовуй `filters` у `get_list`:

```python
# Отримати лише замовлення поточного користувача
orders = await grunt.get_list(
    "Order",
    filters={"assigned_to": grunt.get_user().email},
)
```

Або в контролері перевіряй при читанні:

```python
async def validate(self) -> None:
    if self.assigned_to != grunt.get_user().email:
        if not grunt.get_user().has_role("Manager"):
            grunt.throw("Ви можете редагувати лише власні документи")
```

---

## Типовий патерн для бізнес-правил

```python
class Contract(Document):
    async def before_save(self) -> None:
        # Лише юрист може затверджувати контракти
        if self.status == "Затверджено" and not grunt.get_user().has_role(
            "Legal", "Manager"
        ):
            grunt.throw("Затверджувати контракти може лише юридичний відділ")

        # Автоматичне присвоєння власника при створенні
        if not self.owner:
            self.owner = grunt.get_user().email
```
