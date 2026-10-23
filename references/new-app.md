# Новий додаток на Ґрунт

Виконуй кроки **по порядку**. Не пропускай жоден.

> [!IMPORTANT]
> **Принцип "Everything is a DocType"**: Твій додаток має бути побудований навколо DocTypes. Бізнес-логіка в контролерах доктайпів, сторінки — як доктайпи, UI — налаштовується через метадані.

## Чеклист

```
- [ ] Крок 1: Підтвердити apps/ директорію
- [ ] Крок 2: Scaffold нового додатку
- [ ] Крок 3: Визначити DocType
- [ ] Крок 4: Sync DocType з БД
- [ ] Крок 5: Додати бізнес-логіку (контролери, хуки)
- [ ] Крок 6: Запустити та перевірити
```

---

## Крок 1: Підтвердити apps/ директорію

```bash
ls apps/
```

Якщо команда успішна — ти в правильному місці (корінь grunt-bench або проєкт).

Якщо `apps/` не існує — перевір, чи є ти в корені Ґрунт-проєкту:

```bash
ls apps/ sites/
```

Не запускай нічого додаткового для перевірки.

---

## Крок 2: Scaffold нового додатку

```bash
grunt app create <name>
```

Приклад:

```bash
grunt app create my_crm
```

Команда інтерактивна (запитає title, автора, email, назву модуля, git) і створить `apps/my_crm/`:

```
apps/my_crm/
├── app.json              ← маніфест додатку
├── grunt_app.py
├── install.py
├── README.md
├── .gitignore
└── my_crm/               ← Python-пакет модуля (== назва модуля, типово == name)
    ├── __init__.py
    ├── hooks.py          ← doc_events, scheduler_events, …
    ├── tasks.py          ← фонові задачі
    ├── routes.py         ← APIRouter → /api/v1/app/my_crm
    ├── doctypes/
    ├── fixtures/
    └── templates/
```

Далі:

```bash
grunt app install <name>     # реєструє додаток на сайті, вантажить DocTypes/fixtures
```

### Маніфест app.json

```json
{
  "name": "my_crm",
  "title": "My CRM",
  "version": "0.1.0",
  "description": "Customer Relationship Management",
  "author": "Your Name",
  "email": "you@example.com",
  "modules": ["my_crm"],
  "depends_on": []
}
```

---

## Крок 3: Визначити DocType

Створи теку `apps/<name>/<name>/doctypes/<DocTypeName>/` з файлом `<DocTypeName>.json`.

Мінімальний приклад:

```json
{
  "name": "Customer",
  "label": "Клієнт",
  "module": "my_crm",
  "title_field": "full_name",
  "fields": [
    {
      "fieldname": "full_name",
      "label": "Повне ім'я",
      "fieldtype": "Text",
      "required": true,
      "in_list_view": true
    },
    {
      "fieldname": "email",
      "label": "Email",
      "fieldtype": "Text",
      "in_list_view": true
    },
    {
      "fieldname": "status",
      "label": "Статус",
      "fieldtype": "Select",
      "options": "Лід\nКлієнт\nНеактивний",
      "in_list_view": true
    }
  ],
  "permissions": [
    {"role": "Sales", "read": true, "write": true, "create": true}
  ]
}
```

Докладніше про типи полів та структуру DocType → [doctypes.md](./doctypes.md)

---

## Крок 4: Sync DocType з БД

```bash
grunt doctype sync Customer
```

Це автоматично:
- Генерує Alembic міграцію
- Застосовує її до БД
- Реєструє DocType в реєстрі

**Правила:**
- Завжди передавай точну назву DocType (PascalCase)
- Після sync НЕ потрібно перевіряти схему в БД вручну — output команди є джерелом правди
- Якщо sync падає з помилкою в JSON — виправ файл, потім повтори sync

---

## Крок 5: Додати бізнес-логіку

### Контролер (поряд із JSON файлом)

```python
# apps/my_crm/my_crm/doctypes/Customer/customer.py
from grunt.document.base import Document


class Customer(Document):
    async def validate(self) -> None:
        if self.email and "@" not in self.email:
            grunt.throw("Невірна адреса email")

    async def after_insert(self) -> None:
        await grunt.notify(
            users=[grunt.get_user().email],
            subject=f"Новий клієнт: {self.full_name}",
            message=f"Клієнт {self.full_name} додано до системи.",
            doctype="Customer",
            doc_id=self.id,
        )
```

Докладніше → [controllers.md](./controllers.md)

### Хук (глобальна подія)

```python
# apps/my_crm/my_crm/hooks.py
from grunt.hooks import on_doc


@on_doc("Customer", "after_insert")
async def on_customer_insert(doc, user, **kwargs):
    await grunt.publish(
        user=user.email,
        event="msgprint",
        message=f"Ласкаво просимо, {doc['full_name']}!",
        type="success",
    )
```

Докладніше → [hooks.md](./hooks.md)

---

## Крок 6: Запустити та перевірити

```bash
grunt serve
```

Або тільки backend:

```bash
grunt serve --no-frontend
```

Перевір API:

```
GET http://localhost:8000/api/v1/docs/Customer
POST http://localhost:8000/api/v1/docs/Customer
```

Swagger UI: `http://localhost:8000/docs`
