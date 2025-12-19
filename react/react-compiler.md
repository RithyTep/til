# React Compiler - Automatic Optimization

<div align="center">

![React](https://img.shields.io/badge/React-61DAFB?style=for-the-badge&logo=react&logoColor=black)
![Compiler](https://img.shields.io/badge/React_Compiler-Automatic-green?style=for-the-badge)
![Performance](https://img.shields.io/badge/Performance-No_Memo-blue?style=for-the-badge)

*Automatic memoization - no more useMemo, useCallback, or React.memo.*

</div>

## What React Compiler Does

```
Before (Manual)                After (Compiler)
──────────────────────────     ──────────────────────────
useMemo everywhere             Automatic memoization
useCallback for handlers       Compiler handles it
React.memo for components      Not needed
Easy to forget, bugs           Always optimized
Complex dependency arrays      No dependency arrays
```

## Installation

```bash
npm install -D babel-plugin-react-compiler

# For Next.js
npm install -D babel-plugin-react-compiler
```

## Next.js Configuration

```typescript
// next.config.ts
import type { NextConfig } from 'next'

const nextConfig: NextConfig = {
  experimental: {
    reactCompiler: true,
  },
}

export default nextConfig
```

## Babel Configuration (Other Frameworks)

```javascript
// babel.config.js
module.exports = {
  plugins: [
    ['babel-plugin-react-compiler', {
      // Options
    }],
  ],
}
```

## Vite Configuration

```typescript
// vite.config.ts
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'

export default defineConfig({
  plugins: [
    react({
      babel: {
        plugins: ['babel-plugin-react-compiler'],
      },
    }),
  ],
})
```

## Before: Manual Optimization

```typescript
// Without compiler - lots of manual work
function TodoList({ todos, filter }) {
  // Manual memoization of filtered list
  const filteredTodos = useMemo(
    () => todos.filter(todo => todo.status === filter),
    [todos, filter]
  )

  // Manual callback memoization
  const handleToggle = useCallback(
    (id: string) => {
      // toggle logic
    },
    []
  )

  // Manual callback with dependency
  const handleDelete = useCallback(
    (id: string) => {
      deleteTodo(id, filter)
    },
    [filter]
  )

  return (
    <ul>
      {filteredTodos.map(todo => (
        <MemoizedTodoItem
          key={todo.id}
          todo={todo}
          onToggle={handleToggle}
          onDelete={handleDelete}
        />
      ))}
    </ul>
  )
}

// Manual React.memo
const MemoizedTodoItem = React.memo(function TodoItem({
  todo,
  onToggle,
  onDelete
}) {
  return (
    <li>
      <span onClick={() => onToggle(todo.id)}>{todo.text}</span>
      <button onClick={() => onDelete(todo.id)}>Delete</button>
    </li>
  )
})
```

## After: With React Compiler

```typescript
// With compiler - write naturally, compiler optimizes
function TodoList({ todos, filter }) {
  // Compiler automatically memoizes this
  const filteredTodos = todos.filter(todo => todo.status === filter)

  // Compiler handles function identity
  const handleToggle = (id: string) => {
    // toggle logic
  }

  const handleDelete = (id: string) => {
    deleteTodo(id, filter)
  }

  return (
    <ul>
      {filteredTodos.map(todo => (
        <TodoItem
          key={todo.id}
          todo={todo}
          onToggle={handleToggle}
          onDelete={handleDelete}
        />
      ))}
    </ul>
  )
}

// No React.memo needed
function TodoItem({ todo, onToggle, onDelete }) {
  return (
    <li>
      <span onClick={() => onToggle(todo.id)}>{todo.text}</span>
      <button onClick={() => onDelete(todo.id)}>Delete</button>
    </li>
  )
}
```

## How It Works

```typescript
// Your code
function Component({ items }) {
  const doubled = items.map(x => x * 2)
  return <List data={doubled} />
}

// Compiler transforms to (conceptually)
function Component({ items }) {
  const doubled = useMemo(
    () => items.map(x => x * 2),
    [items]
  )
  return <List data={doubled} />
}
```

## Rules of React

The compiler enforces and relies on React's rules:

```typescript
// Components must be pure
function Component({ count }) {
  // Don't mutate props
  count.value = 5

  // Don't read/write external mutable values
  globalCounter++

  // Don't call hooks conditionally
  if (count > 0) {
    const [state] = useState(0) // Error!
  }
}

// Hooks must follow rules
function useCustomHook(value) {
  // Dependencies must be correct
  useEffect(() => {
    console.log(value) // Compiler tracks this
  }, []) // Compiler may fix or warn
}
```

## ESLint Plugin

```bash
npm install -D eslint-plugin-react-compiler
```

```javascript
// eslint.config.js
import reactCompiler from 'eslint-plugin-react-compiler'

export default [
  {
    plugins: {
      'react-compiler': reactCompiler,
    },
    rules: {
      'react-compiler/react-compiler': 'error',
    },
  },
]
```

## Opt-Out Specific Components

```typescript
// Disable compiler for specific component
function LegacyComponent() {
  'use no memo'

  // Component won't be optimized
  // Use for gradual migration
}
```

## Debugging

```typescript
// Check if compiler is working
// React DevTools shows "Memo ✓" badge on optimized components

// Or check build output
// Compiled components have _c variables
```

## What Still Needs Manual Work

```typescript
// External subscriptions still need cleanup
function Component() {
  useEffect(() => {
    const sub = eventEmitter.subscribe(handler)
    return () => sub.unsubscribe() // Still needed
  }, [])
}

// Refs still work the same
function Component() {
  const ref = useRef(null)
  // ref.current access is fine
}

// Context is still context
function Component() {
  const theme = useContext(ThemeContext)
  // Re-renders when context changes
}
```

## Migration Strategy

```typescript
// 1. Enable ESLint plugin first
// 2. Fix all violations
// 3. Enable compiler in development
// 4. Test thoroughly
// 5. Enable in production

// Gradual opt-in
const nextConfig = {
  experimental: {
    reactCompiler: {
      compilationMode: 'annotation', // Only annotated components
    },
  },
}

// Then annotate components to opt-in
function MyComponent() {
  'use memo'
  // This component is compiled
}
```

## Performance Impact

```
Before Compiler:
- Manual memo: Developer must remember
- Missed optimizations: Common
- Dependency bugs: Frequent
- Code verbosity: High

After Compiler:
- Auto memoization: 100% coverage
- Optimal updates: Guaranteed
- No dependency bugs: Compiler tracks
- Clean code: Just write React
```

---

*Learned: December 20, 2025*
*Tags: React, Compiler, Performance, Optimization, Memoization*
