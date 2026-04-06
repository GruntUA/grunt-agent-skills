# Grunt CLI

Командний рядок для управління Ґрунт-проєктом та додатками.

> Використовуй `grunt` без повного шляху — CLI встановлено глобально.

---

## Запуск сервера

```bash
grunt serve                    # запустити backend + frontend + Celery
grunt serve --backend-only     # тільки FastAPI (порт 8000)
grunt serve --frontend-only    # тільки Vite (порт 5173)
```

Після запуску:
- API: `http://localhost:8000`
- Swagger UI: `http://localhost:8000/docs`
- Frontend: `http://localhost:5173`

---

## Управління додатками

```bash
grunt create-app <name>        # scaffold нового додатку в grunt_apps/
grunt app list                 # список встановлених додатків
grunt app install <name>       # встановити додаток
grunt app export <name>        # експортувати додаток як архів
```

---

## База даних

```bash
grunt db init                  # ініціалізувати схему БД (перший запуск)
grunt db migrate               # запустити Alembic міграції
grunt db reset                 # скинути та повторно ініціалізувати БД ⚠️ тільки dev
```

> ⚠️ `grunt db reset` видаляє **всі дані**. Ніколи не запускай в production.

---

## DocType

```bash
grunt doctype list             # список всіх зареєстрованих DocType
grunt doctype sync <Name>      # синхронізувати DocType з БД (після змін у JSON)
```

Приклади:

```bash
grunt doctype sync Customer
grunt doctype sync SalesOrder
grunt doctype sync OrderItem
```

---

## Аутентифікація

```bash
grunt auth create-user         # створити нового користувача
grunt auth reset-password      # скинути пароль користувача
```

---

## Fixtures (тестові дані)

```bash
grunt fixtures load            # завантажити fixtures у БД
grunt fixtures dump            # вивантажити поточні дані як fixtures
```

---

## Тестування

```bash
pytest backend/tests/          # запустити Python тести
pytest backend/tests/ -v       # з детальним виводом
pytest backend/tests/test_customer.py  # конкретний файл
npm test                       # Vue unit тести (vitest)
npm run e2e                    # E2E тести (Playwright)
```

---

## Якість коду

```bash
ruff check . --fix             # Python linting + автовиправлення
mypy backend/grunt             # Python type checking
npm run typecheck              # Vue/TypeScript type check
```

---

## Встановлення залежностей

```bash
uv sync                        # Python залежності (через uv)
npm install                    # Node.js залежності
```

---

## Конфігурація

Налаштування зчитуються з `.env` або змінних оточення:

```bash
DATABASE_URL=postgresql+asyncpg://user:pass@localhost/grunt_db
REDIS_URL=redis://localhost:6379/0
SECRET_KEY=your-secret-key
DEBUG=true
```

---

## Типовий workflow розробника

```bash
# 1. Встановити залежності (один раз)
uv sync && npm install

# 2. Ініціалізувати БД (один раз)
grunt db init

# 3. Запустити сервер
grunt serve

# 4. Після зміни DocType JSON
grunt doctype sync Customer

# 5. Запустити тести
pytest backend/tests/
npm test
```
