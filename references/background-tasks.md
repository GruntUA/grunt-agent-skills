# Фонові задачі у Ґрунт

Ґрунт використовує **Celery + Redis** для виконання довгих або відкладених операцій у фоні.

## Де розміщувати

```
grunt_apps/<app>/tasks/tasks.py
```

---

## Базовий синтаксис

```python
from grunt.core.tasks.broker import retryable_task
from grunt.app import grunt

@retryable_task(max_retries=3, delay=60)
async def my_task(param1: str, param2: int = 0):
    """Опис задачі."""
    # логіка задачі
    pass
```

| Параметр `@retryable_task` | Опис |
|---------------------------|------|
| `max_retries` | Кількість повторних спроб при помилці |
| `delay` | Затримка між спробами (секунди) |

---

## Запуск задачі з коду

### Асинхронний запуск (в фоні)

```python
# Запустити задачу і одразу повернутись
await my_task.delay(param1="value", param2=42)
```

### З контролера

```python
from grunt.core.document.controller import Document

class Order(Document):
    async def after_insert(self) -> None:
        # Надіслати підтвердження email у фоні
        from my_crm.tasks.tasks import send_order_confirmation
        await send_order_confirmation.delay(
            order_id=str(self.id),
            customer_email=self.customer_email,
        )
```

### З хука

```python
from grunt.core.hooks import on

@on("after_insert", doctype="Lead")
async def on_lead_created(doc, user, **kwargs):
    from my_crm.tasks.tasks import sync_lead_to_external_crm
    await sync_lead_to_external_crm.delay(lead_id=str(doc["id"]))
```

---

## Приклади задач

### Відправка email

```python
@retryable_task(max_retries=3, delay=30)
async def send_order_confirmation(order_id: str, customer_email: str):
    """Надіслати підтвердження замовлення."""
    order = await grunt.get_doc("Order", order_id)
    # відправка через email-сервіс
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
    # API запит до зовнішньої системи
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
    customers = await grunt.get_list("Customer", filters={}, fields=["id"])
    for customer in customers:
        orders = await grunt.get_list(
            "Order",
            filters={"customer": customer["id"], "status": "Завершено"},
            fields=["amount"],
        )
        total = sum(o["amount"] for o in orders)
        await grunt.db.set_value("Customer", customer["id"], "total_orders", total)
```

---

## Розклад (scheduled tasks)

Зареєструй задачу в `app.json` з cron виразом:

```json
{
  "name": "my_crm",
  "scheduled_tasks": [
    {
      "task": "my_crm.tasks.tasks.recalculate_customer_stats",
      "cron": "0 2 * * *"
    },
    {
      "task": "my_crm.tasks.tasks.sync_lead_to_external_crm",
      "cron": "*/30 * * * *"
    }
  ]
}
```

### Приклади cron виразів

| Вираз | Коли |
|-------|------|
| `"0 2 * * *"` | Щодня о 02:00 |
| `"*/30 * * * *"` | Кожні 30 хвилин |
| `"0 */6 * * *"` | Кожні 6 годин |
| `"0 9 * * 1"` | Щопонеділка о 09:00 |
| `"0 0 1 * *"` | Перший день кожного місяця |

---

## Вимоги для запуску

Фонові задачі потребують запущеного Redis:

```bash
grunt serve  # запускає і Celery worker автоматично
```

Або вручну:

```bash
grunt serve --backend-only  # FastAPI
# в іншому терміналі:
celery -A grunt.core.tasks.broker worker --loglevel=info
```

---

## Анти-паттерни

- ❌ Не виконуй довгих операцій (> 5 сек) прямо в контролері/хуку — виноси у фонову задачу
- ❌ Не передавай великі об'єкти в аргументи задачі — передавай лише `id`
- ❌ Не забувай про `max_retries` — задачі без retry можуть загубитись при збої Redis
