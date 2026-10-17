# Тестування у Ґрунт

## Backend тести (Python / pytest)

### Розміщення

```
backend/tests/
├── conftest.py              ← fixtures та налаштування
├── test_customer.py         ← тести для Customer DocType
├── test_order.py
└── <app>/
    └── test_<feature>.py    ← тести конкретного додатку
```

### Запуск

```bash
pytest backend/tests/                    # всі тести
pytest backend/tests/ -v                 # з детальним виводом
pytest backend/tests/test_customer.py    # конкретний файл
pytest backend/tests/ -k "test_create"  # фільтр за назвою
pytest backend/tests/ --tb=short        # коротке трасування помилок
```

### Структура тесту

```python
import pytest
from httpx import AsyncClient
from grunt.main import app
from grunt.core.db.session import get_session

@pytest.fixture
async def client():
    async with AsyncClient(app=app, base_url="http://test") as ac:
        yield ac

@pytest.fixture
async def auth_headers(client):
    """Отримати JWT токен тестового користувача."""
    response = await client.post("/api/v1/auth/login", json={
        "email": "test@example.com",
        "password": "testpass"
    })
    token = response.json()["data"]["access_token"]
    return {"Authorization": f"Bearer {token}"}
```

### Тест CRUD операцій

```python
@pytest.mark.asyncio
async def test_create_customer(client, auth_headers):
    response = await client.post(
        "/api/v1/docs/Customer",
        json={"full_name": "Іван Іванов", "email": "ivan@test.com"},
        headers=auth_headers,
    )
    assert response.status_code == 200
    data = response.json()
    assert data["success"] is True
    assert data["data"]["full_name"] == "Іван Іванов"

@pytest.mark.asyncio
async def test_list_customers(client, auth_headers):
    response = await client.get(
        "/api/v1/docs/Customer?filter[status]=Активний",
        headers=auth_headers,
    )
    assert response.status_code == 200
    data = response.json()
    assert data["success"] is True
    assert isinstance(data["data"], list)

@pytest.mark.asyncio
async def test_validation_error(client, auth_headers):
    response = await client.post(
        "/api/v1/docs/Customer",
        json={"email": "not-an-email"},  # без full_name (required)
        headers=auth_headers,
    )
    assert response.status_code == 422
    data = response.json()
    assert data["success"] is False
```

### Тест контролера / бізнес-логіки

```python
@pytest.mark.asyncio
async def test_invalid_email_throws(client, auth_headers):
    """Контролер Customer.validate() відхиляє невірний email."""
    response = await client.post(
        "/api/v1/docs/Customer",
        json={"full_name": "Тест", "email": "invalid-email"},
        headers=auth_headers,
    )
    assert response.status_code == 422
    assert "email" in response.json()["error"]["message"].lower()
```

### Fixtures для тестових даних

```python
# conftest.py
import pytest

@pytest.fixture
async def sample_customer(client, auth_headers):
    """Створити тестового клієнта та видалити після тесту."""
    response = await client.post(
        "/api/v1/docs/Customer",
        json={"full_name": "Тестовий Клієнт", "email": "test-customer@test.com"},
        headers=auth_headers,
    )
    customer = response.json()["data"]
    yield customer
    # Cleanup
    await client.delete(
        f"/api/v1/docs/Customer/{customer['id']}",
        headers=auth_headers,
    )
```

---

## Frontend тести (vitest)

### Запуск

```bash
npm test              # watch mode
npm run test:run      # одноразово
npm run test:coverage # з покриттям коду
```

### Структура тесту компонента

```typescript
// CustomerForm.test.ts
import { describe, it, expect, vi } from 'vitest'
import { mount } from '@vue/test-utils'
import CustomerForm from '@/components/CustomerForm.vue'

describe('CustomerForm', () => {
  it('відображає поля форми', () => {
    const wrapper = mount(CustomerForm, {
      props: { modelValue: {} }
    })
    expect(wrapper.find('input[name="full_name"]').exists()).toBe(true)
    expect(wrapper.find('input[name="email"]').exists()).toBe(true)
  })

  it('emits update при зміні значення', async () => {
    const wrapper = mount(CustomerForm, {
      props: { modelValue: {} }
    })
    await wrapper.find('input[name="full_name"]').setValue('Іван')
    expect(wrapper.emitted('update:modelValue')).toBeTruthy()
  })
})
```

### Мок API

```typescript
import { vi } from 'vitest'
import { api } from '@/core/api'

vi.mock('@/core/api', () => ({
  api: {
    get: vi.fn().mockResolvedValue({
      data: { success: true, data: [{ id: '1', full_name: 'Тест' }] }
    })
  }
}))
```

---

## E2E тести (Playwright)

### Запуск

```bash
npm run e2e                    # всі E2E тести
npm run e2e -- --headed        # з відкритим браузером
npm run e2e -- test/customer.spec.ts  # конкретний файл
```

### Структура E2E тесту

```typescript
// e2e/customer.spec.ts
import { test, expect } from '@playwright/test'

test('створити нового клієнта', async ({ page }) => {
  await page.goto('/desk/Customer')
  await page.click('button:has-text("Новий")')
  await page.fill('input[name="full_name"]', 'Іван Іванов')
  await page.fill('input[name="email"]', 'ivan@test.com')
  await page.click('button:has-text("Зберегти")')
  await expect(page.locator('.success-toast')).toBeVisible()
})
```

---

## Правила тестування

- Кожен новий endpoint або бізнес-правило **має** мати тест
- Тести мають використовувати тестову БД — ніколи не production
- Cleanup після кожного тесту (fixtures pattern)
- Уникай тестування implementation details — тестуй поведінку
