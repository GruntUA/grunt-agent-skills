# REST API та GruntApp SDK

## GruntApp SDK

Основний API для розробників додатків. Доступний у контролерах, хуках та серверних скриптах.

```python
from grunt.app import grunt
```

---

## Document CRUD

### grunt.get_doc(doctype, id_or_name)

Отримати один документ за ID або name.

```python
invoice = await grunt.get_doc("Invoice", "INV-0001")
print(invoice["amount"])
print(invoice["customer"])
```

### grunt.new_doc(doctype, data)

Створити новий документ (запускає всі lifecycle hooks).

```python
order = await grunt.new_doc("Order", {
    "customer": "CUST-001",
    "status": "Чернетка",
    "amount": 1500.0,
})
print(order["id"])
```

### grunt.save_doc(doctype, id_or_name, data)

Оновити існуючий документ. Часткове оновлення — поля, яких немає в `data`, не змінюються.

```python
await grunt.save_doc("Order", order["id"], {"status": "Підтверджено"})
```

### grunt.delete_doc(doctype, id_or_name)

Видалити документ (запускає `before_delete` / `after_delete` hooks).

```python
await grunt.delete_doc("TempLog", log_id)
```

### grunt.get_list(doctype, *, filters, fields, limit, page, order_by, order, search)

Отримати список документів.

```python
open_orders = await grunt.get_list(
    "Order",
    filters={"status": "Відкрите", "amount__gte": "1000"},
    fields=["id", "name", "customer", "amount", "status"],
    limit=50,
    order_by="created_at",
    order="desc",
)
```

**Оператори фільтрів:**

| Оператор | Приклад | Значення |
|----------|---------|---------|
| (немає) | `{"status": "Active"}` | Рівно |
| `__gte` | `{"amount__gte": "100"}` | >= |
| `__lte` | `{"amount__lte": "1000"}` | <= |
| `__gt` | `{"date__gt": "2024-01-01"}` | > |
| `__lt` | `{"date__lt": "2024-12-31"}` | < |
| `__like` | `{"name__like": "%Іван%"}` | LIKE |

### grunt.count(doctype, *, filters)

Порахувати кількість документів.

```python
total = await grunt.count("Order", filters={"status": "Відкрите"})
```

### grunt.bulk_insert(doctype, records)

Масове створення без hooks (швидко, для імпорту даних).

```python
ids = await grunt.bulk_insert("LogEntry", [
    {"level": "info", "message": "Запуск"},
    {"level": "info", "message": "Завершення"},
])
```

### grunt.bulk_update(doctype, filters, values)

Масове оновлення одним запитом до БД (без hooks).

```python
count = await grunt.bulk_update(
    "Order",
    filters={"status": "Чернетка"},
    values={"status": "Скасовано"},
)
```

---

## Сповіщення та real-time

### grunt.notify(users, subject, message, doctype, doc_id, push)

Створити постійне дзвіночок-сповіщення (зберігається в БД).

```python
await grunt.notify(
    users=["manager@company.com", "director@company.com"],
    subject="Нове замовлення потребує підтвердження",
    message=f"Замовлення #{order['name']} від {order['customer']} чекає на схвалення.",
    doctype="Order",
    doc_id=order["id"],
)
```

### grunt.publish(user, event, data, message, type)

Відправити миттєве WebSocket повідомлення одному користувачу.

```python
await grunt.publish(
    user=grunt.session.user,
    event="msgprint",
    message="Документ успішно збережено",
    type="success",   # "success" | "error" | "warning" | "info"
)
```

### grunt.broadcast(event, data, message, type)

Відправити повідомлення **всім** підключеним користувачам.

```python
await grunt.broadcast(
    event="system_alert",
    message="Заплановане технічне обслуговування о 22:00",
    type="warning",
)
```

---

## Сесія користувача

```python
grunt.session.user          # email поточного користувача
grunt.session.full_name     # повне ім'я
grunt.session.roles         # список ролей ["Manager", "Sales"]
grunt.session.is_superadmin # чи є суперадміном

# Перевірка ролей (будь-яка з перелічених)
grunt.session.has_role("Manager", "Director")  # True або False
```

---

## Помилки

### grunt.throw(message, title=None)

Генерує HTTP 422 з user-facing повідомленням.

```python
if self.amount <= 0:
    grunt.throw("Сума має бути більше нуля", title="Помилка валідації")
```

---

## Метадані

### grunt.get_meta(doctype)

Повертає Pydantic модель DocType з усіма полями.

```python
meta = await grunt.get_meta("Invoice")
field_names = [f.fieldname for f in meta.fields]
link_fields = [f for f in meta.fields if f.fieldtype == "Link"]
```

---

## Переклад

### grunt._(source)

Перекласти рядок відповідно до локалі поточного запиту.

```python
msg = grunt._("Document saved")
```

---

## REST API endpoints

### DocType (метадані)

```
GET    /api/v1/meta/doctypes              → список всіх DocType
POST   /api/v1/meta/doctypes              → створити DocType
GET    /api/v1/meta/doctypes/{name}       → отримати DocType
PUT    /api/v1/meta/doctypes/{name}       → оновити DocType
DELETE /api/v1/meta/doctypes/{name}       → видалити DocType
POST   /api/v1/meta/doctypes/{name}/sync  → синхронізувати з БД
```

### Document (дані)

```
GET    /api/v1/docs/{doctype}             → список документів
POST   /api/v1/docs/{doctype}             → створити документ
GET    /api/v1/docs/{doctype}/{id}        → отримати документ
PUT    /api/v1/docs/{doctype}/{id}        → оновити документ
DELETE /api/v1/docs/{doctype}/{id}        → видалити документ
POST   /api/v1/docs/{doctype}/{id}/submit → зафіксувати (workflow)
```

### Query параметри для GET list

```
?page=1&per_page=20
?sort=name&order=asc
?filter[status]=active
?filter[amount__gte]=1000
?fields=id,name,status,customer
?search=ключове+слово
```

### Приклад запиту

```bash
curl -H "Authorization: Bearer <token>" \
  "http://localhost:8000/api/v1/docs/Order?filter[status]=Відкрите&per_page=10"
```

---

## WebSocket

```
WS /api/v1/ws/{doctype}/{id}
```

**Отримувані події:**

```json
{"event": "doc_change", "data": {...}}
{"event": "workflow_transition", "data": {"from": "Draft", "to": "Active"}}
{"event": "notification", "data": {...}}
```

**Відправка дій:**

```json
{"action": "subscribe", "doctype": "Order"}
{"action": "unsubscribe"}
```
