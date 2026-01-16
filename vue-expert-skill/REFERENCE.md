# Vue Expert - Technical Reference

## Development Workflow

### Project Setup
- Initializes Vue 3 project with Vite or Vue CLI
- Configures TypeScript with strict type checking
- Sets up Pinia for state management
- Implements Vue Router with proper route guards
- Configures Vitest for testing with Vue Test Utils

### Component Development
- Uses Single File Components with Composition API
- Implements proper TypeScript interfaces for props and emits
- Creates reusable composables for shared logic
- Uses Vue DevTools for debugging and performance analysis
- Implements component testing with Vue Test Utils

### State Management
- Designs Pinia stores with proper TypeScript typing
- Implements proper data flow with actions and getters
- Uses store composables for reusable store logic
- Implements persistence strategies with plugins
- Monitors state changes with Vue DevTools

## Problem Areas Addressed

- Complex state management in single-page applications
- Performance optimization in reactive applications
- Component communication and data flow challenges
- SEO optimization for client-side rendered applications
- Integration with third-party libraries and APIs

## Reactivity System Deep Dive

### ref vs reactive

**ref** - Best for primitive values and when you need `.value` access:
```typescript
const count = ref(0)
const name = ref('John')
count.value++ // Access with .value
```

**reactive** - Best for complex objects:
```typescript
const user = reactive({
  name: 'John',
  age: 30,
  address: { city: 'NYC' }
})
user.name = 'Jane' // Direct property access
```

### toRefs for Destructuring

When destructuring reactive objects, use `toRefs` to maintain reactivity:
```typescript
const user = reactive({ name: 'John', age: 30 })

// BAD - loses reactivity
const { name, age } = user

// GOOD - maintains reactivity
const { name, age } = toRefs(user)
console.log(name.value) // Still reactive!
```

### Computed Properties

Use computed for derived state that depends on reactive values:
```typescript
const firstName = ref('John')
const lastName = ref('Doe')

const fullName = computed(() => `${firstName.value} ${lastName.value}`)
```

### Watch vs WatchEffect

**watch** - Explicit dependencies, access to old/new values:
```typescript
watch(count, (newValue, oldValue) => {
  console.log(`Count changed from ${oldValue} to ${newValue}`)
}, { immediate: true, deep: true })
```

**watchEffect** - Auto-tracks dependencies:
```typescript
watchEffect(() => {
  console.log(`Count is now: ${count.value}`)
  // Automatically re-runs when count changes
})
```

## Pinia Store Patterns

### Setup Syntax Store (Recommended)
```typescript
export const useUserStore = defineStore('user', () => {
  // State
  const user = ref<User | null>(null)
  const loading = ref(false)
  
  // Getters
  const isAuthenticated = computed(() => !!user.value)
  
  // Actions
  const login = async (credentials: LoginCredentials) => {
    loading.value = true
    try {
      user.value = await userService.login(credentials)
    } finally {
      loading.value = false
    }
  }
  
  return { user, loading, isAuthenticated, login }
})
```

### Store Composition
```typescript
// Composing stores together
export const useCartStore = defineStore('cart', () => {
  const userStore = useUserStore()
  
  const canCheckout = computed(() => 
    userStore.isAuthenticated && items.value.length > 0
  )
})
```

## Vue Router Patterns

### Route Guards
```typescript
router.beforeEach(async (to, from) => {
  const userStore = useUserStore()
  
  if (to.meta.requiresAuth && !userStore.isAuthenticated) {
    return { name: 'login', query: { redirect: to.fullPath } }
  }
})
```

### Dynamic Routes
```typescript
const routes = [
  { path: '/users/:id', component: UserProfile },
  { path: '/products/:category/:id?', component: ProductPage }
]
```

## Nuxt.js Specifics

### File-based Routing
```
pages/
├── index.vue          → /
├── about.vue          → /about
├── users/
│   ├── index.vue      → /users
│   └── [id].vue       → /users/:id
└── [...slug].vue      → catch-all route
```

### useFetch and useAsyncData
```typescript
// Auto-cached, SSR-friendly data fetching
const { data, pending, error, refresh } = await useFetch('/api/users')

// With options
const { data } = await useFetch('/api/users', {
  key: 'users',
  lazy: true,
  server: true,
  transform: (data) => data.users
})
```

### Server API Routes
```typescript
// server/api/users.get.ts
export default defineEventHandler(async (event) => {
  const query = getQuery(event)
  return await db.users.findMany({ take: query.limit })
})
```

## Performance Optimization Techniques

### v-memo for List Optimization
```vue
<template>
  <div v-for="item in items" :key="item.id" v-memo="[item.selected]">
    <!-- Only re-renders when item.selected changes -->
  </div>
</template>
```

### defineAsyncComponent for Code Splitting
```typescript
const HeavyComponent = defineAsyncComponent(() =>
  import('./HeavyComponent.vue')
)
```

### Virtual Scrolling
```vue
<template>
  <RecycleScroller
    :items="items"
    :item-size="50"
    v-slot="{ item }"
  >
    <ItemComponent :item="item" />
  </RecycleScroller>
</template>
```

## TypeScript Integration

### Typed Props and Emits
```typescript
interface Props {
  user: User
  isEditable?: boolean
}

interface Emits {
  (e: 'update:user', user: User): void
  (e: 'delete', id: string): void
}

const props = withDefaults(defineProps<Props>(), {
  isEditable: false
})
const emit = defineEmits<Emits>()
```

### Typed Refs
```typescript
const inputRef = ref<HTMLInputElement | null>(null)
const componentRef = ref<InstanceType<typeof MyComponent> | null>(null)
```

### Generic Composables
```typescript
function useAsyncData<T>(asyncFn: () => Promise<T>) {
  const data = ref<T | null>(null) as Ref<T | null>
  const error = ref<Error | null>(null)
  const pending = ref(false)
  
  // ... implementation
  
  return { data, error, pending }
}
```
