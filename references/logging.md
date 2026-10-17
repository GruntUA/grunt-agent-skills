# Логування у Ґрунт

Ґрунт використовує **structlog** для структурованого логування. Всі логи — у форматі key-value пар, що дозволяє їх фільтрувати та агрегувати.

> ⚠️ Ніколи не використовуй `print()` або `logging.basicConfig()` — тільки `structlog`.

---

## Базовий синтаксис

```python
import structlog

logger = structlog.get_logger()
```

Оголошуй `logger` на рівні модуля — один раз, не всередині функцій.

### Рівні логування

```python
logger.debug("event.name", key="value", count=42)
logger.info("event.name", key="value", count=42)
logger.warning("event.name", key="value", reason="timeout")
logger.error("event.name", key="value", exc_info=True)
```

| Рівень | Коли використовувати |
|--------|----------------------|
| `debug` | Детальна діагностика (тільки в dev) |
| `info` | Нормальний перебіг подій |
| `warning` | Несподівана, але не критична ситуація |
| `error` | Помилка, яку треба розслідувати |

---

## Стиль іменування подій

Назва події — `"namespace.action"` у snake_case:

```python
# ✅ Правильно
logger.info("customer.created", customer_id=str(self.id), email=self.email)
logger.warning("order.limit_exceeded", order_id=str(self.id), amount=self.amount)
logger.error("sync.failed", doctype="Lead", lead_id=str(lead_id), exc_info=True)

# ❌ Неправильно
logger.info("Customer created!")        # рядок без структури
logger.info(f"Order {id} failed")       # f-string замість key-value
print("debug:", value)                  # print заборонено
```

---

## Логування в контролері

```python
import structlog
from grunt.core.document.controller import Document

logger = structlog.get_logger()

class Order(Document):
    async def validate(self) -> None:
        if self.total < 0:
            grunt.throw("Сума не може бути від'ємною")

    async def after_insert(self) -> None:
        logger.info(
            "order.created",
            order_id=str(self.id),
            customer=self.customer,
            total=float(self.total),
            user=grunt.session.user,
        )

    async def before_delete(self) -> None:
        logger.warning(
            "order.delete_requested",
            order_id=str(self.id),
            user=grunt.session.user,
        )
```

---

## Логування в хуку

```python
import structlog
from grunt.core.hooks import on

logger = structlog.get_logger()

@on("after_insert", doctype="Customer")
async def on_customer_insert(doc, user, **kwargs):
    logger.info(
        "customer.hook_triggered",
        event="after_insert",
        customer_id=doc["id"],
        user=user.email,
    )
```

---

## Логування у фоновій задачі

```python
import structlog
from grunt.core.tasks.broker import retryable_task

logger = structlog.get_logger()

@retryable_task(max_retries=3, delay=60)
async def sync_customers():
    logger.info("sync.started", task="sync_customers")
    try:
        customers = await grunt.get_list("Customer", filters={})
        logger.info("sync.fetched", count=len(customers))
        # ... логіка
        logger.info("sync.completed", synced=len(customers))
    except Exception as e:
        logger.error("sync.failed", error=str(e), exc_info=True)
        raise
```

> Фонові задачі автоматично логуються в DocType **BackgroundTaskLog** через middleware — старт, завершення, помилки та retry.

---

## Логування в API endpoint

```python
import structlog
from fastapi import APIRouter, Depends
from grunt.core.auth.dependencies import current_user

logger = structlog.get_logger()
router = APIRouter()

@router.post("/custom-action")
async def custom_action(user=Depends(current_user)):
    logger.info("api.custom_action", user=user.email)
    # ...
```

---

## exc_info=True для помилок

Передавай `exc_info=True` всередині `except` блоку — structlog автоматично додасть traceback:

```python
try:
    result = await some_external_api()
except Exception as e:
    logger.error(
        "external_api.request_failed",
        service="payment_gateway",
        error=str(e),
        exc_info=True,   # ← додає повний traceback
    )
    raise
```

---

## Вбудовані DocType для логів

Ґрунт зберігає системні події у спеціальних DocType:

| DocType | Що зберігає |
|---------|------------|
| `BackgroundTaskLog` | Виконання фонових задач: статус, час, помилки, retry |
| `ActivityLog` | Дії з документами: Create, Update, Delete, Submit, Share, Comment |

### Запит до BackgroundTaskLog

```python
failed_tasks = await grunt.get_list(
    "BackgroundTaskLog",
    filters={"status": "Error"},
    fields=["task_name", "error_message", "started_at", "finished_at"],
    order_by="started_at",
    order="desc",
    limit=20,
)
```

### Запис до ActivityLog вручну

ActivityLog заповнюється автоматично системою. Але якщо потрібно додати власну подію:

```python
await grunt.new_doc("ActivityLog", {
    "doctype": "Customer",
    "doc_id": str(self.id),
    "user": grunt.session.user,
    "action": "Update",
    "details": {"changed_field": "status", "new_value": self.status},
})
```

---

## Коли що використовувати

| Ситуація | Використовуй |
|----------|-------------|
| Трасування виконання, debug | `logger.info(...)` / `logger.debug(...)` |
| Несподівана, але некритична ситуація | `logger.warning(...)` |
| Помилка в `except` блоці | `logger.error(..., exc_info=True)` |
| Відстеження дій з документами | `ActivityLog` (автоматично або вручну) |
| Моніторинг фонових задач | `BackgroundTaskLog` (заповнюється автоматично) |

---

## Анти-паттерни

- ❌ **Не використовуй `print()`** — логи не потрапляють у систему збору
- ❌ **Не використовуй `logging.basicConfig()`** — конфлікт зі structlog конфігурацією
- ❌ **Не логуй f-strings як повідомлення** — `logger.info(f"Created {id}")` замість key-value
- ❌ **Не оголошуй `logger` всередині функцій** — оголошуй на рівні модуля один раз
- ❌ **Не логуй паролі, токени, секрети** — перевіряй, що key-value не містить чутливих даних
- ❌ **Не ковтай виключення мовчки** — після `logger.error(...)` роби `raise` або `grunt.throw()`
