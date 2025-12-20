# Hono - Ultrafast Web Framework for Edge

<div align="center">

![Hono](https://img.shields.io/badge/Hono-E36002?style=for-the-badge&logo=hono&logoColor=white)
![Edge](https://img.shields.io/badge/Edge-Computing-blue?style=for-the-badge)
![TypeScript](https://img.shields.io/badge/TypeScript-3178C6?style=for-the-badge&logo=typescript&logoColor=white)

*Small, simple, and ultrafast web framework built on Web Standards.*

![Hono](https://hono.dev/images/hono-title.png)

</div>

## Why Hono?

```
Express.js                     Hono
──────────────────────────     ──────────────────────────
500kb+ size                    14kb (hono/tiny)
Node.js only                   Any JS runtime
Callback-based                 Web Standards API
Slower routing                 RegExpRouter (fastest)
No TypeScript                  TypeScript-first
```

## Multi-Runtime Support

```
Cloudflare Workers    Fastly Compute
Deno                  Bun
AWS Lambda            Lambda@Edge
Vercel                Node.js
```

## Installation

```bash
# Generic
npm install hono

# With runtime-specific adapters
npm install hono @hono/node-server  # Node.js
```

## Basic Example

```typescript
import { Hono } from 'hono'

const app = new Hono()

app.get('/', (c) => c.text('Hello Hono!'))

app.get('/json', (c) => c.json({ message: 'Hello' }))

app.post('/posts', async (c) => {
  const body = await c.req.json()
  return c.json({ created: body }, 201)
})

export default app
```

## Routing

```typescript
const app = new Hono()

// Basic routes
app.get('/users', (c) => c.text('List users'))
app.post('/users', (c) => c.text('Create user'))
app.put('/users/:id', (c) => c.text('Update user'))
app.delete('/users/:id', (c) => c.text('Delete user'))

// Path parameters
app.get('/users/:id', (c) => {
  const id = c.req.param('id')
  return c.json({ userId: id })
})

// Wildcards
app.get('/files/*', (c) => {
  const path = c.req.param('*')
  return c.text(`File: ${path}`)
})

// Route groups
const api = new Hono()
api.get('/users', (c) => c.json([]))
api.get('/posts', (c) => c.json([]))

app.route('/api/v1', api)
```

## Middleware

```typescript
import { Hono } from 'hono'
import { cors } from 'hono/cors'
import { logger } from 'hono/logger'
import { prettyJSON } from 'hono/pretty-json'
import { basicAuth } from 'hono/basic-auth'
import { jwt } from 'hono/jwt'

const app = new Hono()

// Built-in middleware
app.use('*', logger())
app.use('*', cors())
app.use('*', prettyJSON())

// Auth middleware
app.use('/admin/*', basicAuth({
  username: 'admin',
  password: 'secret',
}))

// JWT middleware
app.use('/api/*', jwt({
  secret: process.env.JWT_SECRET!,
}))

// Custom middleware
app.use('*', async (c, next) => {
  const start = Date.now()
  await next()
  const ms = Date.now() - start
  c.header('X-Response-Time', `${ms}ms`)
})
```

## Request Handling

```typescript
app.post('/upload', async (c) => {
  // JSON body
  const json = await c.req.json()

  // Form data
  const formData = await c.req.formData()
  const file = formData.get('file')

  // Query parameters
  const page = c.req.query('page')
  const { page, limit } = c.req.queries()

  // Headers
  const auth = c.req.header('Authorization')

  // Cookies
  const session = c.req.cookie('session')

  return c.json({ success: true })
})
```

## Response Helpers

```typescript
app.get('/responses', (c) => {
  // Text
  return c.text('Hello')

  // JSON
  return c.json({ message: 'Hello' })

  // HTML
  return c.html('<h1>Hello</h1>')

  // Redirect
  return c.redirect('/new-location')

  // Status codes
  return c.json({ error: 'Not found' }, 404)

  // Headers
  c.header('X-Custom', 'value')
  return c.json({ data: 'with header' })

  // Streaming
  return c.stream(async (stream) => {
    await stream.write('Hello ')
    await stream.write('World')
  })
})
```

## Validation with Zod

```typescript
import { zValidator } from '@hono/zod-validator'
import { z } from 'zod'

const userSchema = z.object({
  name: z.string().min(1),
  email: z.string().email(),
  age: z.number().min(18),
})

app.post(
  '/users',
  zValidator('json', userSchema),
  (c) => {
    const user = c.req.valid('json')
    // user is fully typed!
    return c.json({ created: user })
  }
)

// Query validation
const querySchema = z.object({
  page: z.coerce.number().default(1),
  limit: z.coerce.number().max(100).default(10),
})

app.get(
  '/posts',
  zValidator('query', querySchema),
  (c) => {
    const { page, limit } = c.req.valid('query')
    return c.json({ page, limit })
  }
)
```

## Error Handling

```typescript
import { HTTPException } from 'hono/http-exception'

// Throw HTTP errors
app.get('/protected', (c) => {
  if (!c.req.header('Authorization')) {
    throw new HTTPException(401, { message: 'Unauthorized' })
  }
  return c.json({ secret: 'data' })
})

// Global error handler
app.onError((err, c) => {
  if (err instanceof HTTPException) {
    return c.json({ error: err.message }, err.status)
  }
  console.error(err)
  return c.json({ error: 'Internal Server Error' }, 500)
})

// Not found handler
app.notFound((c) => {
  return c.json({ error: 'Route not found' }, 404)
})
```

## Deploy to Cloudflare Workers

```typescript
// src/index.ts
import { Hono } from 'hono'

type Bindings = {
  DB: D1Database
  KV: KVNamespace
}

const app = new Hono<{ Bindings: Bindings }>()

app.get('/data', async (c) => {
  const result = await c.env.DB.prepare('SELECT * FROM users').all()
  return c.json(result)
})

export default app
```

```toml
# wrangler.toml
name = "my-api"
main = "src/index.ts"
compatibility_date = "2024-01-01"

[[d1_databases]]
binding = "DB"
database_name = "my-db"
database_id = "xxx"
```

## Deploy to Bun

```typescript
import { Hono } from 'hono'

const app = new Hono()

app.get('/', (c) => c.text('Hello Bun!'))

export default {
  port: 3000,
  fetch: app.fetch,
}
```

## Deploy to Node.js

```typescript
import { serve } from '@hono/node-server'
import { Hono } from 'hono'

const app = new Hono()

app.get('/', (c) => c.text('Hello Node!'))

serve({
  fetch: app.fetch,
  port: 3000,
})
```

## RPC Mode (Type-Safe Client)

```typescript
// Server
import { Hono } from 'hono'
import { hc } from 'hono/client'

const app = new Hono()
  .get('/users/:id', (c) => {
    return c.json({ id: c.req.param('id'), name: 'John' })
  })
  .post('/users', async (c) => {
    const body = await c.req.json()
    return c.json({ created: body })
  })

export type AppType = typeof app

// Client (fully typed!)
const client = hc<AppType>('http://localhost:3000')
const user = await client.users[':id'].$get({ param: { id: '1' } })
```

---

*Learned: December 20, 2025*
*Tags: Hono, Edge, Cloudflare, TypeScript, Web Framework*
