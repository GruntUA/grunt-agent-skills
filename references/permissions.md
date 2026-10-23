# Дозволи у Ґрунт

Ґрунт використовує **Role-Based Access Control (RBAC)** — дозволи прив'язані до ролей, а ролі — до користувачів.

## Permissions у DocType JSON

Визначаються в масиві `permissions` у JSON файлі DocType:

```json
"permissions": [
  {
    "role": "Sales",
    "read": true,
    "write": true,
    "create": true,
    "delete": false
  },
  {
    "role": "Manager",
    "read": true,
    "write": true,
    "create": true,
    "delete": true
  },
  {
    "role": "Viewer",
    "read": true,
    "write": false,
    "create": false,
    "delete": false
  }
]
```

| Поле | Опис |
|------|------|
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
