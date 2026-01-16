# Vue Expert - Code Examples & Patterns

## Advanced Composables with TypeScript

```typescript
// composables/useAsyncData.ts - Reusable async data composable
import { ref, computed, readonly, type Ref } from 'vue'

interface UseAsyncDataOptions<T, E = Error> {
  immediate?: boolean
  resetOnExecute?: boolean
  onError?: (error: E) => void
  onSuccess?: (data: T) => void
}

interface UseAsyncDataReturn<T, E = Error> {
  data: Ref<T | null>
  error: Ref<E | null>
  pending: Ref<boolean>
  execute: () => Promise<T>
  refresh: () => Promise<T>
  reset: () => void
}

export function useAsyncData<T, E = Error>(
  asyncFn: () => Promise<T>,
  options: UseAsyncDataOptions<T, E> = {}
): UseAsyncDataReturn<T, E> {
  const {
    immediate = true,
    resetOnExecute = true,
    onError,
    onSuccess
  } = options

  const data = ref<T | null>(null) as Ref<T | null>
  const error = ref<E | null>(null) as Ref<E | null>
  const pending = ref(false)

  const execute = async (): Promise<T> => {
    if (resetOnExecute) {
      error.value = null
      data.value = null
    }

    pending.value = true

    try {
      const result = await asyncFn()
      data.value = result
      onSuccess?.(result)
      return result
    } catch (e) {
      error.value = e as E
      onError?.(e as E)
      throw e
    } finally {
      pending.value = false
    }
  }

  const refresh = () => execute()
  const reset = () => {
    data.value = null
    error.value = null
    pending.value = false
  }

  if (immediate) {
    execute()
  }

  return {
    data: readonly(data),
    error: readonly(error),
    pending: readonly(pending),
    execute,
    refresh,
    reset
  }
}

// Usage example
const { data: user, error, pending, refresh } = useAsyncData(
  () => userService.fetchUserProfile(),
  {
    immediate: true,
    onSuccess: (userData) => {
      console.log('User data loaded:', userData)
    },
    onError: (err) => {
      console.error('Failed to load user:', err)
    }
  }
)
```

## Pinia Store with TypeScript

```typescript
// stores/user.ts - Typed Pinia store
import { defineStore } from 'pinia'
import { ref, computed } from 'vue'
import type { User, LoginCredentials, RegisterData } from '@/types/user'
import { userService } from '@/services/user'

export const useUserStore = defineStore('user', () => {
  // State
  const user = ref<User | null>(null)
  const token = ref<string | null>(null)
  const loading = ref(false)
  const error = ref<string | null>(null)

  // Getters
  const isAuthenticated = computed(() => !!token.value && !!user.value)
  const userRole = computed(() => user.value?.role ?? 'guest')
  const userName = computed(() => user.value?.name ?? 'Guest')
  const userPermissions = computed(() => {
    if (!user.value?.permissions) return []
    return Array.isArray(user.value.permissions) 
      ? user.value.permissions 
      : [user.value.permissions]
  })

  // Actions
  const login = async (credentials: LoginCredentials) => {
    loading.value = true
    error.value = null

    try {
      const response = await userService.login(credentials)
      user.value = response.user
      token.value = response.token
      
      localStorage.setItem('auth_token', response.token)
      
      return response
    } catch (err) {
      error.value = err instanceof Error ? err.message : 'Login failed'
      throw err
    } finally {
      loading.value = false
    }
  }

  const logout = async () => {
    try {
      await userService.logout()
    } catch (err) {
      console.error('Logout error:', err)
    } finally {
      user.value = null
      token.value = null
      localStorage.removeItem('auth_token')
    }
  }

  const fetchProfile = async () => {
    if (!token.value) return

    loading.value = true
    error.value = null

    try {
      const profile = await userService.getProfile()
      user.value = profile
      return profile
    } catch (err) {
      error.value = err instanceof Error ? err.message : 'Failed to fetch profile'
      throw err
    } finally {
      loading.value = false
    }
  }

  const hasPermission = (permission: string) => {
    return userPermissions.value.includes(permission)
  }

  const hasRole = (role: string) => {
    return userRole.value === role
  }

  // Initialize store from localStorage
  const initialize = () => {
    const storedToken = localStorage.getItem('auth_token')
    if (storedToken) {
      token.value = storedToken
      fetchProfile()
    }
  }

  return {
    // State
    user: readonly(user),
    token: readonly(token),
    loading: readonly(loading),
    error: readonly(error),
    
    // Getters
    isAuthenticated,
    userRole,
    userName,
    userPermissions,
    
    // Actions
    login,
    logout,
    fetchProfile,
    hasPermission,
    hasRole,
    initialize
  }
})
```

