# JavaScript Runtimes 2025 - Node.js vs Deno vs Bun

<div align="center">

![Node.js](https://img.shields.io/badge/Node.js-24-339933?style=for-the-badge&logo=node.js&logoColor=white)
![Deno](https://img.shields.io/badge/Deno-2.0-000000?style=for-the-badge&logo=deno&logoColor=white)
![Bun](https://img.shields.io/badge/Bun-1.3-FBF0DF?style=for-the-badge&logo=bun&logoColor=black)

*Choosing the right runtime for production workloads.*

[![Bun](https://img.shields.io/github/stars/oven-sh/bun?style=social&label=Bun)](https://github.com/oven-sh/bun)
[![Deno](https://img.shields.io/badge/Deno-deno.land-black)](https://deno.land)
[![Node.js](https://img.shields.io/badge/Node.js-nodejs.org-green)](https://nodejs.org)

</div>

## Runtime Comparison

```
┌─────────────────────────────────────────────────────────────────┐
│                 RUNTIME PERFORMANCE (2025)                       │
│                                                                 │
│   Cold Start Time (lower is better)                             │
│   ├── Bun 1.3     ████░░░░░░░░░░░░░░░░  ~8ms                   │
│   ├── Deno 2.5    ██████████████░░░░░░  ~35ms                  │
│   └── Node.js 24  ████████████████░░░░  ~42ms                  │
│                                                                 │
│   HTTP Throughput (higher is better)                            │
│   ├── Bun         ████████████████████  70k+ req/s             │
│   ├── Deno        ████████████████░░░░  55k req/s              │
│   └── Node.js     ██████████████░░░░░░  45k req/s              │
│                                                                 │
│   npm Install Speed (lower is better)                           │
│   ├── Bun         ██░░░░░░░░░░░░░░░░░░  ~2s                    │
│   ├── Deno        ████░░░░░░░░░░░░░░░░  ~5s                    │
│   └── npm         ████████████████████  ~30s                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## Bun 1.3

All-in-one toolkit: runtime, bundler, test runner, package manager.

### Key Features

```typescript
// Native TypeScript - no transpilation needed
// server.ts
const server = Bun.serve({
  port: 3000,
  fetch(req) {
    const url = new URL(req.url);

    if (url.pathname === '/api/users') {
      return Response.json({ users: ['Alice', 'Bob'] });
    }

    return new Response('Not Found', { status: 404 });
  },
});

console.log(`Server running at http://localhost:${server.port}`);
```

### Built-in SQLite

```typescript
import { Database } from 'bun:sqlite';

const db = new Database('app.db');

// Create table
db.run(`
  CREATE TABLE IF NOT EXISTS users (
    id INTEGER PRIMARY KEY,
    name TEXT,
    email TEXT UNIQUE
  )
`);

// Prepared statements
const insertUser = db.prepare(
  'INSERT INTO users (name, email) VALUES ($name, $email)'
);

insertUser.run({ $name: 'Alice', $email: 'alice@example.com' });

// Query
const users = db.query('SELECT * FROM users').all();
```

### Bun Shell

```typescript
import { $ } from 'bun';

// Shell commands with template literals
const files = await $`ls -la`.text();
const branch = await $`git branch --show-current`.text();

// Pipe commands
await $`cat package.json | grep "version"`;

// Environment variables
await $`DATABASE_URL=${process.env.DB_URL} prisma migrate deploy`;
```

### Performance Comparison

```typescript
// package.json
// bun install: ~500ms
// npm install: ~15s
// pnpm install: ~3s

// bun run build: ~200ms
// npm run build: ~800ms
```

## Deno 2.0

Secure by default with native TypeScript and npm compatibility.

### npm Compatibility

```typescript
// deno.json
{
  "imports": {
    "express": "npm:express@4",
    "prisma": "npm:@prisma/client@5"
  }
}
```

```typescript
// Works with npm packages directly
import express from 'npm:express@4';
import { PrismaClient } from 'npm:@prisma/client';

const app = express();
const prisma = new PrismaClient();

app.get('/users', async (req, res) => {
  const users = await prisma.user.findMany();
  res.json(users);
});

app.listen(3000);
```

### Security Model

```bash
# Explicit permissions required
deno run --allow-net --allow-read server.ts

# Permission prompt at runtime
deno run --prompt server.ts

# Allow specific domains only
deno run --allow-net=api.example.com server.ts
```

### Deno Deploy

```typescript
// Instant global deployment
Deno.serve((req) => {
  const url = new URL(req.url);

  if (url.pathname === '/api/time') {
    return Response.json({
      time: new Date().toISOString(),
      region: Deno.env.get('DENO_REGION'),
    });
  }

  return new Response('Hello from the edge!');
});
```

### Built-in Tooling

```bash
# Format code
deno fmt

# Lint code
deno lint

# Run tests
deno test

# Type check
deno check server.ts

# Compile to executable
deno compile --output=app server.ts
```

## Node.js 24

Mature ecosystem with enterprise-grade stability.

### Native Fetch (Stable)

```typescript
// No more node-fetch needed
const response = await fetch('https://api.example.com/users');
const users = await response.json();
```

### Native Test Runner

```typescript
// test/user.test.ts
import { test, describe, beforeEach, mock } from 'node:test';
import assert from 'node:assert';

describe('UserService', () => {
  beforeEach(() => {
    // Setup
  });

  test('creates user with valid email', async () => {
    const user = await createUser('test@example.com');
    assert.strictEqual(user.email, 'test@example.com');
  });

  test('mocking external calls', async () => {
    const mockFetch = mock.fn(() =>
      Promise.resolve({ json: () => ({ id: 1 }) })
    );

    const result = await fetchUser(1, mockFetch);
    assert.strictEqual(mockFetch.mock.calls.length, 1);
  });
});
```

### Watch Mode

```bash
# Auto-restart on changes
node --watch server.js

# With TypeScript (via loader)
node --watch --loader tsx server.ts
```

### Permission Model (Experimental)

```bash
# Node.js 24 experimental permissions
node --experimental-permission --allow-fs-read=./data server.js
```

## When to Use Each

| Use Case | Recommended Runtime |
|----------|-------------------|
| Startup/MVP | Bun - fastest development |
| Enterprise Production | Node.js - stability, ecosystem |
| Security-Critical | Deno - permissions model |
| Edge Functions | Deno Deploy or Bun |
| Existing Node.js Project | Node.js or gradual Bun migration |
| New TypeScript Project | Bun or Deno |
| Monorepo with Workspaces | Bun (fast installs) |

## Framework Compatibility

| Framework | Node.js | Deno | Bun |
|-----------|---------|------|-----|
| Next.js | Full | Partial | Full |
| Express | Full | Full | Full |
| Fastify | Full | Full | Full |
| Hono | Full | Full | Full |
| Remix | Full | Full | Full |
| SvelteKit | Full | Full | Full |
| Astro | Full | Full | Full |

## Migration Path

### Node.js to Bun

```bash
# Replace package manager
rm -rf node_modules package-lock.json
bun install

# Update scripts in package.json
{
  "scripts": {
    "dev": "bun run --watch src/index.ts",
    "start": "bun src/index.ts",
    "test": "bun test"
  }
}

# Run existing Node.js code
bun run src/index.ts  # Just works
```

### Node.js to Deno

```bash
# Add deno.json
{
  "imports": {
    "@/": "./src/"
  },
  "tasks": {
    "dev": "deno run --watch --allow-all src/index.ts",
    "start": "deno run --allow-all src/index.ts"
  }
}

# Update imports to use npm: prefix
import express from 'npm:express';
```

---

*Learned: December 20, 2025*
*Tags: Node.js, Deno, Bun, JavaScript, Runtime, Performance, TypeScript*
