# Хуки у Ґрунт

Хуки дозволяють реагувати на події DocType **глобально** — на відміну від контролерів, які прив'язані до конкретного документа. Хуки зручні для cross-cutting logic: логування, сповіщень, синхронізації з зовнішніми системами.

## Де розміщувати

```
grunt_apps/<app>/hooks/hooks.py
```

---

## Базовий синтаксис

```python
from grunt.core.hooks import on
from grunt.app import grunt

@on("<event>", doctype="DocTypeName")
async def handler_name(doc, user, **kwargs):
    ...
```

| Параметр | Тип | Опис |
|----------|-----|------|
| `doc` | dict | Дані документа (поля як ключі словника) |
| `user` | User | Поточний користувач (об'єкт з `.email`, `.roles`) |
| `**kwargs` | dict | Додаткові параметри події |

---

## Доступні події

| Подія | Коли викликається |
|-------|------------------|
| `"validate"` | Перед збереженням (insert або update) |
| `"before_insert"` | До запису нового документа в БД |
| `"after_insert"` | Після успішного створення |
| `"before_save"` | До оновлення існуючого документа |
| `"after_save"` | Після успішного оновлення |
| `"before_delete"` | До видалення |
| `"after_delete"` | Після видалення |

---

## Приклади

### Сповіщення після створення

```python
from grunt.core.hooks import on
from grunt.app import grunt

@on("after_insert", doctype="Customer")
async def on_customer_created(doc, user, **kwargs):
    await grunt.publish(
        user=user.email,
        event="msgprint",
        message=f"Ласкаво просимо, {doc['full_name']}!",
        type="success",
    )
```

### Валідація через хук (для всіх DocType одного додатку)

```python
@on("validate", doctype="Order")
async def validate_order_total(doc, user, **kwargs):
    if doc.get("total", 0) < 0:
        grunt.throw("Сума замовлення не може бути від'ємною")
```

### Логування змін

```python
@on("after_save", doctype="Invoice")
async def log_invoice_change(doc, user, **kwargs):
    await grunt.new_doc("AuditLog", {
        "doctype_ref": "Invoice",
        "doc_id": doc["id"],
        "action": "updated",
        "user": user.email,
    })
```

### Автоматична синхронізація з зовнішньою системою

```python
@on("after_insert", doctype="Lead")
async def sync_lead_to_crm(doc, user, **kwargs):
    # Запуск фонової задачі для синхронізації
    from my_crm.tasks.tasks import push_lead_to_external_crm
    await push_lead_to_external_crm.delay(lead_id=doc["id"])
```

### Реакція на видалення

```python
@on("after_delete", doctype="Customer")
async def notify_on_customer_delete(doc, user, **kwargs):
    await grunt.notify(
        users=["admin@company.com"],
        subject=f"Клієнта видалено",
        message=f"Клієнт {doc.get('full_name')} видалений користувачем {user.email}.",
    )
```

---

## Кілька хуків для одного DocType

Можна зареєструвати кілька функцій на одну й ту саму подію:

```python
@on("after_insert", doctype="Order")
async def send_order_notification(doc, user, **kwargs):
    ...

@on("after_insert", doctype="Order")
async def create_order_log(doc, user, **kwargs):
    ...
```

Обидва хуки виконаються — порядок не гарантований.

---

## Різниця між контролером і хуком

| | Контролер (`DocType.py`) | Хук (`hooks.py`) |
|---|---|---|
| Де | Поряд із JSON файлом DocType | В `hooks/hooks.py` додатку |
| Прив'язка | До одного DocType | До будь-якого DocType |
| Доступ до `self` | Так (об'єкт Document) | Ні (лише `doc` як dict) |
| Використовуй коли | Основна бізнес-логіка цього DocType | Cross-cutting логіка між DocTypes |
