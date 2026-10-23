# Grunt CLI

Командний рядок для управління Ґрунт-проєктом та додатками.

> Використовуй `grunt` без повного шляху — CLI встановлено глобально.
> Повний список: `grunt --help`; довідка по групі: `grunt <group> --help`.

---

## Запуск сервера

```bash
grunt serve                    # backend (FastAPI :8000) + frontend (Vite :5173)
grunt serve --no-frontend      # тільки FastAPI
grunt serve --port 9000        # інший порт API
grunt serve --reload           # авто-перезапуск (увімкнено за замовчуванням)
grunt worker                   # воркер фонових задач (TaskIQ)
```

Після запуску:
- API: `http://localhost:8000`
- Swagger UI: `http://localhost:8000/api/docs`
- Frontend: `http://localhost:5173`

---

## Управління додатками

```bash
grunt app create <name>        # scaffold нового додатку в apps/ (інтерактивно)
grunt app install <name>       # встановити додаток на сайт (DocTypes, fixtures, workspace)
grunt app uninstall <name>     # видалити додаток із сайту та очистити його дані
grunt app remove <name>        # прибрати додаток з bench: pip-пакет + apps/<name> → archive/apps/ (спершу uninstall на сайтах)
grunt app list                 # список встановлених додатків
grunt app doctor [name]        # перевірити/виправити структуру додатків
```

---

## База даних та схема

```bash
grunt migrate                  # синхронізувати схему: Alembic + system tables + DocType tables + fixtures
grunt migrate --dry-run        # показати SQL без виконання
grunt migrate --no-alembic     # пропустити Alembic-історію
grunt db migrate               # те саме, що grunt migrate
grunt db backup                # резервна копія (pg_dump або sqlite3)
grunt db trim-tables           # прибрати колонки, яких немає в метаданих DocType
```

Запускай `grunt migrate` після: оновлення фреймворку, встановлення додатків із DocType, змін у fixture-файлах.

---

## DocType

```bash
grunt doctype list             # список усіх зареєстрованих DocType
grunt doctype scaffold <Name>  # створити новий DocType із шаблонів
grunt doctype sync <Name>      # перечитати JSON з диску та оновити БД
grunt doctype sync-types <Name># оновити auto-generated типи в контролері з JSON
```

`grunt doctype sync <Name>` — обов'язково після **будь-якої** зміни JSON DocType.

---

## Користувачі

```bash
grunt users create             # створити користувача
grunt users set-password       # змінити пароль
grunt users list               # список користувачів
```

---

## Сайти

```bash
grunt site create <name>       # створити сайт
grunt site list                # усі сайти в bench
grunt site use <name>          # активний сайт за замовчуванням для CLI
grunt site info                # конфігурація сайту
grunt site delete <name>       # видалити сайт та всі його дані
```

---

## Скіли додатків

```bash
grunt skills list              # локальні скіли з усіх додатків
grunt skills install [app]     # встановити скіли з додатку (симлінк)
grunt skills uninstall <name>  # прибрати встановлений скіл
```

---

## Тести та якість коду

```bash
grunt test                     # тести фреймворку
grunt test <app>               # тести конкретного додатку
grunt test <app> --doctype <Name>   # тести конкретного DocType
grunt lint                     # ruff (Python) + vue-tsc (TS/Vue)
```

---

## Оновлення та ініціалізація

```bash
grunt update                   # оновити фреймворк, додатки та залежності
grunt init                     # заглушка ініціалізації проєкту
grunt ui                       # керування shadcn-vue UI компонентами
```

---

## Конфігурація

Налаштування зчитуються з `.env` або змінних оточення:

```bash
DATABASE_URL=postgresql+asyncpg://user:pass@localhost/grunt_db
REDIS_URL=redis://localhost:6379/0     # без неї — in-memory брокер задач
SECRET_KEY=your-secret-key
DEBUG=true
```

---

## Оновлення коду та залежностей

```bash
grunt update                   # оновити код (git pull), пакети в межах lock-файлів, міграції
grunt upgrade                  # оновити всі залежності (uv, mise, python, node, rclone, пакети) до найновіших версій та оновити файли версій (package.json, pyproject.toml, mise.toml)
grunt upgrade --dry-run        # перегляд доступних оновлень без внесення змін
grunt upgrade runtimes         # тільки системні рантайми (mise, uv, node, python, rclone)
grunt upgrade packages         # тільки пакети Python та Node.js
```

---

## Типовий workflow розробника

```bash
# 1. Запустити сервер
grunt serve

# 2. Після зміни DocType JSON
grunt doctype sync Customer

# 3. Після встановлення додатку / зміни fixtures / оновлення фреймворку
grunt migrate

# 4. Тести
grunt test
```
