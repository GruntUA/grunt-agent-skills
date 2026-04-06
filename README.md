# grunt-agent-skills

Agent skill для розробки повноцінних додатків на базі [Ґрунт](https://github.com/GruntUA/Grunt) — metadata-driven framework (Python/FastAPI/Vue 3).

## Встановлення

### Claude Code Marketplace (рекомендовано)

```bash
/plugin marketplace add GruntUA/grunt-agent-skills
/plugin install grunt-dev@grunt-agent-skills
```

Після встановлення skill доступний як `/grunt-dev:grunt-dev`.

### Skills CLI (legacy)

```bash
npx skills add GruntUA/grunt-agent-skills --skill grunt-dev
```

## Що покриває

- **DocType**: визначення моделей, типи полів, Child DocType, sync з БД
- **Controllers**: lifecycle hooks (validate, after_insert, before_save...), grunt.throw()
- **Hooks**: `@on` декоратор, cross-cutting логіка між DocTypes
- **REST API + GruntApp SDK**: grunt.get_doc, grunt.new_doc, grunt.get_list, grunt.db.*
- **Database**: grunt.db helpers, фільтри, bulk операції
- **Permissions**: RBAC, ролі, grunt.session, row-level security
- **Background tasks**: Celery + @retryable_task, розклад в app.json
- **Frontend**: Vue 3 + grunt-ui, composables (useDocType, useDocument, useList), Pinia
- **CLI**: grunt serve, db init/migrate, doctype sync, create-app
- **Testing**: pytest (backend), vitest (Vue), Playwright (E2E)

## Використання

Після встановлення skill активується автоматично, коли ти запитуєш про:

- Створення або зміну DocType
- Написання контролерів і хуків
- Роботу з базою даних через `grunt.db.*`
- REST API ендпоінти Ґрунт
- CLI команди (`grunt serve`, `grunt doctype sync`...)
- Frontend компоненти на Vue 3 + grunt-ui
- Написання тестів

## Структура

```
grunt-agent-skills/
├── .claude-plugin/
│   ├── plugin.json         ← маніфест плагіна для Claude marketplace
│   └── marketplace.json    ← каталог маркетплейсу
├── skills/
│   └── grunt-dev/
│       └── SKILL.md        ← skill для Claude Code plugin system
├── SKILL.md                ← skill для legacy skills CLI
└── references/
    ├── new-app.md          ← крок за кроком: новий додаток
    ├── existing-app.md     ← розширення існуючого додатку
    ├── doctypes.md         ← DocType: JSON, типи полів, sync
    ├── controllers.md      ← lifecycle hooks, validate, after_insert
    ├── hooks.md            ← @on декоратор, глобальні хуки
    ├── api.md              ← REST API + GruntApp SDK
    ├── database.md         ← grunt.db.*, фільтри, запити
    ├── permissions.md      ← RBAC, ролі, grunt.session
    ├── background-tasks.md ← Celery, @retryable_task, розклад
    ├── frontend.md         ← Vue 3 + grunt-ui + composables
    ├── grunt-cli.md        ← всі CLI команди
    └── testing.md          ← pytest, vitest, Playwright
```
