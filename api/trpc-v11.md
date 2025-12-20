# tRPC v11 - Type-Safe APIs Without Schema

<div align="center">

![tRPC](https://img.shields.io/badge/tRPC-2596BE?style=for-the-badge&logo=trpc&logoColor=white)
![TypeScript](https://img.shields.io/badge/TypeScript-3178C6?style=for-the-badge&logo=typescript&logoColor=white)
![API](https://img.shields.io/badge/Feature-End_to_End_Type_Safety-orange?style=for-the-badge)

*End-to-end type-safe APIs without code generation or schema files.*

![tRPC](https://raw.githubusercontent.com/trpc/trpc/main/www/static/img/logo.svg)

</div>

## Why tRPC?

```
REST/GraphQL                   tRPC
──────────────────────────     ──────────────────────────
Schema definitions             No schema needed
Code generation                Zero codegen
Manual type sync               Auto type inference
OpenAPI/GraphQL SDL            Just TypeScript
Runtime validation             Compile-time safety
```

## What's New in v11

- TanStack Query v5 support with full Suspense
- Non-JSON content types (FormData, Blob, File)
- Server-Sent Events (SSE) for subscriptions
- React Server Components (RSC) support
- HTTP Batch Stream Link
- Removed .interop() mode

## Installation

```bash
# Server
npm install @trpc/server zod

# Client (React)
npm install @trpc/client @trpc/react-query @tanstack/react-query
```

## Server Setup

```typescript
// server/trpc.ts
import { initTRPC, TRPCError } from '@trpc/server'
import { z } from 'zod'

const t = initTRPC.context<Context>().create()

export const router = t.router
export const publicProcedure = t.procedure

// Protected procedure
export const protectedProcedure = t.procedure.use(async ({ ctx, next }) => {
  if (!ctx.user) {
    throw new TRPCError({ code: 'UNAUTHORIZED' })
  }
  return next({ ctx: { user: ctx.user } })
})
```

## Define Router

```typescript
// server/routers/user.ts
import { z } from 'zod'
import { router, publicProcedure, protectedProcedure } from '../trpc'

export const userRouter = router({
  // Query - fetch data
  getById: publicProcedure
    .input(z.object({ id: z.string() }))
    .query(async ({ input, ctx }) => {
      const user = await ctx.db.user.findUnique({
        where: { id: input.id },
      })
      return user
    }),

  // List with pagination
  list: publicProcedure
    .input(z.object({
      page: z.number().default(1),
      limit: z.number().max(100).default(10),
    }))
    .query(async ({ input, ctx }) => {
      return ctx.db.user.findMany({
        skip: (input.page - 1) * input.limit,
        take: input.limit,
      })
    }),

  // Mutation - modify data
  create: protectedProcedure
    .input(z.object({
      name: z.string().min(1),
      email: z.string().email(),
    }))
    .mutation(async ({ input, ctx }) => {
      return ctx.db.user.create({
        data: input,
      })
    }),

  // Update
  update: protectedProcedure
    .input(z.object({
      id: z.string(),
      name: z.string().optional(),
      email: z.string().email().optional(),
    }))
    .mutation(async ({ input, ctx }) => {
      const { id, ...data } = input
      return ctx.db.user.update({
        where: { id },
        data,
      })
    }),
})
```

## Root Router

```typescript
// server/routers/index.ts
import { router } from '../trpc'
import { userRouter } from './user'
import { postRouter } from './post'

export const appRouter = router({
  user: userRouter,
  post: postRouter,
})

export type AppRouter = typeof appRouter
```

## Next.js Integration

```typescript
// app/api/trpc/[trpc]/route.ts
import { fetchRequestHandler } from '@trpc/server/adapters/fetch'
import { appRouter } from '@/server/routers'
import { createContext } from '@/server/context'

const handler = (req: Request) =>
  fetchRequestHandler({
    endpoint: '/api/trpc',
    req,
    router: appRouter,
    createContext,
  })

export { handler as GET, handler as POST }
```

## Client Setup

```typescript
// lib/trpc.ts
import { createTRPCReact } from '@trpc/react-query'
import type { AppRouter } from '@/server/routers'

export const trpc = createTRPCReact<AppRouter>()
```

```typescript
// app/providers.tsx
'use client'
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'
import { httpBatchLink } from '@trpc/client'
import { trpc } from '@/lib/trpc'

const queryClient = new QueryClient()

const trpcClient = trpc.createClient({
  links: [
    httpBatchLink({
      url: '/api/trpc',
    }),
  ],
})

export function Providers({ children }: { children: React.ReactNode }) {
  return (
    <trpc.Provider client={trpcClient} queryClient={queryClient}>
      <QueryClientProvider client={queryClient}>
        {children}
      </QueryClientProvider>
    </trpc.Provider>
  )
}
```

## Using in Components

```typescript
'use client'
import { trpc } from '@/lib/trpc'

function UserProfile({ userId }: { userId: string }) {
  // Query - fully typed!
  const { data: user, isLoading, error } = trpc.user.getById.useQuery({
    id: userId,
  })

  // Mutation
  const updateUser = trpc.user.update.useMutation({
    onSuccess: () => {
      // Invalidate cache
      trpc.useUtils().user.getById.invalidate({ id: userId })
    },
  })

  if (isLoading) return <Spinner />
  if (error) return <Error message={error.message} />

  return (
    <div>
      <h1>{user?.name}</h1>
      <button onClick={() => updateUser.mutate({
        id: userId,
        name: 'New Name',
      })}>
        Update
      </button>
    </div>
  )
}
```

## v11: FormData & File Uploads

```typescript
// Server
import { z } from 'zod'

export const uploadRouter = router({
  uploadFile: publicProcedure
    .input(z.instanceof(FormData))
    .mutation(async ({ input }) => {
      const file = input.get('file') as File
      const buffer = await file.arrayBuffer()
      // Process file...
      return { filename: file.name, size: file.size }
    }),
})

// Client
const formData = new FormData()
formData.append('file', file)
await trpc.upload.uploadFile.mutate(formData)
```

## v11: Server-Sent Events

```typescript
// Server
export const notificationRouter = router({
  onNewMessage: publicProcedure.subscription(async function* ({ ctx }) {
    while (true) {
      const message = await ctx.messageQueue.next()
      yield message
    }
  }),
})

// Client
const { data } = trpc.notification.onNewMessage.useSubscription(undefined, {
  onData: (message) => {
    console.log('New message:', message)
  },
})
```

## v11: HTTP Batch Stream

```typescript
// Stream responses for large datasets
import { httpBatchStreamLink } from '@trpc/client'

const trpcClient = trpc.createClient({
  links: [
    httpBatchStreamLink({
      url: '/api/trpc',
    }),
  ],
})
```

## React Server Components

```typescript
// app/users/page.tsx
import { createServerSideHelpers } from '@trpc/react-query/server'
import { appRouter } from '@/server/routers'
import { createContext } from '@/server/context'

export default async function UsersPage() {
  const helpers = createServerSideHelpers({
    router: appRouter,
    ctx: await createContext(),
  })

  // Prefetch on server
  const users = await helpers.user.list.fetch({ page: 1 })

  return (
    <ul>
      {users.map(user => (
        <li key={user.id}>{user.name}</li>
      ))}
    </ul>
  )
}
```

## Error Handling

```typescript
import { TRPCError } from '@trpc/server'

// Throw typed errors
if (!user) {
  throw new TRPCError({
    code: 'NOT_FOUND',
    message: 'User not found',
  })
}

// Available codes:
// UNAUTHORIZED, FORBIDDEN, NOT_FOUND,
// BAD_REQUEST, INTERNAL_SERVER_ERROR,
// TIMEOUT, CONFLICT, PRECONDITION_FAILED,
// PAYLOAD_TOO_LARGE, TOO_MANY_REQUESTS
```

---

*Learned: December 20, 2025*
*Tags: tRPC, TypeScript, API, Type Safety, React Query*
