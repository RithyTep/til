# Zustand - Lightweight State Management

<div align="center">

![Zustand](https://img.shields.io/badge/Zustand-433E38?style=for-the-badge&logo=react&logoColor=white)
![State](https://img.shields.io/badge/State-Management-blue?style=for-the-badge)
![React](https://img.shields.io/badge/React-61DAFB?style=for-the-badge&logo=react&logoColor=black)

*Small, fast, scalable state management with a minimal API.*

</div>

## Why Zustand?

```
Redux                          Zustand
──────────────────────────     ──────────────────────────
Boilerplate heavy              Minimal boilerplate
Actions, reducers, types       Just functions
Provider required              No provider needed
~8kb                           ~1kb
Complex setup                  5 minutes to learn
```

## Installation

```bash
npm install zustand
```

## Basic Store

```typescript
import { create } from 'zustand'

interface CounterStore {
  count: number
  increment: () => void
  decrement: () => void
  reset: () => void
}

const useCounterStore = create<CounterStore>((set) => ({
  count: 0,
  increment: () => set((state) => ({ count: state.count + 1 })),
  decrement: () => set((state) => ({ count: state.count - 1 })),
  reset: () => set({ count: 0 }),
}))

// Usage in component
function Counter() {
  const { count, increment, decrement } = useCounterStore()

  return (
    <div>
      <span>{count}</span>
      <button onClick={increment}>+</button>
      <button onClick={decrement}>-</button>
    </div>
  )
}
```

## Selecting State (Performance)

```typescript
// Only re-render when count changes
function CountDisplay() {
  const count = useCounterStore((state) => state.count)
  return <span>{count}</span>
}

// Only re-render when increment changes
function IncrementButton() {
  const increment = useCounterStore((state) => state.increment)
  return <button onClick={increment}>+</button>
}

// Select multiple values (use shallow for objects)
import { useShallow } from 'zustand/react/shallow'

function Controls() {
  const { increment, decrement } = useCounterStore(
    useShallow((state) => ({
      increment: state.increment,
      decrement: state.decrement,
    }))
  )

  return (
    <>
      <button onClick={increment}>+</button>
      <button onClick={decrement}>-</button>
    </>
  )
}
```

## Async Actions

```typescript
interface TodoStore {
  todos: Todo[]
  loading: boolean
  error: string | null
  fetchTodos: () => Promise<void>
  addTodo: (title: string) => Promise<void>
}

const useTodoStore = create<TodoStore>((set, get) => ({
  todos: [],
  loading: false,
  error: null,

  fetchTodos: async () => {
    set({ loading: true, error: null })
    try {
      const response = await fetch('/api/todos')
      const todos = await response.json()
      set({ todos, loading: false })
    } catch (error) {
      set({ error: 'Failed to fetch', loading: false })
    }
  },

  addTodo: async (title) => {
    const response = await fetch('/api/todos', {
      method: 'POST',
      body: JSON.stringify({ title }),
    })
    const newTodo = await response.json()

    // Access current state with get()
    set({ todos: [...get().todos, newTodo] })
  },
}))
```

## Middleware: Persist

```typescript
import { create } from 'zustand'
import { persist, createJSONStorage } from 'zustand/middleware'

interface AuthStore {
  user: User | null
  token: string | null
  login: (user: User, token: string) => void
  logout: () => void
}

const useAuthStore = create<AuthStore>()(
  persist(
    (set) => ({
      user: null,
      token: null,
      login: (user, token) => set({ user, token }),
      logout: () => set({ user: null, token: null }),
    }),
    {
      name: 'auth-storage',
      storage: createJSONStorage(() => localStorage),
      partialize: (state) => ({ token: state.token }), // Only persist token
    }
  )
)
```

## Middleware: DevTools

```typescript
import { create } from 'zustand'
import { devtools } from 'zustand/middleware'

const useStore = create<Store>()(
  devtools(
    (set) => ({
      count: 0,
      increment: () => set(
        (state) => ({ count: state.count + 1 }),
        false,
        'increment' // Action name for DevTools
      ),
    }),
    { name: 'MyStore' }
  )
)
```

## Middleware: Immer

```typescript
import { create } from 'zustand'
import { immer } from 'zustand/middleware/immer'

interface Store {
  users: User[]
  updateUser: (id: string, data: Partial<User>) => void
  addUser: (user: User) => void
}

const useStore = create<Store>()(
  immer((set) => ({
    users: [],

    // Mutate directly with Immer
    updateUser: (id, data) =>
      set((state) => {
        const user = state.users.find((u) => u.id === id)
        if (user) {
          Object.assign(user, data)
        }
      }),

    addUser: (user) =>
      set((state) => {
        state.users.push(user) // Direct mutation OK with Immer
      }),
  }))
)
```

## Combine Multiple Middleware

```typescript
import { create } from 'zustand'
import { devtools, persist } from 'zustand/middleware'
import { immer } from 'zustand/middleware/immer'

const useStore = create<Store>()(
  devtools(
    persist(
      immer((set) => ({
        // store definition
      })),
      { name: 'store' }
    ),
    { name: 'MyStore' }
  )
)
```

## Slices Pattern

```typescript
// stores/userSlice.ts
interface UserSlice {
  user: User | null
  setUser: (user: User) => void
}

const createUserSlice = (set: SetState<Store>): UserSlice => ({
  user: null,
  setUser: (user) => set({ user }),
})

// stores/cartSlice.ts
interface CartSlice {
  items: CartItem[]
  addItem: (item: CartItem) => void
  clearCart: () => void
}

const createCartSlice = (set: SetState<Store>): CartSlice => ({
  items: [],
  addItem: (item) => set((state) => ({
    items: [...state.items, item],
  })),
  clearCart: () => set({ items: [] }),
})

// stores/index.ts
type Store = UserSlice & CartSlice

const useStore = create<Store>()((...a) => ({
  ...createUserSlice(...a),
  ...createCartSlice(...a),
}))
```

## Outside React

```typescript
// Access store outside components
const { count, increment } = useCounterStore.getState()

// Subscribe to changes
const unsubscribe = useCounterStore.subscribe(
  (state) => console.log('Count changed:', state.count)
)

// Subscribe to specific slice
const unsubscribe2 = useCounterStore.subscribe(
  (state) => state.count,
  (count, prevCount) => console.log('Count:', prevCount, '->', count)
)
```

## TypeScript Best Practices

```typescript
// Define actions separately for better typing
interface State {
  count: number
}

interface Actions {
  increment: () => void
  decrement: () => void
  setCount: (count: number) => void
}

type Store = State & Actions

const useStore = create<Store>((set) => ({
  count: 0,
  increment: () => set((s) => ({ count: s.count + 1 })),
  decrement: () => set((s) => ({ count: s.count - 1 })),
  setCount: (count) => set({ count }),
}))
```

## With React Query

```typescript
// Zustand for UI state, React Query for server state
const useUIStore = create<UIStore>((set) => ({
  sidebarOpen: false,
  toggleSidebar: () => set((s) => ({ sidebarOpen: !s.sidebarOpen })),
}))

function App() {
  const sidebarOpen = useUIStore((s) => s.sidebarOpen)
  const { data: todos } = useQuery({ queryKey: ['todos'], queryFn: fetchTodos })

  return (
    <Layout sidebarOpen={sidebarOpen}>
      <TodoList todos={todos} />
    </Layout>
  )
}
```

---

*Learned: December 20, 2025*
*Tags: Zustand, State Management, React, TypeScript*
