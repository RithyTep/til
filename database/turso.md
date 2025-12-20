# Turso - SQLite at the Edge

<div align="center">

![Turso](https://img.shields.io/badge/Turso-4FF8D2?style=for-the-badge&logo=turso&logoColor=black)
![SQLite](https://img.shields.io/badge/SQLite-003B57?style=for-the-badge&logo=sqlite&logoColor=white)
![Edge](https://img.shields.io/badge/Edge-Distributed-orange?style=for-the-badge)

*SQLite for production - distributed, replicated, and embedded at the edge.*

![Turso](https://turso.tech/og-image.png)

</div>

## Why Turso?

```
Traditional Database           Turso (libSQL)
──────────────────────────     ──────────────────────────
Single region                  Global replicas
Network latency                Edge-close reads
Complex setup                  Instant provisioning
Connection limits              Embedded mode
Always remote                  Local-first possible
```

## Installation

```bash
# CLI
brew install tursodatabase/tap/turso

# SDK
npm install @libsql/client
```

## CLI Basics

```bash
# Login
turso auth login

# Create database
turso db create my-app

# Get connection URL
turso db show my-app --url

# Create auth token
turso db tokens create my-app

# Create replica in region
turso db replicate my-app lhr  # London

# Open SQL shell
turso db shell my-app
```

## Client Setup

```typescript
import { createClient } from '@libsql/client'

// Remote connection
const db = createClient({
  url: process.env.TURSO_DATABASE_URL!,
  authToken: process.env.TURSO_AUTH_TOKEN!,
})

// Embedded replica (local + sync)
const dbWithReplica = createClient({
  url: 'file:local.db',
  syncUrl: process.env.TURSO_DATABASE_URL!,
  authToken: process.env.TURSO_AUTH_TOKEN!,
})

// Sync periodically
await dbWithReplica.sync()
```

## Basic Queries

```typescript
// Execute single statement
await db.execute(`
  CREATE TABLE IF NOT EXISTS users (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    email TEXT UNIQUE NOT NULL,
    name TEXT NOT NULL,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
  )
`)

// Insert with parameters (safe from SQL injection)
await db.execute({
  sql: 'INSERT INTO users (email, name) VALUES (?, ?)',
  args: ['john@example.com', 'John Doe'],
})

// Query data
const result = await db.execute('SELECT * FROM users')
console.log(result.rows)
// [{ id: 1, email: 'john@example.com', name: 'John Doe', ... }]

// Query with parameters
const user = await db.execute({
  sql: 'SELECT * FROM users WHERE id = ?',
  args: [1],
})
```

## Batch Operations

```typescript
// Execute multiple statements in single round-trip
const results = await db.batch([
  {
    sql: 'INSERT INTO users (email, name) VALUES (?, ?)',
    args: ['alice@example.com', 'Alice'],
  },
  {
    sql: 'INSERT INTO users (email, name) VALUES (?, ?)',
    args: ['bob@example.com', 'Bob'],
  },
  'SELECT * FROM users',
])

// Last result contains the SELECT
console.log(results[2].rows)
```

## Transactions

```typescript
const transaction = await db.transaction('write')

try {
  await transaction.execute({
    sql: 'INSERT INTO orders (user_id, total) VALUES (?, ?)',
    args: [1, 99.99],
  })

  await transaction.execute({
    sql: 'UPDATE users SET order_count = order_count + 1 WHERE id = ?',
    args: [1],
  })

  await transaction.commit()
} catch (error) {
  await transaction.rollback()
  throw error
}
```

## With Drizzle ORM

```typescript
import { drizzle } from 'drizzle-orm/libsql'
import { createClient } from '@libsql/client'
import { sqliteTable, text, integer } from 'drizzle-orm/sqlite-core'

const client = createClient({
  url: process.env.TURSO_DATABASE_URL!,
  authToken: process.env.TURSO_AUTH_TOKEN!,
})

const db = drizzle(client)

// Define schema
const users = sqliteTable('users', {
  id: integer('id').primaryKey({ autoIncrement: true }),
  email: text('email').notNull().unique(),
  name: text('name').notNull(),
})

// Type-safe queries
const allUsers = await db.select().from(users)
const user = await db.select().from(users).where(eq(users.id, 1))

await db.insert(users).values({
  email: 'new@example.com',
  name: 'New User',
})
```

## Embedded Replicas

```typescript
// Perfect for edge functions
import { createClient } from '@libsql/client'

const db = createClient({
  url: 'file:/tmp/local-replica.db',
  syncUrl: process.env.TURSO_DATABASE_URL!,
  authToken: process.env.TURSO_AUTH_TOKEN!,
  syncInterval: 60, // Sync every 60 seconds
})

// Initial sync
await db.sync()

// All reads are local (fast!)
const users = await db.execute('SELECT * FROM users')

// Writes go to primary, then sync
await db.execute({
  sql: 'INSERT INTO users (email, name) VALUES (?, ?)',
  args: ['edge@example.com', 'Edge User'],
})
await db.sync()
```

## Next.js Integration

```typescript
// lib/db.ts
import { createClient } from '@libsql/client'

export const db = createClient({
  url: process.env.TURSO_DATABASE_URL!,
  authToken: process.env.TURSO_AUTH_TOKEN!,
})

// app/api/users/route.ts
import { db } from '@/lib/db'
import { NextResponse } from 'next/server'

export async function GET() {
  const result = await db.execute('SELECT * FROM users')
  return NextResponse.json(result.rows)
}

export async function POST(req: Request) {
  const { email, name } = await req.json()

  await db.execute({
    sql: 'INSERT INTO users (email, name) VALUES (?, ?)',
    args: [email, name],
  })

  return NextResponse.json({ success: true })
}
```

## Cloudflare Workers

```typescript
// wrangler.toml
// [vars]
// TURSO_URL = "libsql://..."
// TURSO_AUTH_TOKEN = "..."

import { createClient } from '@libsql/client/web'

export default {
  async fetch(request: Request, env: Env) {
    const db = createClient({
      url: env.TURSO_URL,
      authToken: env.TURSO_AUTH_TOKEN,
    })

    const result = await db.execute('SELECT * FROM users LIMIT 10')

    return Response.json(result.rows)
  },
}
```

## Migrations

```bash
# Using Drizzle Kit
npx drizzle-kit generate
npx drizzle-kit push

# Or manual SQL
turso db shell my-app < migrations/001_init.sql
```

## Pricing Tiers

| Feature | Free | Scaler | Pro |
|---------|------|--------|-----|
| Databases | 500 | Unlimited | Unlimited |
| Locations | 3 | All | All |
| Rows read/mo | 9B | 100B | Custom |
| Rows written/mo | 25M | 100M | Custom |

---

*Learned: December 20, 2025*
*Tags: Turso, SQLite, libSQL, Edge, Database*
