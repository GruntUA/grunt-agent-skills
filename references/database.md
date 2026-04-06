# База даних у Ґрунт

Ґрунт надає два рівні роботи з БД:

1. **`grunt.db.*`** — зручні helper функції для типових операцій
2. **`grunt.get_list()` / `grunt.get_doc()`** — повний Document API з hooks

> ⚠️ **Ніколи не пиши raw SQL.** Завжди використовуй `grunt.db.*` або SQLAlchemy Core/ORM.

---

## grunt.db helpers

### grunt.db.get_value(doctype, filters, fieldname)

Отримати значення одного поля з першого знайденого документа.

```python
# Отримати ім'я клієнта за email
name = await grunt.db.get_value("Customer", {"email": "ivan@example.com"}, "full_name")

# Отримати статус замовлення за ID
status = await grunt.db.get_value("Order", {"id": order_id}, "status")
```

Повертає `None` якщо запис не знайдено.

### grunt.db.set_value(doctype, doc_id, fieldname, value)

Оновити одне поле документа **напряму в БД** (без lifecycle hooks).

```python
await grunt.db.set_value("Invoice", invoice_id, "status", "Оплачено")
```

> ⚠️ Використовуй лише для полів **без бізнес-логіки в контролері**. Якщо поле має `validate`, `before_save` тощо — використовуй `grunt.save_doc()` замість цього.

### grunt.db.exists(doctype, filters)

Перевірити, чи існує документ. Повертає `name` або `None`.

```python
if await grunt.db.exists("Customer", {"email": "ivan@example.com"}):
    grunt.throw("Клієнт з таким email вже існує")
```

```python
# Перевірка перед операцією
customer_id = await grunt.db.exists("Customer", {"phone": "+380501234567"})
if customer_id:
    # оновити існуючого
    await grunt.save_doc("Customer", customer_id, {"status": "Активний"})
else:
    # створити нового
    await grunt.new_doc("Customer", {...})
```

### grunt.db.get_all(doctype, *, filters, fields, limit, order_by, order)

Отримати список документів як список словників (нижчий рівень, ніж `grunt.get_list`).

```python
customers = await grunt.db.get_all(
    "Customer",
    filters={"status": "Активний"},
    fields=["id", "full_name", "email"],
    limit=100,
    order_by="created_at",
    order="desc",
)
```

---

## grunt.get_list() — повний Document API

Рекомендований спосіб для більшості запитів. Підтримує пагінацію, пошук, складні фільтри.

```python
result = await grunt.get_list(
    "Order",
    filters={
        "status": "Відкрите",
        "amount__gte": "500",
        "created_at__gte": "2024-01-01",
    },
    fields=["id", "name", "customer", "amount", "status"],
    limit=20,
    page=1,
    order_by="created_at",
    order="desc",
    search="Іван",  # full-text пошук
)
```

---

## Фільтри

### Прості рівні

```python
filters={"status": "Active"}                    # = 'Active'
filters={"status": "Active", "type": "VIP"}    # AND умова
```

### Порівняння

```python
filters={"amount__gte": "1000"}     # amount >= 1000
filters={"amount__lte": "5000"}     # amount <= 5000
filters={"amount__gt": "100"}       # amount > 100
filters={"amount__lt": "9999"}      # amount < 9999
```

### Дати

```python
filters={"created_at__gte": "2024-01-01"}   # від початку 2024
filters={"due_date__lt": "2024-12-31"}      # до кінця 2024
```

### LIKE пошук

```python
filters={"full_name__like": "%Іванен%"}     # містить "Іванен"
```

---

## Транзакції

Ґрунт автоматично управляє транзакціями на рівні запиту. **Не викликай `commit()` вручну** в контролерах або хуках — це порушить транзакційну цілісність.

```python
# ✅ Правильно — Ґрунт сам робить commit
async def after_insert(self) -> None:
    await grunt.new_doc("Log", {"message": "Created"})

# ❌ Неправильно — ніколи не роби так
async def after_insert(self) -> None:
    await session.commit()  # ЗАБОРОНЕНО
```

---

## Приклади реальних сценаріїв

### Перевірити унікальність перед збереженням

```python
async def validate(self) -> None:
    if await grunt.db.exists("Customer", {"email": self.email}):
        grunt.throw(f"Клієнт з email '{self.email}' вже існує")
```

### Отримати пов'язані записи

```python
async def after_insert(self) -> None:
    # Знайти всі відкриті замовлення цього клієнта
    orders = await grunt.get_list(
        "Order",
        filters={"customer": self.name, "status": "Відкрите"},
        fields=["id", "amount"],
    )
    total_open = sum(o["amount"] for o in orders)
```

### Масове оновлення

```python
# Закрити всі прострочені замовлення
count = await grunt.bulk_update(
    "Order",
    filters={"status": "Відкрите", "due_date__lt": "2024-01-01"},
    values={"status": "Прострочено"},
)
print(f"Закрито {count} замовлень")
```

---

## Анти-паттерни

- ❌ **Не пиши raw SQL** — `session.execute("SELECT ...")` заборонено
- ❌ **Не роби N+1 запитів** — завантажуй пов'язані дані одним `get_list` з потрібними `fields`
- ❌ **Не використовуй `set_value` для полів з валідацією** — використовуй `save_doc`
- ❌ **Не викликай `commit()` вручну** — Ґрунт керує транзакціями