## Advanced DataTable Component

```vue
<!-- components/DataTable.vue -->
<template>
  <div class="data-table">
    <!-- Search and filters -->
    <div class="table-controls mb-4">
      <div class="flex gap-4 mb-4">
        <input
          v-model="searchQuery"
          type="text"
          placeholder="Search..."
          class="px-4 py-2 border rounded-lg"
          @input="debouncedSearch"
        />
        
        <select
          v-model="selectedPageSize"
          class="px-4 py-2 border rounded-lg"
          @change="changePageSize"
        >
          <option v-for="size in pageSizeOptions" :key="size" :value="size">
            {{ size }} per page
          </option>
        </select>
      </div>
      
      <!-- Column visibility toggle -->
      <div class="flex gap-2 flex-wrap">
        <button
          v-for="column in columns"
          :key="column.key"
          @click="toggleColumnVisibility(column.key)"
          :class="[
            'px-3 py-1 rounded text-sm',
            visibleColumns.includes(column.key)
              ? 'bg-blue-500 text-white'
              : 'bg-gray-200 text-gray-700'
          ]"
        >
          {{ column.label }}
        </button>
      </div>
    </div>

    <!-- Loading state -->
    <div v-if="pending" class="text-center py-8">
      <div class="inline-block animate-spin rounded-full h-8 w-8 border-b-2 border-blue-500"></div>
      <p class="mt-2">Loading...</p>
    </div>

    <!-- Error state -->
    <div v-else-if="error" class="text-center py-8 text-red-500">
      <p>{{ error }}</p>
      <button @click="refresh" class="mt-2 px-4 py-2 bg-blue-500 text-white rounded">
        Retry
      </button>
    </div>

    <!-- Table -->
    <div v-else class="overflow-x-auto">
      <table class="min-w-full bg-white border border-gray-200">
        <thead>
          <tr>
            <th
              v-for="column in visibleColumnsData"
              :key="column.key"
              @click="sortBy(column.key)"
              :class="[
                'px-6 py-3 border-b text-left text-xs font-medium text-gray-500 uppercase cursor-pointer hover:bg-gray-50',
                sortColumn === column.key && 'bg-gray-100'
              ]"
            >
              <div class="flex items-center">
                {{ column.label }}
                <span v-if="sortColumn === column.key" class="ml-2">
                  {{ sortDirection === 'asc' ? '↑' : '↓' }}
                </span>
              </div>
            </th>
          </tr>
        </thead>
        
        <tbody>
          <tr v-for="item in paginatedData" :key="getItemKey(item)" class="hover:bg-gray-50">
            <td
              v-for="column in visibleColumnsData"
              :key="column.key"
              class="px-6 py-4 whitespace-no-wrap border-b border-gray-200"
            >
              <slot :name="`cell-${column.key}`" :item="item" :value="getNestedValue(item, column.key)">
                {{ formatCellValue(getNestedValue(item, column.key), column) }}
              </slot>
            </td>
          </tr>
        </tbody>
      </table>
    </div>

    <!-- Pagination -->
    <div class="flex items-center justify-between mt-4">
      <div class="text-sm text-gray-700">
        Showing {{ (currentPage - 1) * pageSize + 1 }} to {{ Math.min(currentPage * pageSize, filteredData.length) }} 
        of {{ filteredData.length }} results
      </div>
      
      <div class="flex gap-2">
        <button
          @click="goToPage(currentPage - 1)"
          :disabled="currentPage === 1"
          class="px-3 py-1 border rounded disabled:opacity-50"
        >
          Previous
        </button>
        
        <span class="px-3 py-1">
          Page {{ currentPage }} of {{ totalPages }}
        </span>
        
        <button
          @click="goToPage(currentPage + 1)"
          :disabled="currentPage === totalPages"
          class="px-3 py-1 border rounded disabled:opacity-50"
        >
          Next
        </button>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, computed, watch, onMounted } from 'vue'
import { useDebounceFn } from '@vueuse/core'
import type { TableColumn, DataTableItem } from '@/types/table'

interface Props {
  columns: TableColumn[]
  data: DataTableItem[]
  loading?: boolean
  error?: string
  pageSize?: number
  itemKey?: string
}

const props = withDefaults(defineProps<Props>(), {
  pageSize: 10,
  itemKey: 'id'
})

const emit = defineEmits<{
  refresh: []
  sort: [{ column: string; direction: 'asc' | 'desc' }]
}>()

// Reactive state
const searchQuery = ref('')
const selectedPageSize = ref(props.pageSize)
const currentPage = ref(1)
const sortColumn = ref<string>('')
const sortDirection = ref<'asc' | 'desc'>('asc')
const visibleColumns = ref(props.columns.map(col => col.key))

// Computed properties
const pageSizeOptions = computed(() => [5, 10, 20, 50, 100])

const visibleColumnsData = computed(() => 
  props.columns.filter(col => visibleColumns.value.includes(col.key))
)

const filteredData = computed(() => {
  if (!searchQuery.value) return props.data
  
  const query = searchQuery.value.toLowerCase()
  return props.data.filter(item => 
    props.columns.some(column => {
      const value = getNestedValue(item, column.key)
      return String(value).toLowerCase().includes(query)
    })
  )
})

const sortedData = computed(() => {
  if (!sortColumn.value) return filteredData.value
  
  return [...filteredData.value].sort((a, b) => {
    const aValue = getNestedValue(a, sortColumn.value)
    const bValue = getNestedValue(b, sortColumn.value)
    
    let comparison = 0
    if (aValue < bValue) comparison = -1
    if (aValue > bValue) comparison = 1
    
    return sortDirection.value === 'asc' ? comparison : -comparison
  })
})

const totalPages = computed(() => 
  Math.ceil(sortedData.value.length / selectedPageSize.value)
)

const paginatedData = computed(() => {
  const start = (currentPage.value - 1) * selectedPageSize.value
  const end = start + selectedPageSize.value
  return sortedData.value.slice(start, end)
})

// Methods
const debouncedSearch = useDebounceFn(() => {
  currentPage.value = 1
}, 300)

const sortBy = (column: string) => {
  if (sortColumn.value === column) {
    sortDirection.value = sortDirection.value === 'asc' ? 'desc' : 'asc'
  } else {
    sortColumn.value = column
    sortDirection.value = 'asc'
  }
  
  emit('sort', { column, direction: sortDirection.value })
}

const getNestedValue = (obj: any, path: string) => {
  return path.split('.').reduce((current, key) => current?.[key], obj)
}
</script>
```

