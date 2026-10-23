# REST API та GruntApp SDK

## GruntApp SDK

Основний API для розробників додатків. Доступний у контролерах, хуках та серверних скриптах.

```python
import grunt
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
order = await grunt.new_doc(
    "Order",
    {
        "customer": "CUST-001",
        "status": "Чернетка",
        "amount": 1500.0,
    },
)
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
ids = await grunt.bulk_insert(
    "LogEntry",
    [
        {"level": "info", "message": "Запуск"},
        {"level": "info", "message": "Завершення"},
    ],
)
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
    user=grunt.get_user().email,
    event="msgprint",
    message="Документ успішно збережено",
    type="success",  # "success" | "error" | "warning" | "info"
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
grunt.get_user().email  # email поточного користувача
grunt.get_user().full_name  # повне ім'я
grunt.get_user().role_names  # список ролей ["Manager", "Sales"]
grunt.get_user().has_role("System Manager")  # чи є суперадміном

# Перевірка ролей (будь-яка з перелічених)
grunt.get_user().has_role("Manager", "Director")  # True або False
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

## REST API endpoints — два санкціоновані транспорти

**1. REST `/api/v1/docs/{doctype}(/{id})`** — ЛИШЕ 5 базових CRUD-операцій, більше нічого:

```
GET    /api/v1/docs/{doctype}             → список документів
POST   /api/v1/docs/{doctype}             → створити документ
GET    /api/v1/docs/{doctype}/{id}        → отримати документ
PUT    /api/v1/docs/{doctype}/{id}        → оновити документ
DELETE /api/v1/docs/{doctype}/{id}        → видалити документ (204, без тіла)
```

Працює "з коробки" для будь-якого зареєстрованого DocType — не пиши власний CRUD-роутер.

**2. RPC `/api/v1/method/{dotted.path}`** — усе інше: будь-яка дія, що не є одним із 5 verbs вище (коментарі, bookmark, версії, дерево, workflow-переходи, link-search, bulk-операції, експорт/друк, крос-доктайп пошук, звіти тощо). `dotted.path` — це Python import-path до функції, або `module.Class.method` для staticmethod/classmethod (диспетчер підтримує обидва). Диспетчер підтримує лише GET/POST — колишні PATCH/DELETE-дії стають POST-викликами RPC.

**Ніколи не вигадуй дієслово-екшен у шляху `/docs/{doctype}/щось`** (`bulk-delete`, `tree/move/{id}` тощо) — це завжди RPC whitelisted-метод, не REST-роут.

**Дженерик (крос-доктайп) операції над документом** — не прив'язані до одного конкретного DocType — живуть як `@staticmethod` на базовому контролері `Document` (`grunt/document/mixins/*_rpc.py`, підмішуються в `grunt.document.base.Document`), а не в `grunt/api/`: `grunt.document.base.Document.get_tree`, `.link_search`, `.print` тощо. Операція, специфічна для ОДНОГО DocType (напр. запуск збереженого `Report`) — у контролері ЦЬОГО доктайпу: `grunt.reports.doctypes.Report.report.run`, за тим самим патерном що й `grunt.activity.doctypes.ActivityLog.activity_log.list_activity`.

**Не дублюй контекст класу/модуля в імені методу**, але не йди в іншу крайність — на пласкому класі `Document` голе `get`/`move`/`search` занадто загальне (`get` ще й конфліктує з `Document.get(fieldname)`): `Document.get_tree`/`Document.move_tree_node`, `Document.link_search`, `Document.apply_workflow_transition`. Прибирай слово лише коли нічого не втрачається: `Document.get_versions`/`get_activity_log`/`get_timeline`, `Document.get_comments`/`add_comment`, `Document.get_backlinks`.

### Query параметри для GET list (REST)

```
?page=1&per_page=20
?sort_by=name&sort_order=asc
?filter[status]=active
?filter[amount__gte]=1000
?fields=id,name,status,customer
?search=ключове+слово
```

Для RPC-методів еквівалент фільтрів — один kwarg `filters: dict[str, Any] | None` (JSON-об'єкт, а не bracket-style query) — диспетчер сам JSON-парсить значення, що виглядають як об'єкт/масив.

### Конверт відповіді

Завжди через `ok()`/`ok_list()`/`error_body()` (`grunt/api/v1/schemas/response.py`) — `{"success": true, "data": ...}` / `{"success": false, "error": {...}}`. Диспетчер `/api/v1/method/...` сам обгортає результат whitelisted-функції — сама функція повертає сирі дані, без `ok(...)`. Виняток — бінарні/стрімінгові відповіді (export/print): повертаються сирим `Response`/`StreamingResponse`, диспетчер форвардить їх без обгортки.

### Приклад запиту

```bash
curl -H "Authorization: Bearer <token>" \
  "http://localhost:8000/api/v1/docs/Order?filter[status]=Відкрите&per_page=10"

curl -H "Authorization: Bearer <token>" \
  "http://localhost:8000/api/v1/method/grunt.document.base.Document.get_tree?doctype=Category"
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
