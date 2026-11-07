# DocType у Ґрунт

DocType — це **центральна абстракція Ґрунт**. Він описує модель даних і автоматично генерує:
- Таблицю в БД
- REST API (`/api/v1/docs/<DocTypeName>`)
- UI форму та список документів
- WebSocket канал

> [!IMPORTANT]
> **Принцип "Everything is a DocType"**: Будь-яка нова функція, сторінка чи елемент бізнес-логики повинні починатися з DocType. Якщо DocType не може реалізувати задачу — розширюй можливості ядра DocType.

## Де розміщувати

```
apps/<app>/<module>/doctypes/<DocTypeName>/<DocTypeName>.json   ← обов'язково
apps/<app>/<module>/doctypes/<DocTypeName>/<doctypename>.py  ← опціонально (контролер)
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
| `search_fields` | list[str] | Поля для пошуку в списку й Link-полі (окрім `name` і `title_field`, які шукаються завжди) |
| `in_global_search` | bool | `false` — документи не потрапляють у глобальний пошук (Ctrl+K). Default `true` |

---

## Пошук

Усі поля пошуку (список, Link-dropdown, дерево, віртуальні DocType, глобальний пошук) мають однакову семантику (`grunt/search/terms.py`):

- запит ділиться на слова; рядок підходить, коли **кожне** слово знайдено в **якомусь** полі, в будь-якому порядку («Іван Петренко» знайде запис з іменем і прізвищем у різних полях);
- регістр не важливий (кирилиця теж), `%` і `_` шукаються буквально;
- список і Link шукають підрядок у `name`, `title_field` і `search_fields` (Link-поля в `search_fields` — за назвою пов'язаного документа);
- глобальний пошук — повнотекстовий індекс `grunt_search_index` (SQLite FTS5 / PostgreSQL tsvector / MySQL FULLTEXT): слово шукається як **початок** слова («петр» → «Петренко»), результати ранжуються, збіг у назві вище. Індексуються поля, чий тип `searchable`; показуються лише документи, які користувач може відкрити (з урахуванням row-level прав).

Коли ставити `"in_global_search": false`: службові записи (зв'язки, версії, підписки), телеметрія, великі довідники (КАТОТТГ) — вони засмічують видачу Ctrl+K. Пошук у списку й Link-полі для них працює як і раніше. Записи вже вимкнених DocType прибирає `grunt doctype sync` / `grunt migrate`; `is_log`, `is_child`, `is_virtual` не індексуються взагалі.

Пошук у віртуальному DocType робить сам `VirtualStore` (`search_fields` сховища), у SQL — `match_columns(search, columns, dialect)` з `grunt.search.terms`, а не `col.ilike(f"%{search}%")`.

---

## Типи полів (fieldtype)

### Прості типи

| fieldtype | БД | Опис |
|-----------|-----|------|
| `"Data"` | varchar(255) | Короткий текст, 1 рядок (`options` — автопідказки) |
| `"LongText"` | text | Довгий текст |
| `"Int"` | integer | Ціле число; `"options": "Bytes"` — розмір, показується як `1.4 MB` |
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
| `"Attach"` | Посилання на завантажений файл (типи понад вбудовані, напр. `.exe`, — у `SystemSettings.allowed_file_types`) |
| `"Image"` | Зображення |

### Структурні (не в БД — лише для UI)

| fieldtype | Опис |
|-----------|------|
| `"Section"` | Розділяє форму на секції. Ширина будь-якого поля в колонці — `"columns": 1–12` (6+6, 4+4+4 стають в один рядок) |
| `"Column"` | Колонка (до 4 поруч; ширина — `columns` у дванадцятих, без неї колонки рівні) |
| `"Tab"` | Вкладка в формі |
| `"Table"` | Дочірня таблиця (вкажи `"options": "ChildDocTypeName"`) |

### Спеціальні

| fieldtype | Опис |
|-----------|------|
| `"Select"` | Dropdown. Опції через `\n` в `"options"` |
| `"RichText"` | HTML редактор (Tiptap) |
| `"HTMLView"` | HTML, сформований сервером (таблиця, звіт) — показується як розмітка, лише перегляд; санітизується при записі |
| `"JSON"` | JSONB поле |
| `"Code"` | Code editor |
| `"Color"` | Color picker |
| `"Signature"` | Поле підпису |
| `"Geolocation"` | GPS координати |

---

## Зарезервовані назви полів

Не називай поля `name`, `owner`, `created_at`, `modified_at`, `modified_by`, `docstatus`, `doctype` (системні колонки) і `user`, `data`, `session`, `engine`, `doc_before_save`: це атрибути контролера `Document`, тож `self.user` поверне поточного користувача, а не значення поля. Для «чий це запис» бери `account` або `member`.

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
    {"fieldname": "product", "label": "Товар", "fieldtype": "Data", "required": true},
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

## Статуси: кольори та іконки

Select-поле статусу показується кольоровим бейджем з іконкою (список, шапка форми, канбан):

```json
"status_field": "status",
"status_indicators": [
  {"value": "Чернетка",     "color": "secondary", "icon": "pencil-line"},
  {"value": "На розгляді",  "color": "info",      "icon": "loader"},
  {"value": "Затверджено",  "color": "success",   "icon": "circle-check"},
  {"value": "Призупинено",  "color": "warn",      "icon": "circle-pause"},
  {"value": "Скасовано",    "color": "danger",    "icon": "circle-x"}
]
```

- `status_field` — fieldname Select-поля (не обов'язково `status`); `value` — точне значення опції.
- `color`: `secondary` (сірий), `info` (синій), `success` (зелений), `warn` (жовтий),
  `danger` (червоний), `contrast`; значення без індикатора — сірі.
- `icon` — Lucide (lowercase, дефіс); `label` — інший підпис замість значення.
- Бейдж малюється для Select-поля; для «Активний/Неактивний» з бейджем — Select, не Check.

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