## Anti-Patterns & Fixes

### Anti-Pattern 1: Mixing ref and reactive Inconsistently

**BAD:**
```typescript
// ❌ BAD: Inconsistent reactivity patterns
import { ref, reactive } from 'vue';

const user = reactive({
  name: 'John',
  age: 30
});

const userName = ref('John'); // ❌ Duplicate primitive
const userAge = ref(30);      // ❌ Mixing patterns unnecessarily

// ❌ Loses reactivity when destructuring
const { name, age } = user;
console.log(name); // Not reactive anymore!
```

**GOOD:**
```typescript
// ✅ GOOD: Consistent ref usage for primitives
import { ref, computed } from 'vue';

const userName = ref('John');
const userAge = ref(30);

// ✅ Or: Use reactive for entire object
import { reactive, toRefs } from 'vue';

const user = reactive({
  name: 'John',
  age: 30
});

// ✅ Maintain reactivity when destructuring
const { name, age } = toRefs(user);
console.log(name.value); // Still reactive!

// ✅ Or: Use computed for derived values
const userDisplayName = computed(() => `${user.name} (${user.age})`);
```

**Impact:** Consistent reactivity patterns, fewer bugs, easier maintenance.

### Anti-Pattern 2: Direct Mutation of Props

**BAD:**
```vue
<!-- ❌ BAD: Mutating props directly -->
<script setup lang="ts">
interface Props {
  user: { name: string; age: number };
}

const props = defineProps<Props>();

// ❌ Direct prop mutation (Vue will warn!)
function updateName(newName: string) {
  props.user.name = newName; // ❌ ERROR: Props are readonly!
}
</script>
```

**GOOD:**
```vue
<!-- ✅ GOOD: Emit events for parent updates -->
<script setup lang="ts">
interface Props {
  user: { name: string; age: number };
}

interface Emits {
  (e: 'update:user', user: { name: string; age: number }): void;
}

const props = defineProps<Props>();
const emit = defineEmits<Emits>();

// ✅ Emit event to parent
function updateName(newName: string) {
  emit('update:user', { ...props.user, name: newName });
}

// ✅ Or use v-model pattern with computed
const userLocal = computed({
  get: () => props.user,
  set: (value) => emit('update:user', value)
});
</script>

<!-- Parent component -->
<template>
  <ChildComponent v-model:user="user" />
</template>
```

**Impact:** Proper data flow, no console warnings, predictable state management.
