# DocType у Ґрунт

DocType — це **центральна абстракція Ґрунт**. Він описує модель даних і автоматично генерує:
- Таблицю в БД
- REST API (`/api/v1/docs/<DocTypeName>`)
- UI форму та список документів
- WebSocket канал

## Де розміщувати

```
grunt_apps/<app>/doctypes/<DocTypeName>.json   ← обов'язково
grunt_apps/<app>/doctypes/<DocTypeName>.py     ← опціонально (контролер)
```

Назва файлу = назва DocType. Наприклад: `Customer.json`, `SalesOrder.json`.

**НЕ створюй папки вручну** — Ґрунт керує структурою сам.

---

## Структура JSON файлу

```json
{
  "name": "Customer",
  "label": "Клієнт",
  "module": "my_crm",
  "title_field": "full_name",
  "is_child": false,
  "fields": [...],
  "permissions": [...]
}
```

| Поле | Тип | Опис |
|------|-----|------|
| `name` | string | Унікальний ідентифікатор (PascalCase) |
| `label` | string | Людська назва (відображається в UI) |
| `module` | string | Назва додатку |
| `title_field` | string | Поле, що використовується як заголовок документа |
| `is_child` | bool | `true` для Child DocType (використовується в TABLE полях) |

---

## Типи полів (fieldtype)

### Прості типи

| fieldtype | БД | Опис |
|-----------|-----|------|
| `"Text"` | varchar(255) | Короткий текст |
| `"LongText"` | text | Довгий текст |
| `"Int"` | integer | Ціле число |
| `"Float"` | numeric(20,6) | Дробове число |
| `"Check"` | boolean | Прапорець (true/false) |
| `"Date"` | date | Дата |
| `"Datetime"` | timestamp with timezone | Дата та час |
| `"Time"` | time | Час |

### Зв'язки

| fieldtype | Опис |
|-----------|------|
| `"Link"` | FK до іншого DocType. Вкажи `"options": "DocTypeName"` |
| `"MultiLink"` | Many-to-many зв'язок |

### Медіа

| fieldtype | Опис |
|-----------|------|
| `"Attach"` | Посилання на завантажений файл |
| `"Image"` | Зображення |

### Структурні (не в БД — лише для UI)

| fieldtype | Опис |
|-----------|------|
| `"Section"` | Розділяє форму на секції |
| `"Column"` | Колонка (2 або 3 колонки поруч) |
| `"Tab"` | Вкладка в формі |
| `"Table"` | Дочірня таблиця (вкажи `"options": "ChildDocTypeName"`) |

### Спеціальні

| fieldtype | Опис |
|-----------|------|
| `"Select"` | Dropdown. Опції через `\n` в `"options"` |
| `"RichText"` | HTML редактор (Tiptap) |
| `"JSON"` | JSONB поле |
| `"Code"` | Code editor |
| `"Color"` | Color picker |
| `"Signature"` | Поле підпису |
| `"Geolocation"` | GPS координати |

---

## Повний опис поля

```json
{
  "fieldname": "status",
  "label": "Статус",
  "fieldtype": "Select",
  "options": "Чернетка\nАктивний\nЗакритий",
  "required": true,
  "in_list_view": true,
  "default": "Чернетка",
  "read_only": false,
  "hidden": false,
  "description": "Поточний стан запису"
}
```

| Атрибут | Опис |
|---------|------|
| `fieldname` | Унікальний snake_case ідентифікатор поля |
| `label` | Назва в UI |
| `required` | Обов'язкове поле |
| `in_list_view` | Показувати в списку документів |
| `default` | Значення за замовчуванням |
| `read_only` | Тільки для читання |
| `hidden` | Приховане поле |
| `description` | Підказка під полем |

---

## Link поле (зв'язок між DocType)

```json
{
  "fieldname": "customer",
  "label": "Клієнт",
  "fieldtype": "Link",
  "options": "Customer",
  "required": true,
  "in_list_view": true
}
```

---

## Child DocType та TABLE поле

Для відносин "один до багатьох" (наприклад, рядки замовлення):

### 1. Створи Child DocType

```json
{
  "name": "OrderItem",
  "label": "Рядок замовлення",
  "module": "my_crm",
  "is_child": true,
  "fields": [
    {"fieldname": "product", "label": "Товар", "fieldtype": "Text", "required": true},
    {"fieldname": "qty", "label": "Кількість", "fieldtype": "Int", "required": true},
    {"fieldname": "price", "label": "Ціна", "fieldtype": "Float"}
  ]
}
```

### 2. Додай TABLE поле у батьківський DocType

```json
{
  "fieldname": "items",
  "label": "Товари",
  "fieldtype": "Table",
  "options": "OrderItem"
}
```

---

## Permissions у DocType

```json
"permissions": [
  {"role": "Sales", "read": true, "write": true, "create": true, "delete": false},
  {"role": "Manager", "read": true, "write": true, "create": true, "delete": true}
]
```

Докладніше → [permissions.md](./permissions.md)

---

## Sync DocType з БД

**Після кожної зміни** у JSON файлі:

```bash
grunt doctype sync <DocTypeName>
```

Приклад:

```bash
grunt doctype sync Customer
grunt doctype sync OrderItem
```

**Правила:**
- Sync запускай окремо для кожного DocType
- Якщо sync падає — виправи JSON, потім повтори
- Після sync **не перевіряй БД вручну** — output команди є джерелом правди
- **Не видаляй існуючі поля** без розуміння наслідків — це видалить дані в БД

---

## Анти-паттерни

- ❌ Не пиши міграції вручну (Alembic `.py` файли) — вони генеруються автоматично
- ❌ Не видаляй поля без backup даних
- ❌ Не використовуй `fieldname` з пробілами або CamelCase — тільки `snake_case`
- ❌ Не дублюй `fieldname` в межах одного DocType
