# TanStack Router - Type-Safe Routing for React

<div align="center">

![TanStack](https://img.shields.io/badge/TanStack-FF4154?style=for-the-badge&logo=react&logoColor=white)
![TypeScript](https://img.shields.io/badge/TypeScript-3178C6?style=for-the-badge&logo=typescript&logoColor=white)
![Router](https://img.shields.io/badge/Feature-Type_Safe_Routing-orange?style=for-the-badge)

*100% type-safe routing with built-in caching, search params management, and code splitting.*

![TanStack Router](https://upload.wikimedia.org/wikipedia/commons/a/a7/React-icon.svg)

</div>

## Why TanStack Router?

```
Traditional Router          TanStack Router
─────────────────────       ─────────────────────
Manual type assertions      Full type inference
String-based paths          Autocompleted paths
No param validation         Schema-validated params
Basic search params         First-class search APIs
```

## Installation

```bash
npm install @tanstack/react-router
# or
pnpm add @tanstack/react-router
```

## File-Based Routing

```
src/
├── routes/
│   ├── __root.tsx        # Root layout
│   ├── index.tsx         # /
│   ├── about.tsx         # /about
│   ├── posts/
│   │   ├── index.tsx     # /posts
│   │   ├── $postId.tsx   # /posts/:postId
│   │   └── $postId/
│   │       └── edit.tsx  # /posts/:postId/edit
```

## Type-Safe Route Definitions

```typescript
// routes/__root.tsx
import { createRootRoute, Outlet } from '@tanstack/react-router'

export const Route = createRootRoute({
  component: () => (
    <div>
      <nav>{/* Navigation */}</nav>
      <Outlet />
    </div>
  ),
})

// routes/posts/$postId.tsx
import { createFileRoute } from '@tanstack/react-router'

export const Route = createFileRoute('/posts/$postId')({
  // Loader runs before component renders
  loader: async ({ params }) => {
    // params.postId is fully typed!
    const post = await fetchPost(params.postId)
    return { post }
  },
  component: PostComponent,
})

function PostComponent() {
  const { post } = Route.useLoaderData()
  const { postId } = Route.useParams()

  return <article>{post.title}</article>
}
```

## Type-Safe Search Parameters

```typescript
import { createFileRoute } from '@tanstack/react-router'
import { z } from 'zod'

// Define search params schema
const searchSchema = z.object({
  page: z.number().default(1),
  filter: z.enum(['all', 'active', 'completed']).default('all'),
  sort: z.enum(['date', 'name']).optional(),
})

export const Route = createFileRoute('/posts')({
  validateSearch: searchSchema,
  component: PostsComponent,
})

function PostsComponent() {
  // Fully typed search params!
  const { page, filter, sort } = Route.useSearch()

  // Navigate with type-safe search updates
  const navigate = Route.useNavigate()

  return (
    <button onClick={() => navigate({ search: { page: page + 1 } })}>
      Next Page
    </button>
  )
}
```

## Type-Safe Navigation

```typescript
import { Link, useNavigate } from '@tanstack/react-router'

function Navigation() {
  const navigate = useNavigate()

  return (
    <>
      {/* Autocomplete for paths and params */}
      <Link to="/posts/$postId" params={{ postId: '123' }}>
        View Post
      </Link>

      {/* TypeScript error if params missing */}
      <Link to="/posts/$postId">  {/* Error: missing postId */}
        Invalid Link
      </Link>

      {/* Programmatic navigation */}
      <button onClick={() => navigate({
        to: '/posts/$postId',
        params: { postId: '456' },
        search: { tab: 'comments' },
      })}>
        Go to Post
      </button>
    </>
  )
}
```

## Data Loading with Caching

```typescript
export const Route = createFileRoute('/posts/$postId')({
  // Loader with automatic caching
  loader: async ({ params, context }) => {
    return context.queryClient.ensureQueryData({
      queryKey: ['post', params.postId],
      queryFn: () => fetchPost(params.postId),
      staleTime: 1000 * 60 * 5, // 5 minutes
    })
  },

  // Prefetch on hover
  preload: true,
  preloadDelay: 50,

  component: PostComponent,
})
```

## Nested Layouts

```typescript
// routes/dashboard.tsx - Parent layout
export const Route = createFileRoute('/dashboard')({
  component: DashboardLayout,
})

function DashboardLayout() {
  return (
    <div className="dashboard">
      <Sidebar />
      <main>
        <Outlet /> {/* Child routes render here */}
      </main>
    </div>
  )
}

// routes/dashboard/settings.tsx - Child route
export const Route = createFileRoute('/dashboard/settings')({
  component: SettingsPage,
})
```

## Error Boundaries

```typescript
export const Route = createFileRoute('/posts/$postId')({
  loader: async ({ params }) => {
    const post = await fetchPost(params.postId)
    if (!post) throw new Error('Post not found')
    return { post }
  },

  // Route-level error boundary
  errorComponent: ({ error }) => (
    <div className="error">
      <h2>Error loading post</h2>
      <p>{error.message}</p>
    </div>
  ),

  // Pending/loading state
  pendingComponent: () => <Spinner />,

  component: PostComponent,
})
```

## Code Splitting

```typescript
// Automatic code splitting with .lazy.tsx files
// routes/posts/$postId.lazy.tsx
import { createLazyFileRoute } from '@tanstack/react-router'

export const Route = createLazyFileRoute('/posts/$postId')({
  component: () => import('./PostComponent'),
})
```

## Key Benefits

- **100% Type-Safe**: No manual type assertions needed
- **Autocomplete**: Full path and param autocompletion
- **Built-in Caching**: Automatic data caching and preloading
- **Search Params**: First-class URL state management
- **Code Splitting**: Automatic route-based splitting
- **Lightweight**: ~12kb minified

---

*Learned: December 20, 2025*
*Tags: TanStack, React, Router, TypeScript, Type Safety*
