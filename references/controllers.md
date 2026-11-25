# Контролери у Ґрунт

Контролери реалізують серверну бізнес-логіку для DocType через Python клас.

## Розміщення та структура

```
apps/<app>/<module>/doctypes/<DocTypeName>/<doctypename>.py
```

Файл розміщується **поряд** із JSON файлом DocType.

```python
from grunt.document.base import Document


class Customer(Document):
    async def validate(self) -> None: ...

    async def after_insert(self) -> None: ...
```

**Правила іменування:**
- Назва класу = назва DocType (PascalCase, без пробілів)
- Наприклад: DocType `"Sales Order"` → клас `SalesOrder`

## Типізовані поля й конструктор

`grunt doctype sync-types <Name> --app <app>` пише в клас блок анотацій полів. `Document` — `dataclass_transform`, тож ці анотації типізують і конструктор (як у dataclass), без жодного дублювання:

```python
class Customer(Document):
    # begin: auto-generated types
    full_name: str                 # обов'язкове при створенні (required, без default)
    email: str | None = None       # необов'язкове
    contacts: list[CustomerContact] = []  # Table: дочірні рядки — документи дочірнього контролера
    # end: auto-generated types

c = Customer(full_name="Іван", email="i@x.ua")   # не збережено → await c.insert()
c = await Customer.create(full_name="Іван")      # збережено, перевіряє право create
doc = await Customer.get_doc(name)                # читання як Customer (find_doc, get_list — так само)
row_doc = Customer(**row)                         # готовий рядок (dict[str, Any])
```

- `= None` / `= []` — лише для pyright: `__init_subclass__` прибирає їх з класу, поле читається з `self.data`.
- Не писати `Customer(doctype="Customer", data=...)` чи `Customer("Customer", {...})` — pyright позначить; готовий рядок — `Customer(**row)`. Позиційний словник `Customer({...})` pyright не прийме (типізація неможлива без дублювання полів). `**{літерал}` pyright звіряє за типом значень — краще іменовані аргументи.
- Не дублювати анотації полів руками поза блоком (pyright: redeclaration).
- Атрибути класу з `_` (`_prev: dict | None = None`) — звичайні атрибути, їх не чіпає.

---

## Lifecycle hooks (порядок виклику)

### INSERT (створення нового документа)

1. `validate()` — перевірка даних перед збереженням
2. `before_insert()` — до запису в БД
3. `after_insert()` — після успішного запису в БД

### UPDATE (оновлення)

1. `validate()` — перевірка даних
2. `before_save()` — до оновлення в БД
3. `after_save()` — після успішного оновлення

### DELETE (видалення)

1. `before_delete()` — до видалення з БД
2. `after_delete()` — після успішного видалення

---

## Доступ до полів документа

Всередині методів контролера поля доступні через `self`:

```python
async def validate(self) -> None:
    print(self.full_name)  # значення поля full_name
    print(self.status)  # значення поля status
    print(self.id)  # унікальний ID документа
```

---

## grunt.throw() — user-facing помилка

```python
grunt.throw(message, title=None)
```

Генерує HTTP 422 з повідомленням для користувача:

```python
async def validate(self) -> None:
    if not self.email:
        grunt.throw("Email є обов'язковим")

    if self.amount <= 0:
        grunt.throw("Сума має бути більше нуля", title="Помилка валідації")
```

---

## Приклади lifecycle hooks

### validate — перевірка даних

```python
async def validate(self) -> None:
    # Перевірка формату email
    if self.email and "@" not in self.email:
        grunt.throw("Невірна адреса email")

    # Перевірка числових обмежень
    if self.discount and self.discount > 100:
        grunt.throw("Знижка не може перевищувати 100%")

    # Встановлення значень за замовчуванням
    if not self.status:
        self.status = "Чернетка"
```

### before_insert — підготовка перед створенням

```python
async def before_insert(self) -> None:
    # Генерація унікального номера
    count = await grunt.count("Order", filters={})
    self.order_number = f"ORD-{count + 1:05d}"
```

### after_insert — дії після створення

```python
async def after_insert(self) -> None:
    # Сповіщення менеджерів
    await grunt.notify(
        users=["manager@company.com"],
        subject=f"Нове замовлення #{self.order_number}",
        message=f"Замовлення від {self.customer} потребує підтвердження.",
        doctype="Order",
        doc_id=self.id,
    )

    # Миттєве повідомлення поточному користувачу
    await grunt.publish(
        user=grunt.get_user().email,
        event="msgprint",
        message="Замовлення успішно створено!",
        type="success",
    )
```

### before_save — перед оновленням

```python
async def before_save(self) -> None:
    # Автоматичний розрахунок
    if self.qty and self.price:
        self.total = self.qty * self.price
```

### after_save — після оновлення

```python
async def after_save(self) -> None:
    # Якщо статус змінився на "Завершено"
    if self.status == "Завершено":
        await grunt.notify(
            users=[self.assigned_to],
            subject="Задачу завершено",
            message=f"Задача '{self.title}' позначена як завершена.",
            doctype=self.doctype,
            doc_id=self.id,
        )
```

### before_delete — перевірка перед видаленням

```python
async def before_delete(self) -> None:
    # Заборона видалення, якщо є пов'язані записи
    count = await grunt.count("Order", filters={"customer": self.name})
    if count > 0:
        grunt.throw(
            f"Неможливо видалити клієнта: існує {count} замовлень",
            title="Помилка видалення",
        )
```

---

## Робота з пов'язаними документами

```python
async def after_insert(self) -> None:
    # Створити пов'язаний запис
    log = await grunt.new_doc(
        "ActivityLog",
        {
            "doctype_ref": "Customer",
            "doc_id": self.id,
            "action": "created",
            "user": grunt.get_user().email,
        },
    )
```

---

## Доступ до сесії користувача

```python
async def validate(self) -> None:
    # Перевірка ролі
    if not grunt.get_user().has_role("Manager", "Admin"):
        grunt.throw("Недостатньо прав для цієї операції")

    # Автоматичне присвоєння
    if not self.assigned_to:
        self.assigned_to = grunt.get_user().email
```

---

## Анти-паттерни

- ✅ **Методи класу**: Розміщуй бізнес-логіку в кастомних методах контролера (DocType), а не в API endpoints.
- ❌ **Не використовуй `grunt.db.set_value()`** для полів з валідацією — обхід усіх lifecycle hooks
- ❌ **Не роби синхронних операцій** в async методах — завжди `await`
- ❌ **Не пиши бізнес-логіку в API endpoints** — тільки в контролерах або service шарі
- ❌ **Не catch-суй всі виключення** — дозволь Ґрунт обробляти HTTP помилки
- ❌ Не залишай `print()` в production коді — використовуй `structlog`
