# Vitest - Blazing Fast Unit Testing

<div align="center">

![Vitest](https://img.shields.io/badge/Vitest-6E9F18?style=for-the-badge&logo=vitest&logoColor=white)
![Testing](https://img.shields.io/badge/Testing-Unit-blue?style=for-the-badge)
![Vite](https://img.shields.io/badge/Vite-646CFF?style=for-the-badge&logo=vite&logoColor=white)

*Next-gen testing framework powered by Vite - Jest compatible, instant HMR.*

![Vitest](https://vitest.dev/logo-shadow.svg)

</div>

## Why Vitest?

```
Jest                           Vitest
──────────────────────────     ──────────────────────────
Separate config                Reuses Vite config
Slow startup                   Instant startup
Manual TS setup                Native TypeScript
No ESM by default              Native ESM
Separate watch mode            HMR-powered watch
```

## Installation

```bash
npm install -D vitest

# With React Testing Library
npm install -D @testing-library/react @testing-library/jest-dom jsdom
```

## Configuration

```typescript
// vite.config.ts
import { defineConfig } from 'vitest/config'
import react from '@vitejs/plugin-react'

export default defineConfig({
  plugins: [react()],
  test: {
    globals: true,
    environment: 'jsdom',
    setupFiles: './src/test/setup.ts',
    css: true,
    coverage: {
      provider: 'v8',
      reporter: ['text', 'json', 'html'],
    },
  },
})
```

```typescript
// src/test/setup.ts
import '@testing-library/jest-dom'
```

```json
// tsconfig.json - add types
{
  "compilerOptions": {
    "types": ["vitest/globals"]
  }
}
```

## Basic Tests

```typescript
// math.test.ts
import { describe, it, expect, vi } from 'vitest'
import { add, multiply, divide } from './math'

describe('math operations', () => {
  it('adds two numbers', () => {
    expect(add(2, 3)).toBe(5)
  })

  it('multiplies two numbers', () => {
    expect(multiply(4, 5)).toBe(20)
  })

  it('throws on division by zero', () => {
    expect(() => divide(10, 0)).toThrow('Division by zero')
  })
})
```

## React Component Testing

```typescript
// Button.test.tsx
import { render, screen, fireEvent } from '@testing-library/react'
import { describe, it, expect, vi } from 'vitest'
import { Button } from './Button'

describe('Button', () => {
  it('renders with text', () => {
    render(<Button>Click me</Button>)
    expect(screen.getByRole('button')).toHaveTextContent('Click me')
  })

  it('calls onClick when clicked', () => {
    const handleClick = vi.fn()
    render(<Button onClick={handleClick}>Click</Button>)

    fireEvent.click(screen.getByRole('button'))

    expect(handleClick).toHaveBeenCalledOnce()
  })

  it('is disabled when loading', () => {
    render(<Button loading>Submit</Button>)

    expect(screen.getByRole('button')).toBeDisabled()
    expect(screen.getByText('Loading...')).toBeInTheDocument()
  })
})
```

## Mocking

```typescript
import { vi, describe, it, expect, beforeEach } from 'vitest'

// Mock module
vi.mock('./api', () => ({
  fetchUser: vi.fn(),
}))

import { fetchUser } from './api'
import { getUser } from './userService'

describe('userService', () => {
  beforeEach(() => {
    vi.clearAllMocks()
  })

  it('fetches and transforms user', async () => {
    vi.mocked(fetchUser).mockResolvedValue({
      id: 1,
      name: 'John',
    })

    const user = await getUser(1)

    expect(fetchUser).toHaveBeenCalledWith(1)
    expect(user.displayName).toBe('John')
  })

  it('handles errors', async () => {
    vi.mocked(fetchUser).mockRejectedValue(new Error('Not found'))

    await expect(getUser(1)).rejects.toThrow('Not found')
  })
})
```

## Spying

```typescript
import { vi, describe, it, expect } from 'vitest'

describe('spying', () => {
  it('spies on object methods', () => {
    const cart = {
      items: [],
      addItem(item: string) {
        this.items.push(item)
      },
    }

    const spy = vi.spyOn(cart, 'addItem')

    cart.addItem('apple')

    expect(spy).toHaveBeenCalledWith('apple')
    expect(cart.items).toContain('apple')
  })

  it('mocks implementation', () => {
    const spy = vi.spyOn(Math, 'random').mockReturnValue(0.5)

    expect(Math.random()).toBe(0.5)

    spy.mockRestore()
  })
})
```

## Timers

```typescript
import { vi, describe, it, expect, beforeEach, afterEach } from 'vitest'

describe('timers', () => {
  beforeEach(() => {
    vi.useFakeTimers()
  })

  afterEach(() => {
    vi.useRealTimers()
  })

  it('handles setTimeout', () => {
    const callback = vi.fn()

    setTimeout(callback, 1000)

    expect(callback).not.toHaveBeenCalled()

    vi.advanceTimersByTime(1000)

    expect(callback).toHaveBeenCalledOnce()
  })

  it('handles setInterval', () => {
    const callback = vi.fn()

    setInterval(callback, 100)

    vi.advanceTimersByTime(350)

    expect(callback).toHaveBeenCalledTimes(3)
  })

  it('runs all pending timers', async () => {
    const callback = vi.fn()

    setTimeout(callback, 5000)
    setTimeout(callback, 10000)

    vi.runAllTimers()

    expect(callback).toHaveBeenCalledTimes(2)
  })
})
```

## Async Testing

```typescript
import { describe, it, expect } from 'vitest'

describe('async operations', () => {
  it('awaits promises', async () => {
    const result = await fetchData()
    expect(result).toBe('data')
  })

  it('resolves correctly', async () => {
    await expect(fetchData()).resolves.toBe('data')
  })

  it('rejects correctly', async () => {
    await expect(failingFetch()).rejects.toThrow('Error')
  })
})
```

## Snapshot Testing

```typescript
import { describe, it, expect } from 'vitest'
import { render } from '@testing-library/react'

describe('snapshots', () => {
  it('matches snapshot', () => {
    const { container } = render(<UserCard user={mockUser} />)
    expect(container).toMatchSnapshot()
  })

  it('matches inline snapshot', () => {
    const user = { name: 'John', age: 30 }
    expect(user).toMatchInlineSnapshot(`
      {
        "age": 30,
        "name": "John",
      }
    `)
  })
})
```

## Test Hooks with Testing Library

```typescript
import { renderHook, act, waitFor } from '@testing-library/react'
import { describe, it, expect, vi } from 'vitest'
import { useCounter } from './useCounter'

describe('useCounter', () => {
  it('increments counter', () => {
    const { result } = renderHook(() => useCounter())

    expect(result.current.count).toBe(0)

    act(() => {
      result.current.increment()
    })

    expect(result.current.count).toBe(1)
  })
})

// Async hook
import { useFetch } from './useFetch'

describe('useFetch', () => {
  it('fetches data', async () => {
    const { result } = renderHook(() => useFetch('/api/users'))

    expect(result.current.loading).toBe(true)

    await waitFor(() => {
      expect(result.current.loading).toBe(false)
    })

    expect(result.current.data).toHaveLength(5)
  })
})
```

## In-Source Testing

```typescript
// math.ts
export function add(a: number, b: number): number {
  return a + b
}

// In-source tests (removed in production)
if (import.meta.vitest) {
  const { describe, it, expect } = import.meta.vitest

  describe('add', () => {
    it('works', () => {
      expect(add(1, 2)).toBe(3)
    })
  })
}
```

## CLI Commands

```bash
# Run all tests
npx vitest

# Run once (no watch)
npx vitest run

# Run specific file
npx vitest math.test.ts

# Run with coverage
npx vitest --coverage

# UI mode
npx vitest --ui

# Filter tests
npx vitest -t "adds"

# Update snapshots
npx vitest -u
```

## package.json Scripts

```json
{
  "scripts": {
    "test": "vitest",
    "test:run": "vitest run",
    "test:ui": "vitest --ui",
    "test:coverage": "vitest run --coverage"
  }
}
```

---

*Learned: December 20, 2025*
*Tags: Vitest, Testing, Unit Tests, Vite, Jest*
