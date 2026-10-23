# Хуки у Ґрунт

Хуки дозволяють реагувати на події DocType **глобально** — на відміну від контролерів, які прив'язані до конкретного документа. Хуки зручні для cross-cutting logic: логування, сповіщень, синхронізації з зовнішніми системами.

## Де розміщувати

```
apps/<app>/<module>/hooks.py
```

Ґрунт знаходить кожен `*/hooks.py` встановленого застосунку і читає з нього відомі імена (`doc_events`, `scheduler_events`, `io_exporters`, `doc_actions`, …) через реєстр консюмерів (`grunt.apps.consumers.HOOK_CONSUMERS`). Декоратори `@on` / `@on_doc` спрацьовують у момент імпорту цього модуля.

---

## Два способи реєстрації

### 1. Декоратори `@on` / `@on_doc`

```python
from grunt.hooks import on, on_doc


# Глобальний хук — на подію будь-якого DocType
@on("after_save", priority=10)
async def audit_every_save(doctype, doc, user, **kwargs): ...


# Хук для конкретного DocType
@on_doc("SalesOrder", "after_submit")
async def on_so_submit(doc, user, **kwargs): ...
```

| | `@on(event, priority=10)` | `@on_doc(doctype, event, priority=10)` |
|---|---|---|
| Область | усі DocType | один DocType |
| Сигнатура хендлера | `(doctype, doc, user, **kwargs)` | `(doc, user, **kwargs)` |

Менший `priority` виконується раніше (типово `10`).

### 2. Декларативний словник `doc_events`

Той самий формат, яким користується саме ядро (`grunt/core_hooks.py`):

```python
doc_events = {
    "SalesOrder": {
        "after_save": [
            "my_app.sales.handlers.recalc_totals",
            {"handler": "my_app.sales.handlers.notify_manager", "priority": 1},
        ],
    },
    # "*" — хук на всі DocType
    "*": {
        "after_delete": ["my_app.audit.log_delete"],
    },
}
```

Рядок — це шлях `module.function`; форма `{"handler": ..., "priority": ...}` задає порядок.

---

## Параметри хендлера

| Параметр | Тип | Опис |
|----------|-----|------|
| `doc` | dict | Дані документа (поля як ключі; первинний ключ — `doc["name"]`) |
| `user` | User | Поточний користувач (об'єкт з `.email`, `.roles`) |
| `doctype` | str | Ім'я DocType (передається завжди; у сигнатурі `@on` — позиційно) |
| `event` | str | Ім'я події |
| `session` | AsyncSession | Активна сесія БД поточної транзакції |

Хендлер може бути `async def` або звичайним `def`. Виняток у хуку логується і **не** зупиняє інші хуки чи саму операцію запису.

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
| `"after_read"` | Після читання документа |
| `"on_transition"` | Після переходу стану у workflow |

---

## Приклади

### Сповіщення після створення

```python
from grunt.hooks import on_doc
import grunt


@on_doc("Customer", "after_insert")
async def on_customer_created(doc, user, **kwargs):
    await grunt.publish(
        user=user.email,
        event="msgprint",
        message=f"Ласкаво просимо, {doc['full_name']}!",
        type="success",
    )
```

### Валідація через хук

```python
from grunt.hooks import on_doc
import grunt


@on_doc("Order", "validate")
async def validate_order_total(doc, user, **kwargs):
    if doc.get("total", 0) < 0:
        grunt.throw("Сума замовлення не може бути від'ємною")
```

### Логування змін

```python
from grunt.hooks import on_doc
import grunt


@on_doc("Invoice", "after_save")
async def log_invoice_change(doc, user, **kwargs):
    await grunt.new_doc(
        "AuditLog",
        {
            "doctype_ref": "Invoice",
            "doc_id": doc["name"],
            "action": "updated",
            "user": user.email,
        },
    )
```

### Автоматична синхронізація з зовнішньою системою

```python
from grunt.hooks import on_doc


@on_doc("Lead", "after_insert")
async def sync_lead_to_crm(doc, user, **kwargs):
    from my_crm.tasks.tasks import push_lead_to_external_crm

    await push_lead_to_external_crm.kiq(lead_id=doc["name"])
```

### Глобальний хук на видалення будь-чого

```python
from grunt.hooks import on


@on("after_delete")
async def notify_on_delete(doctype, doc, user, **kwargs):
    await grunt.notify(
        users=["admin@company.com"],
        subject="Документ видалено",
        message=f"{doctype} {doc.get('name')} видалив {user.email}.",
    )
```

---

## Кілька хуків для однієї події

Можна зареєструвати кілька функцій на одну подію одного DocType:

```python
@on_doc("Order", "after_insert")
async def send_order_notification(doc, user, **kwargs): ...


@on_doc("Order", "after_insert")
async def create_order_log(doc, user, **kwargs): ...
```

Виконаються обидва, у порядку зростання `priority` (за рівних `priority` — у порядку реєстрації).

---

## Різниця між контролером і хуком

| | Контролер (`DocType.py`) | Хук (`hooks.py`) |
|---|---|---|
| Де | Поряд із JSON файлом DocType | В `apps/<app>/<module>/hooks.py` |
| Прив'язка | До одного DocType | До одного або всіх DocType |
| Доступ до `self` | Так (об'єкт Document) | Ні (лише `doc` як dict) |
| Використовуй коли | Основна бізнес-логіка цього DocType | Cross-cutting логіка між DocTypes |

> Шина, яка розсилає ці події (`fire()` → хуки → server scripts → backlinks → notification/assignment rules), живе в `grunt.events`. Автори застосунків її не викликають — лише реєструють хуки.
