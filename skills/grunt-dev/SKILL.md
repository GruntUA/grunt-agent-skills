---
name: grunt-dev
description: Skill для розробки додатків на Ґрунт (metadata-driven framework). Активується при роботі з DocType, controllers, hooks, grunt.db, REST API, Vue 3 frontend, CLI командами та тестуванням.
---

# Ґрунт — Skill для розробника додатків

Цей skill допомагає будувати повноцінні додатки на базі **Ґрунт** — metadata-driven framework (Python/FastAPI/Vue 3).

## Активація

Skill активується, коли задача стосується:
- Створення або зміни DocType
- Написання контролерів (validate, after_insert тощо)
- Реєстрації хуків (`@on`)
- REST API або GruntApp SDK (`grunt.get_doc`, `grunt.new_doc`...)
- Роботи з базою даних через `grunt.db`
- Налаштування дозволів (RBAC, ролі)
- Фонових задач (TaskIQ, `@retryable_task`)
- Frontend на Vue 3 + grunt-ui
- CLI команд (`grunt serve`, `grunt db migrate`...)
- Написання тестів (pytest, vitest)

## Два головних workflow

### 1. Новий додаток
Розпочни з [new-app.md](../../references/new-app.md)

### 2. Існуючий додаток
Розпочни з [existing-app.md](../../references/existing-app.md)

---

## Таблиця референсів

Завантажуй **лише** той файл, який стосується поточної задачі.

| Тема | Референс |
|------|----------|
| Новий додаток (scaffold) | [new-app.md](../../references/new-app.md) |
| Існуючий додаток (розширення) | [existing-app.md](../../references/existing-app.md) |
| DocType: визначення, типи полів | [doctypes.md](../../references/doctypes.md) |
| Контролери: lifecycle hooks | [controllers.md](../../references/controllers.md) |
| Хуки: `@on` декоратор | [hooks.md](../../references/hooks.md) |
| REST API + GruntApp SDK | [api.md](../../references/api.md) |
| База даних: `grunt.db.*` | [database.md](../../references/database.md) |
| Дозволи: RBAC, ролі | [permissions.md](../../references/permissions.md) |
| Фонові задачі: TaskIQ | [background-tasks.md](../../references/background-tasks.md) |
| Frontend: Vue 3 + grunt-ui | [frontend.md](../../references/frontend.md) |
| CLI команди | [grunt-cli.md](../../references/grunt-cli.md) |
| Логування: structlog, BackgroundTaskLog, ActivityLog | [logging.md](../../references/logging.md) |
| Тестування | [testing.md](../../references/testing.md) |

---

## Загальні правила

- **Завжди** запускай `grunt doctype sync <Name>` після будь-яких змін у DocType
- **Ніколи** не пиши raw SQL — тільки SQLAlchemy або `grunt.db.*`
- **Ніколи** не пиши міграції вручну — вони генеруються автоматично через sync
- **Завжди** async/await — Ґрунт є async-first (FastAPI + SQLAlchemy 2.0)
- **Завжди** type hints в Python і TypeScript
- Логуй через `structlog`, не через `print()` чи `logging.basicConfig()`
- Секрети — тільки через `config.py` + env змінні, ніколи в коді
- Бізнес-логіку — в сервісному шарі, не в API endpoints напряму
- **Строго дотримуйся стандарту PEP8** для будь-якого коду на Python.
- **Використовуй максимально нові версії** всіх пакетів і фреймворків, включно з найсвіжішими стабільними версіями Python та Node.js.

## Імпорти (Python 3.15)

- **Не пиши** `from __future__ import annotations` і не бери анотації в лапки: з 3.14 анотації обчислюються відкладено (PEP 649).
- **Не пиши** `if TYPE_CHECKING:` і не ховай імпорти в тіло функції заради циклів. Пиши `lazy from grunt.x import Y` нагорі модуля (PEP 810): модуль завантажиться при першому зверненні до імені.
- `lazy` став лише там, де він щось дає: для модулів `grunt.*`, які можуть утворити цикл, і для важких опційних бібліотек (openpyxl, PIL…). Стандартну бібліотеку й пакети, які й так завантажуються при старті (sqlalchemy, fastapi, pydantic…), імпортуй звичайно.
- Імпорт опційної залежності в `try/except ImportError` лишається всередині функції: `lazy` не можна ставити в `try`.
- **Пастка з тестами:** імʼя з `lazy from m import f` привʼязується при першому виклику, тож `monkeypatch.setattr(m, "f", …)` після цього до коду вже не дійде. Якщо функцію підміняють у тестах, імпортуй модуль (`lazy from grunt.storage.cloud import rclone` → `rclone.is_installed()`) або патч імʼя в модулі-споживачі.
- Не пиши `encoding="utf-8"` в `open()` / `read_text()` / `write_text()`: у 3.15 UTF-8 і так за замовчуванням (PEP 686).

## Структура відповіді API

```json
// Успіх
{"success": true, "data": {...}, "meta": {"total": 100, "page": 1, "per_page": 20}}

// Помилка
{"success": false, "error": {"code": "VALIDATION_ERROR", "message": "...", "details": []}}
```

## Готовність до модернізації

- **Пропонуй стратегію модернізації коду** для підтримки оновлення інфраструктури (пакетів, Python, Node.js) до новітніх версій.
- Якщо бачиш, що можна покращити апі, клі, будь-який компонент, документацію, тести - запропонуй покращення. Якщо користувач погодиться - зроби це.
- Якщо бачиш, що можна покращити skill - запропонуй покращення. Якщо користувач погодиться - зроби це. Зверни увагу - зміни треба вносити в grunt-agent-skills.
