# Новий додаток на Ґрунт

Виконуй кроки **по порядку**. Не пропускай жоден.

## Чеклист

```
- [ ] Крок 1: Підтвердити grunt_apps/ директорію
- [ ] Крок 2: Scaffold нового додатку
- [ ] Крок 3: Визначити DocType
- [ ] Крок 4: Sync DocType з БД
- [ ] Крок 5: Додати бізнес-логіку (контролери, хуки)
- [ ] Крок 6: Запустити та перевірити
```

---

## Крок 1: Підтвердити grunt_apps/ директорію

```bash
ls grunt_apps/
```

Якщо команда успішна — ти в правильному місці (корінь grunt-bench або проєкт).

Якщо `grunt_apps/` не існує — перевір, чи є ти в корені Ґрунт-проєкту:

```bash
ls grunt_apps/ sites/ apps/
```

Не запускай нічого додаткового для перевірки.

---

## Крок 2: Scaffold нового додатку

```bash
grunt create-app <name>
```

Приклад:

```bash
grunt create-app my_crm
```

Команда створить структуру в `grunt_apps/my_crm/`:

```
grunt_apps/my_crm/
├── app.json              ← маніфест додатку
├── __init__.py
├── doctypes/             ← JSON визначення DocType + .py контролери
├── tasks/
│   ├── __init__.py
│   └── tasks.py          ← фонові задачі
└── hooks/
    ├── __init__.py
    └── hooks.py          ← хуки подій
```

Перевір:

```bash
ls grunt_apps/<name>/
```

### Маніфест app.json

```json
{
  "name": "my_crm",
  "label": "My CRM",
  "version": "0.1.0",
  "description": "Customer Relationship Management",
  "author": "Your Name"
}
```

---

## Крок 3: Визначити DocType

Створи JSON файл у `grunt_apps/<name>/doctypes/<DocTypeName>.json`.

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
# grunt_apps/my_crm/doctypes/Customer.py
from grunt.core.document.controller import Document

class Customer(Document):
    async def validate(self) -> None:
        if self.email and "@" not in self.email:
            grunt.throw("Невірна адреса email")

    async def after_insert(self) -> None:
        await grunt.notify(
            users=[grunt.session.user],
            subject=f"Новий клієнт: {self.full_name}",
            message=f"Клієнт {self.full_name} додано до системи.",
            doctype="Customer",
            doc_id=self.id,
        )
```

Докладніше → [controllers.md](./controllers.md)

### Хук (глобальна подія)

```python
# grunt_apps/my_crm/hooks/hooks.py
from grunt.core.hooks import on

@on("after_insert", doctype="Customer")
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
grunt serve --backend-only
```

Перевір API:

```
GET http://localhost:8000/api/v1/docs/Customer
POST http://localhost:8000/api/v1/docs/Customer
```

Swagger UI: `http://localhost:8000/docs`
