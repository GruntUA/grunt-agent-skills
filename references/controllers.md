# Контролери у Ґрунт

Контролери реалізують серверну бізнес-логіку для DocType через Python клас.

## Розміщення та структура

```
grunt_apps/<app>/doctypes/<DocTypeName>.py
```

Файл розміщується **поряд** із JSON файлом DocType.

```python
from grunt.core.document.controller import Document

class Customer(Document):
    async def validate(self) -> None:
        ...

    async def after_insert(self) -> None:
        ...
```

**Правила іменування:**
- Назва класу = назва DocType (PascalCase, без пробілів)
- Наприклад: DocType `"Sales Order"` → клас `SalesOrder`

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
    print(self.full_name)   # значення поля full_name
    print(self.status)      # значення поля status
    print(self.id)          # унікальний ID документа
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
        user=grunt.session.user,
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
            title="Помилка видалення"
        )
```

---

## Робота з пов'язаними документами

```python
async def after_insert(self) -> None:
    # Створити пов'язаний запис
    log = await grunt.new_doc("ActivityLog", {
        "doctype_ref": "Customer",
        "doc_id": self.id,
        "action": "created",
        "user": grunt.session.user,
    })
```

---

## Доступ до сесії користувача

```python
async def validate(self) -> None:
    # Перевірка ролі
    if not grunt.session.has_role("Manager", "Admin"):
        grunt.throw("Недостатньо прав для цієї операції")

    # Автоматичне присвоєння
    if not self.assigned_to:
        self.assigned_to = grunt.session.user
```

---

## Анти-паттерни

- ❌ **Не використовуй `grunt.db.set_value()`** для полів з валідацією — обхід усіх lifecycle hooks
- ❌ **Не роби синхронних операцій** в async методах — завжди `await`
- ❌ **Не пиши бізнес-логіку в API endpoints** — тільки в контролерах або service шарі
- ❌ **Не catch-суй всі виключення** — дозволь Ґрунт обробляти HTTP помилки
- ❌ Не залишай `print()` в production коді — використовуй `structlog`
