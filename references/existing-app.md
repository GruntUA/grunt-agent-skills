# Існуючий додаток на Ґрунт

Використовуй цей workflow, коли потрібно розширити, змінити або виправити додаток, який вже існує.

> [!TIP]
> **Принцип "Everything is a DocType"**: Перед внесенням змін подумай, як максимально зав'язати новий функціонал на DocType. Контролери та нові сторінки мають бути частиною DocType.

## Чеклист

```
- [ ] Крок 1: Знайти та підтвердити директорію apps/
- [ ] Крок 2: Знайти потрібний додаток
- [ ] Крок 3: Прочитати app.json та поточну структуру
- [ ] Крок 4: Внести зміни
- [ ] Крок 5: Sync та перевірити
```

---

## Крок 1: Знайти директорію apps/

```bash
ls apps/
```

Якщо воркспейс відкритий всередині додатку (наприклад `apps/my_crm/`), перейди вище:

```bash
ls ../../apps/
```

Або пошук за характерними файлами:

```bash
ls apps/ sites/
```

**Не створюй новий додаток** (`grunt app create`), якщо вже є існуючий.

---

## Крок 2: Знайти потрібний додаток

```bash
ls apps/
```

Знайди директорію додатку та ознайомся з її структурою:

```bash
ls apps/<name>/
ls apps/<name>/<name>/doctypes/
```

---

## Крок 3: Прочитати поточну структуру

Перед будь-якими змінами прочитай ключові файли:

```bash
cat apps/<name>/app.json                  # маніфест: name, title, version, modules
ls apps/<name>/<name>/doctypes/            # наявні DocType (тека на кожен)
cat apps/<name>/<name>/hooks.py           # наявні хуки
cat apps/<name>/<name>/tasks.py           # наявні фонові задачі
```

Типова структура існуючого додатку (same-name layout):

```
apps/<name>/
├── app.json                        ← маніфест: name, title, version, modules, depends_on
├── <name>/                         ← Python-пакет застосунку (== name)
│   ├── __init__.py
│   ├── hooks.py                    ← doc_events, scheduler_events, io_exporters, …
│   ├── routes.py                   ← APIRouter → /api/v1/app/<name> (опц.)
│   ├── tasks.py                    ← фонові задачі
│   ├── documents.py               ← override-контролери (опц.)
│   └── doctypes/
│       ├── Customer/
│       │   ├── Customer.json       ← визначення DocType
│       │   └── customer.py         ← контролер (опц., ім'я файлу — snake_case)
│       └── Order/
│           └── Order.json
├── public/                         ← статика → /assets/<name>  (опц.)
└── www/                            ← вебсторінки  (опц.)
```

Багатомодульний застосунок повторює `<module>/` замість одного `<name>/` для кожного запису в `modules`.

---

## Крок 4: Внести зміни

### Додати нове поле до DocType

1. Відкрий `apps/<name>/<name>/doctypes/<DocType>/<DocType>.json`
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
# apps/<name>/<name>/doctypes/NewModel/NewModel.json
# apps/<name>/<name>/doctypes/NewModel/new_model.py  (опціонально)
```

Докладніше → [doctypes.md](./doctypes.md)

### Змінити контролер

Якщо `<DocType>.py` не існує — створи його поряд із JSON.
Якщо існує — читай його перед змінами, щоб зрозуміти наявну логіку.

Докладніше → [controllers.md](./controllers.md)

### Додати хук

Додай функцію до `apps/<name>/<module>/hooks.py`:

```python
from grunt.hooks import on_doc


@on_doc("Customer", "before_save")
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
