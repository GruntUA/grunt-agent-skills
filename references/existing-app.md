# Існуючий додаток на Ґрунт

Використовуй цей workflow, коли потрібно розширити, змінити або виправити додаток, який вже існує.

> [!TIP]
> **Принцип "Everything is a DocType"**: Перед внесенням змін подумай, як максимально зав'язати новий функціонал на DocType. Контролери та нові сторінки мають бути частиною DocType.

## Чеклист

```
- [ ] Крок 1: Знайти та підтвердити директорію grunt_apps/
- [ ] Крок 2: Знайти потрібний додаток
- [ ] Крок 3: Прочитати app.json та поточну структуру
- [ ] Крок 4: Внести зміни
- [ ] Крок 5: Sync та перевірити
```

---

## Крок 1: Знайти директорію grunt_apps/

```bash
ls grunt_apps/
```

Якщо воркспейс відкритий всередині додатку (наприклад `grunt_apps/my_crm/`), перейди вище:

```bash
ls ../../grunt_apps/
```

Або пошук за характерними файлами:

```bash
ls grunt_apps/ sites/ apps/
```

**Не створюй новий додаток** (`grunt create-app`), якщо вже є існуючий.

---

## Крок 2: Знайти потрібний додаток

```bash
ls grunt_apps/
```

Знайди директорію додатку та ознайомся з її структурою:

```bash
ls grunt_apps/<name>/
ls grunt_apps/<name>/doctypes/
```

---

## Крок 3: Прочитати поточну структуру

Перед будь-якими змінами прочитай ключові файли:

```bash
cat grunt_apps/<name>/app.json           # маніфест: назва, версія, scheduled_tasks
ls grunt_apps/<name>/doctypes/           # наявні DocType (.json + .py)
cat grunt_apps/<name>/hooks/hooks.py     # наявні хуки
cat grunt_apps/<name>/tasks/tasks.py     # наявні фонові задачі
```

Типова структура існуючого додатку:

```
grunt_apps/<name>/
├── app.json                    ← маніфест (назва, label, scheduled_tasks)
├── __init__.py
├── doctypes/
│   ├── Customer.json           ← визначення DocType
│   ├── Customer.py             ← контролер (якщо є)
│   ├── Order.json
│   └── Order.py
├── tasks/
│   ├── __init__.py
│   └── tasks.py
└── hooks/
    ├── __init__.py
    └── hooks.py
```

---

## Крок 4: Внести зміни

### Додати нове поле до DocType

1. Відкрий `grunt_apps/<name>/doctypes/<DocType>.json`
2. Додай поле до масиву `fields`
3. **Ніколи не видаляй наявні поля** без міграції — це призведе до втрати даних

```json
{
  "fieldname": "phone",
  "label": "Телефон",
  "fieldtype": "Text"
}
```

4. Запусти sync (Крок 5)

### Додати новий DocType

Створи нові файли:

```bash
# grunt_apps/<name>/doctypes/NewModel.json
# grunt_apps/<name>/doctypes/NewModel.py  (опціонально)
```

Докладніше → [doctypes.md](./doctypes.md)

### Змінити контролер

Якщо `<DocType>.py` не існує — створи його поряд із JSON.
Якщо існує — читай його перед змінами, щоб зрозуміти наявну логіку.

Докладніше → [controllers.md](./controllers.md)

### Додати хук

Додай функцію до `grunt_apps/<name>/hooks/hooks.py`:

```python
from grunt.core.hooks import on

@on("before_save", doctype="Customer")
async def validate_customer_limit(doc, user, **kwargs):
    count = await grunt.count("Customer", filters={"status": "Клієнт"})
    if count >= 1000:
        grunt.throw("Досягнуто ліміт активних клієнтів")
```

Докладніше → [hooks.md](./hooks.md)

---

## Крок 5: Sync та перевірити

Після зміни **будь-якого** DocType JSON:

```bash
grunt doctype sync <DocTypeName>
```

Після зміни контролерів та хуків — перезапуск сервера (якщо запущено):

```bash
grunt serve
```

Перевір, що API повертає очікувані дані:

```
GET http://localhost:8000/api/v1/docs/<DocType>
```
