# Frontend у Ґрунт

Ґрунт використовує **Vue 3** з Composition API, TypeScript та власну UI-бібліотеку `grunt-ui`.

## Структура frontend

```
frontend/src/
├── core/
│   ├── api/             ← API client (axios + Vue Query)
│   ├── builder/         ← Form/View builder engine
│   ├── renderer/        ← Dynamic form renderer
│   └── composables/     ← useDocType, useDocument, useList...
├── components/
│   ├── ui/              ← Design system (Button, Input, Modal...)
│   ├── builder/         ← Builder UI компоненти
│   └── views/           ← ListView, FormView, KanbanView, CalendarView
├── pages/
│   ├── studio/          ← App Studio (builder для розробників)
│   ├── desk/            ← App Desk (UI для кінцевих користувачів)
│   └── auth/            ← Авторизація
└── stores/              ← Pinia stores
```

---

## Стандарт компонента

**Завжди** використовуй `<script setup lang="ts">` з Composition API:

```vue
<script setup lang="ts">
import { ref, computed } from 'vue'
import type { DocType, DocField } from '@/types'

const props = defineProps<{
  doctype: DocType
  modelValue: Record<string, unknown>
}>()

const emit = defineEmits<{
  'update:modelValue': [value: Record<string, unknown>]
}>()

const isLoading = ref(false)
const title = computed(() => props.doctype.label)
</script>

<template>
  <div class="form-wrapper">
    <h1>{{ title }}</h1>
    <!-- ... -->
  </div>
</template>
```

---

## Ключові composables

### useDocType — метадані DocType

```typescript
import { useDocType } from '@/core/composables/useDocType'

const { doctype, fields, isLoading } = useDocType('Customer')
```

### useDocument — один документ

```typescript
import { useDocument } from '@/core/composables/useDocument'

const { doc, save, isLoading, error } = useDocument('Customer', docId)

// Зберегти зміни
await save({ status: 'Активний' })
```

### useList — список документів

```typescript
import { useList } from '@/core/composables/useList'

const { items, total, isLoading, refetch } = useList('Customer', {
  filters: { status: 'Активний' },
  fields: ['id', 'full_name', 'email', 'status'],
  orderBy: 'created_at',
  order: 'desc',
  perPage: 20,
})
```

---

## API клієнт

```typescript
import { api } from '@/core/api'

// GET список
const { data } = await api.get('/docs/Customer', {
  params: { 'filter[status]': 'Активний', per_page: 20 }
})

// POST створити
const { data: newDoc } = await api.post('/docs/Customer', {
  full_name: 'Іван Іванов',
  email: 'ivan@example.com',
})

// PUT оновити
await api.put(`/docs/Customer/${id}`, { status: 'Закритий' })

// DELETE видалити
await api.delete(`/docs/Customer/${id}`)
```

---

## grunt-ui компоненти

Власна UI-бібліотека. Імпортуй з `@/components/ui/`:

```vue
<script setup lang="ts">
import Button from '@/components/ui/Button.vue'
import Input from '@/components/ui/Input.vue'
import Modal from '@/components/ui/Modal.vue'
import Badge from '@/components/ui/Badge.vue'
</script>

<template>
  <Modal :open="isOpen" title="Новий клієнт" @close="isOpen = false">
    <Input v-model="form.full_name" label="Повне ім'я" required />
    <Input v-model="form.email" label="Email" type="email" />
    <template #footer>
      <Button variant="primary" :loading="isSaving" @click="save">
        Зберегти
      </Button>
    </template>
  </Modal>
</template>
```

---

## Pinia stores

```typescript
// stores/useOrderStore.ts
import { defineStore } from 'pinia'
import { ref } from 'vue'
import { api } from '@/core/api'

export const useOrderStore = defineStore('order', () => {
  const orders = ref<Order[]>([])
  const isLoading = ref(false)

  async function fetchOrders() {
    isLoading.value = true
    try {
      const { data } = await api.get('/docs/Order')
      orders.value = data.data
    } finally {
      isLoading.value = false
    }
  }

  return { orders, isLoading, fetchOrders }
})
```

---

## Vue Router

```typescript
// Навігація з коду
import { useRouter } from 'vue-router'

const router = useRouter()

// Перейти до документа
router.push({ name: 'desk-form', params: { doctype: 'Customer', id: customer.id } })

// Перейти до списку
router.push({ name: 'desk-list', params: { doctype: 'Customer' } })
```

---

## TypeScript типи

```typescript
import type { DocType, DocField, Document } from '@/types'

// DocType метадані
interface DocType {
  name: string
  label: string
  module: string
  fields: DocField[]
  permissions: Permission[]
}

// Поле DocType
interface DocField {
  fieldname: string
  label: string
  fieldtype: string
  required?: boolean
  options?: string
}
```

---

## Правила та анти-паттерни

✅ **Завжди:**
- `<script setup lang="ts">` — ніяких Options API
- Строгі TypeScript типи — ніяких `any` без коментаря
- `defineProps<{...}>()` з явними типами
- `defineEmits<{...}>()` для всіх подій
- **Пріоритет DocType**: Нові сторінки та складні UI-компоненти повинні описуватися та налаштовуватися через DocTypes (метадані), а не бути hard-coded.

❌ **Заборонено:**
- Options API (`export default { data() {...} }`)
- `any` тип без `// eslint-disable-next-line` коментаря
- `v-html` без sanitize (XSS вразливість)
- Прямі запити до API без `useQuery` або `api` клієнта
- Глобальні змінні поза Pinia stores
