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
- Фонових задач (Celery, `@retryable_task`)
- Frontend на Vue 3 + grunt-ui
- CLI команд (`grunt serve`, `grunt db migrate`...)
- Написання тестів (pytest, vitest)

## Два головних workflow

### 1. Новий додаток
Розпочни з [new-app.md](./references/new-app.md)

### 2. Існуючий додаток
Розпочни з [existing-app.md](./references/existing-app.md)

---

## Таблиця референсів

Завантажуй **лише** той файл, який стосується поточної задачі.

| Тема | Референс |
|------|----------|
| Новий додаток (scaffold) | [new-app.md](./references/new-app.md) |
| Існуючий додаток (розширення) | [existing-app.md](./references/existing-app.md) |
| DocType: визначення, типи полів | [doctypes.md](./references/doctypes.md) |
| Контролери: lifecycle hooks | [controllers.md](./references/controllers.md) |
| Хуки: `@on` декоратор | [hooks.md](./references/hooks.md) |
| REST API + GruntApp SDK | [api.md](./references/api.md) |
| База даних: `grunt.db.*` | [database.md](./references/database.md) |
| Дозволи: RBAC, ролі | [permissions.md](./references/permissions.md) |
| Фонові задачі: Celery | [background-tasks.md](./references/background-tasks.md) |
| Frontend: Vue 3 + grunt-ui | [frontend.md](./references/frontend.md) |
| CLI команди | [grunt-cli.md](./references/grunt-cli.md) |
| Логування: structlog, BackgroundTaskLog, ActivityLog | [logging.md](./references/logging.md) |
| Тестування | [testing.md](./references/testing.md) |

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

## Структура відповіді API

```json
// Успіх
{"success": true, "data": {...}, "meta": {"total": 100, "page": 1, "per_page": 20}}

// Помилка
{"success": false, "error": {"code": "VALIDATION_ERROR", "message": "...", "details": []}}
```
