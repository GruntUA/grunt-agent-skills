# Фонові задачі у Ґрунт

Ґрунт використовує **TaskIQ** для виконання довгих або відкладених операцій у фоні. Брокер — Redis Stream (якщо задано `REDIS_URL`), інакше in-memory брокер для локальної розробки.

## Де розміщувати

```
apps/<app>/<module>/tasks.py
```

---

## Базовий синтаксис

```python
from grunt.tasks.broker import retryable_task
import grunt


@retryable_task(max_retries=3, delay=60)
async def my_task(param1: str, param2: int = 0):
    """Опис задачі."""
    # логіка задачі
    ...
```

| Параметр `@retryable_task` | Опис |
|---------------------------|------|
| `max_retries` | Кількість повторних спроб при помилці (типово 3) |
| `delay` | Затримка між спробами, секунди (типово 60) |

---

## Запуск задачі з коду

### Асинхронний запуск (у фоні)

```python
# Поставити в чергу і одразу повернутись
await my_task.kiq(param1="value", param2=42)
```

### З контролера

```python
from grunt.document.base import Document


class Order(Document):
    async def after_insert(self) -> None:
        # Надіслати підтвердження email у фоні
        from my_crm.sales.tasks import send_order_confirmation

        await send_order_confirmation.kiq(
            order_id=self.name,
            customer_email=self.customer_email,
        )
```

### З хука

```python
from grunt.hooks import on, on_doc


@on_doc("Lead", "after_insert")
async def on_lead_created(doc, user, **kwargs):
    from my_crm.crm.tasks import sync_lead_to_external_crm

    await sync_lead_to_external_crm.kiq(lead_id=doc["name"])
```

---

## Приклади задач

### Відправка email

```python
@retryable_task(max_retries=3, delay=30)
async def send_order_confirmation(order_id: str, customer_email: str):
    """Надіслати підтвердження замовлення."""
    order = await grunt.get_doc("Order", order_id)
    await send_email(
        to=customer_email,
        subject=f"Замовлення #{order['name']} підтверджено",
        body=f"Дякуємо за замовлення на суму {order['amount']} грн.",
    )
```

### Синхронізація з зовнішньою системою

```python
@retryable_task(max_retries=5, delay=120)
async def sync_lead_to_external_crm(lead_id: str):
    """Синхронізувати лід із зовнішньою CRM."""
    lead = await grunt.get_doc("Lead", lead_id)
    response = await http_client.post(
        "https://external-crm.example.com/api/leads",
        json={"name": lead["full_name"], "email": lead["email"]},
    )
    if response.status_code == 200:
        await grunt.db.set_value("Lead", lead_id, "synced_to_crm", True)
```

### Масова операція

```python
@retryable_task(max_retries=1, delay=0)
async def recalculate_customer_stats():
    """Перерахувати статистику для всіх клієнтів."""
    customers = await grunt.get_list("Customer", filters={}, fields=["name"])
    for customer in customers:
        orders = await grunt.get_list(
            "Order",
            filters={"customer": customer["name"], "status": "Завершено"},
            fields=["amount"],
        )
        total = sum(o["amount"] for o in orders)
        await grunt.db.set_value("Customer", customer["name"], "total_orders", total)
```

---

## Розклад (scheduled tasks)

Зареєструй задачі у `hooks.py` застосунку через `scheduler_events` — Ґрунт підхоплює цей словник під час завантаження застосунку:

```python
# apps/my_crm/crm/hooks.py
scheduler_events = {
    "daily": ["my_crm.crm.tasks.recalculate_customer_stats"],
    "cron": [
        {
            "handler": "my_crm.crm.tasks.sync_lead_to_external_crm",
            "expression": "*/30 * * * *",
        },
    ],
}
```

Ключі-псевдоніми: `all` (щохвилини), `hourly`, `daily`. Для довільного розкладу — `cron` зі списком `{"handler", "expression"}`.

### Приклади cron виразів

| Вираз | Коли |
|-------|------|
| `"0 2 * * *"` | Щодня о 02:00 |
| `"*/30 * * * *"` | Кожні 30 хвилин |
| `"0 */6 * * *"` | Кожні 6 годин |
| `"0 9 * * 1"` | Щопонеділка о 09:00 |
| `"0 0 1 * *"` | Перший день кожного місяця |

---

## Запуск воркера

```bash
grunt worker           # запуск воркера TaskIQ
# еквівалент: taskiq worker grunt.tasks.broker:broker
```

Для Redis-брокера (продакшн) потрібен запущений Redis і змінна `REDIS_URL`. Без неї працює in-memory брокер — задачі виконуються в тому ж процесі, розклад і ретраї не переживають рестарт.

---

## Анти-паттерни

- ❌ Не виконуй довгих операцій (> 5 сек) прямо в контролері/хуку — виноси у фонову задачу
- ❌ Не передавай великі об'єкти в аргументи задачі — передавай лише `name` (id)
- ❌ Не забувай про `max_retries` — задачі без retry можуть загубитись при збої брокера
