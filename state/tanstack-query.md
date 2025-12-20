# TanStack Query v5 - Server State Management

<div align="center">

![TanStack Query](https://img.shields.io/badge/TanStack_Query-FF4154?style=for-the-badge&logo=react-query&logoColor=white)
![React](https://img.shields.io/badge/React-61DAFB?style=for-the-badge&logo=react&logoColor=black)
![Data](https://img.shields.io/badge/Feature-Data_Fetching-blue?style=for-the-badge)

*Powerful async state management for fetching, caching, and updating server data.*

![TanStack Query](https://upload.wikimedia.org/wikipedia/commons/a/a7/React-icon.svg)

</div>

## Why TanStack Query?

```
Manual Fetching                TanStack Query
──────────────────────────     ──────────────────────────
Loading states by hand         Automatic loading states
Manual caching                 Smart caching built-in
Refetch logic                  Auto refetch on focus
Stale data handling            Stale-while-revalidate
Duplicate requests             Automatic deduplication
```

## Installation

```bash
npm install @tanstack/react-query
```

## Setup

```typescript
// app/providers.tsx
'use client'
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'
import { ReactQueryDevtools } from '@tanstack/react-query-devtools'

const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 60 * 1000, // 1 minute
      gcTime: 5 * 60 * 1000, // 5 minutes (previously cacheTime)
    },
  },
})

export function Providers({ children }: { children: React.ReactNode }) {
  return (
    <QueryClientProvider client={queryClient}>
      {children}
      <ReactQueryDevtools initialIsOpen={false} />
    </QueryClientProvider>
  )
}
```

## Basic Query

```typescript
import { useQuery } from '@tanstack/react-query'

interface User {
  id: number
  name: string
  email: string
}

function UserProfile({ userId }: { userId: string }) {
  const {
    data: user,
    isLoading,
    isError,
    error,
    isFetching,
    refetch,
  } = useQuery({
    queryKey: ['user', userId],
    queryFn: async (): Promise<User> => {
      const res = await fetch(`/api/users/${userId}`)
      if (!res.ok) throw new Error('Failed to fetch user')
      return res.json()
    },
  })

  if (isLoading) return <Spinner />
  if (isError) return <Error message={error.message} />

  return (
    <div>
      <h1>{user.name}</h1>
      <p>{user.email}</p>
      {isFetching && <span>Updating...</span>}
    </div>
  )
}
```

## Query Keys

```typescript
// Simple key
useQuery({ queryKey: ['todos'], queryFn: fetchTodos })

// Key with parameters (auto-refetches when params change)
useQuery({ queryKey: ['todo', todoId], queryFn: () => fetchTodo(todoId) })

// Complex key with filters
useQuery({
  queryKey: ['todos', { status: 'done', page: 1 }],
  queryFn: () => fetchTodos({ status: 'done', page: 1 }),
})
```

## Mutations

```typescript
import { useMutation, useQueryClient } from '@tanstack/react-query'

function AddTodo() {
  const queryClient = useQueryClient()

  const mutation = useMutation({
    mutationFn: async (newTodo: NewTodo) => {
      const res = await fetch('/api/todos', {
        method: 'POST',
        body: JSON.stringify(newTodo),
      })
      return res.json()
    },
    onSuccess: () => {
      // Invalidate and refetch
      queryClient.invalidateQueries({ queryKey: ['todos'] })
    },
    onError: (error) => {
      console.error('Failed to add todo:', error)
    },
  })

  return (
    <form onSubmit={(e) => {
      e.preventDefault()
      mutation.mutate({ title: 'New Todo' })
    }}>
      <button disabled={mutation.isPending}>
        {mutation.isPending ? 'Adding...' : 'Add Todo'}
      </button>
    </form>
  )
}
```

## Optimistic Updates

```typescript
const mutation = useMutation({
  mutationFn: updateTodo,
  onMutate: async (newTodo) => {
    // Cancel outgoing refetches
    await queryClient.cancelQueries({ queryKey: ['todos'] })

    // Snapshot previous value
    const previousTodos = queryClient.getQueryData(['todos'])

    // Optimistically update
    queryClient.setQueryData(['todos'], (old: Todo[]) =>
      old.map((todo) =>
        todo.id === newTodo.id ? { ...todo, ...newTodo } : todo
      )
    )

    // Return context for rollback
    return { previousTodos }
  },
  onError: (err, newTodo, context) => {
    // Rollback on error
    queryClient.setQueryData(['todos'], context?.previousTodos)
  },
  onSettled: () => {
    // Refetch after error or success
    queryClient.invalidateQueries({ queryKey: ['todos'] })
  },
})
```

## Infinite Queries

```typescript
import { useInfiniteQuery } from '@tanstack/react-query'

function InfiniteList() {
  const {
    data,
    fetchNextPage,
    hasNextPage,
    isFetchingNextPage,
  } = useInfiniteQuery({
    queryKey: ['posts'],
    queryFn: async ({ pageParam }) => {
      const res = await fetch(`/api/posts?cursor=${pageParam}`)
      return res.json()
    },
    initialPageParam: 0,
    getNextPageParam: (lastPage) => lastPage.nextCursor,
  })

  return (
    <div>
      {data?.pages.map((page, i) => (
        <div key={i}>
          {page.items.map((post) => (
            <Post key={post.id} {...post} />
          ))}
        </div>
      ))}

      <button
        onClick={() => fetchNextPage()}
        disabled={!hasNextPage || isFetchingNextPage}
      >
        {isFetchingNextPage
          ? 'Loading...'
          : hasNextPage
          ? 'Load More'
          : 'Nothing more'}
      </button>
    </div>
  )
}
```

## Prefetching

```typescript
// Prefetch on hover
function TodoList({ todos }: { todos: Todo[] }) {
  const queryClient = useQueryClient()

  return (
    <ul>
      {todos.map((todo) => (
        <li
          key={todo.id}
          onMouseEnter={() => {
            queryClient.prefetchQuery({
              queryKey: ['todo', todo.id],
              queryFn: () => fetchTodo(todo.id),
            })
          }}
        >
          <Link href={`/todos/${todo.id}`}>{todo.title}</Link>
        </li>
      ))}
    </ul>
  )
}
```

## Dependent Queries

```typescript
function UserPosts({ userId }: { userId: string }) {
  // First query
  const { data: user } = useQuery({
    queryKey: ['user', userId],
    queryFn: () => fetchUser(userId),
  })

  // Dependent query - only runs when user is available
  const { data: posts } = useQuery({
    queryKey: ['posts', user?.id],
    queryFn: () => fetchPostsByUser(user!.id),
    enabled: !!user, // Only run when user exists
  })

  return <PostList posts={posts} />
}
```

## Parallel Queries

```typescript
import { useQueries } from '@tanstack/react-query'

function UserStats({ userIds }: { userIds: string[] }) {
  const results = useQueries({
    queries: userIds.map((id) => ({
      queryKey: ['user', id],
      queryFn: () => fetchUser(id),
    })),
  })

  const isLoading = results.some((r) => r.isLoading)
  const users = results.map((r) => r.data).filter(Boolean)

  if (isLoading) return <Spinner />

  return <UserList users={users} />
}
```

## Suspense Mode (v5)

```typescript
import { useSuspenseQuery } from '@tanstack/react-query'

function UserProfile({ userId }: { userId: string }) {
  // No loading state needed - Suspense handles it
  const { data: user } = useSuspenseQuery({
    queryKey: ['user', userId],
    queryFn: () => fetchUser(userId),
  })

  return <div>{user.name}</div>
}

// Parent component
function App() {
  return (
    <Suspense fallback={<Spinner />}>
      <UserProfile userId="123" />
    </Suspense>
  )
}
```

## React Server Components

```typescript
// app/users/page.tsx
import { dehydrate, HydrationBoundary, QueryClient } from '@tanstack/react-query'

export default async function UsersPage() {
  const queryClient = new QueryClient()

  await queryClient.prefetchQuery({
    queryKey: ['users'],
    queryFn: fetchUsers,
  })

  return (
    <HydrationBoundary state={dehydrate(queryClient)}>
      <UserList />
    </HydrationBoundary>
  )
}

// UserList.tsx - Client component uses cached data
'use client'
function UserList() {
  const { data: users } = useQuery({
    queryKey: ['users'],
    queryFn: fetchUsers,
  })

  return <ul>{users?.map(...)}</ul>
}
```

## Query Options Factory

```typescript
// queries/users.ts
import { queryOptions } from '@tanstack/react-query'

export const userQueries = {
  all: () => queryOptions({
    queryKey: ['users'],
    queryFn: fetchUsers,
  }),
  detail: (id: string) => queryOptions({
    queryKey: ['users', id],
    queryFn: () => fetchUser(id),
    staleTime: 5 * 60 * 1000,
  }),
}

// Usage
const { data } = useQuery(userQueries.detail('123'))
await queryClient.prefetchQuery(userQueries.all())
```

---

*Learned: December 20, 2025*
*Tags: TanStack Query, React Query, Data Fetching, State Management, React*
