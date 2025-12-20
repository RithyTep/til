# Drizzle ORM - Type-Safe SQL for TypeScript

<div align="center">

![Drizzle](https://img.shields.io/badge/Drizzle-C5F74F?style=for-the-badge&logo=drizzle&logoColor=black)
![TypeScript](https://img.shields.io/badge/TypeScript-3178C6?style=for-the-badge&logo=typescript&logoColor=white)
![SQL](https://img.shields.io/badge/SQL-First-blue?style=for-the-badge)

*If you know SQL, you know Drizzle. Lightweight, type-safe ORM with zero dependencies.*

![Drizzle ORM](https://raw.githubusercontent.com/drizzle-team/drizzle-orm/main/misc/readme/logo-github-sq-dark.svg)

</div>

## Why Drizzle?

```
Prisma                         Drizzle
──────────────────────────     ──────────────────────────
Schema file (.prisma)          TypeScript schemas
Code generation required       No generation needed
Query abstraction              SQL-like syntax
Heavy runtime                  ~7.4kb, zero deps
N+1 by default                 Always 1 query
```

## Installation

```bash
# Core + PostgreSQL
npm install drizzle-orm postgres
npm install -D drizzle-kit

# Or with MySQL
npm install drizzle-orm mysql2

# Or with SQLite
npm install drizzle-orm better-sqlite3
```

## Schema Definition

```typescript
// db/schema.ts
import { pgTable, serial, text, timestamp, integer, boolean } from 'drizzle-orm/pg-core'

export const users = pgTable('users', {
  id: serial('id').primaryKey(),
  name: text('name').notNull(),
  email: text('email').notNull().unique(),
  role: text('role', { enum: ['admin', 'user'] }).default('user'),
  createdAt: timestamp('created_at').defaultNow(),
})

export const posts = pgTable('posts', {
  id: serial('id').primaryKey(),
  title: text('title').notNull(),
  content: text('content'),
  published: boolean('published').default(false),
  authorId: integer('author_id').references(() => users.id),
  createdAt: timestamp('created_at').defaultNow(),
})

// Infer types automatically
export type User = typeof users.$inferSelect
export type NewUser = typeof users.$inferInsert
export type Post = typeof posts.$inferSelect
```

## Database Connection

```typescript
// db/index.ts
import { drizzle } from 'drizzle-orm/postgres-js'
import postgres from 'postgres'
import * as schema from './schema'

const connection = postgres(process.env.DATABASE_URL!)
export const db = drizzle(connection, { schema })
```

## CRUD Operations

### Select Queries

```typescript
import { eq, and, or, like, desc, asc } from 'drizzle-orm'
import { db } from './db'
import { users, posts } from './db/schema'

// Select all
const allUsers = await db.select().from(users)

// Select with conditions
const admins = await db
  .select()
  .from(users)
  .where(eq(users.role, 'admin'))

// Select specific columns
const names = await db
  .select({ id: users.id, name: users.name })
  .from(users)

// Complex conditions
const filtered = await db
  .select()
  .from(users)
  .where(
    and(
      eq(users.role, 'user'),
      like(users.email, '%@gmail.com')
    )
  )
  .orderBy(desc(users.createdAt))
  .limit(10)
```

### Insert

```typescript
// Single insert
const newUser = await db
  .insert(users)
  .values({
    name: 'John Doe',
    email: 'john@example.com',
  })
  .returning()

// Bulk insert
await db.insert(users).values([
  { name: 'Alice', email: 'alice@example.com' },
  { name: 'Bob', email: 'bob@example.com' },
])

// Upsert (insert or update)
await db
  .insert(users)
  .values({ id: 1, name: 'Updated', email: 'new@example.com' })
  .onConflictDoUpdate({
    target: users.id,
    set: { name: 'Updated' },
  })
```

### Update

```typescript
await db
  .update(users)
  .set({ name: 'Jane Doe' })
  .where(eq(users.id, 1))
```

### Delete

```typescript
await db
  .delete(users)
  .where(eq(users.id, 1))
```

## Relations & Joins

```typescript
// Define relations
import { relations } from 'drizzle-orm'

export const usersRelations = relations(users, ({ many }) => ({
  posts: many(posts),
}))

export const postsRelations = relations(posts, ({ one }) => ({
  author: one(users, {
    fields: [posts.authorId],
    references: [users.id],
  }),
}))

// Query with relations
const usersWithPosts = await db.query.users.findMany({
  with: {
    posts: true,
  },
})

// Manual join
const result = await db
  .select({
    user: users,
    post: posts,
  })
  .from(users)
  .leftJoin(posts, eq(users.id, posts.authorId))
```

## Transactions

```typescript
await db.transaction(async (tx) => {
  const [user] = await tx
    .insert(users)
    .values({ name: 'John', email: 'john@example.com' })
    .returning()

  await tx
    .insert(posts)
    .values({
      title: 'First Post',
      authorId: user.id,
    })
})
```

## Drizzle Kit (Migrations)

```typescript
// drizzle.config.ts
import { defineConfig } from 'drizzle-kit'

export default defineConfig({
  schema: './db/schema.ts',
  out: './drizzle',
  dialect: 'postgresql',
  dbCredentials: {
    url: process.env.DATABASE_URL!,
  },
})
```

```bash
# Generate migration
npx drizzle-kit generate

# Apply migrations
npx drizzle-kit migrate

# Push schema directly (dev only)
npx drizzle-kit push

# Open Drizzle Studio
npx drizzle-kit studio
```

## Serverless Support

```typescript
// Works with all serverless databases
// Neon
import { neon } from '@neondatabase/serverless'
import { drizzle } from 'drizzle-orm/neon-http'

const sql = neon(process.env.DATABASE_URL!)
const db = drizzle(sql)

// PlanetScale
import { connect } from '@planetscale/database'
import { drizzle } from 'drizzle-orm/planetscale-serverless'

const connection = connect({ url: process.env.DATABASE_URL })
const db = drizzle(connection)

// Cloudflare D1
import { drizzle } from 'drizzle-orm/d1'
const db = drizzle(env.DB)
```

## Type Safety Example

```typescript
// Full type inference
const user = await db.query.users.findFirst({
  where: eq(users.email, 'john@example.com'),
})

// user is typed as User | undefined
if (user) {
  console.log(user.name)  // TypeScript knows this exists
  console.log(user.foo)   // Error: Property 'foo' does not exist
}

// Insert type checking
await db.insert(users).values({
  name: 'John',
  // Error: 'email' is required
})
```

## Best Practices

```typescript
// 1. Use prepared statements for repeated queries
const getUser = db
  .select()
  .from(users)
  .where(eq(users.id, sql.placeholder('id')))
  .prepare('get_user')

const user = await getUser.execute({ id: 1 })

// 2. Use transactions for multiple operations
// 3. Always handle errors with DrizzleQueryError
// 4. Use identity columns for PostgreSQL (recommended in 2025)
```

---

*Learned: December 20, 2025*
*Tags: Drizzle, ORM, TypeScript, SQL, PostgreSQL, Database*
