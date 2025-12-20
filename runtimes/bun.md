# Bun - All-in-One JavaScript Runtime

<div align="center">

![Bun](https://img.shields.io/badge/Bun-000000?style=for-the-badge&logo=bun&logoColor=white)
![Runtime](https://img.shields.io/badge/Runtime-Ultrafast-orange?style=for-the-badge)
![Bundler](https://img.shields.io/badge/Bundler-Built_In-blue?style=for-the-badge)

*Incredibly fast JavaScript runtime, bundler, test runner, and package manager - all in one.*

![Bun](https://raw.githubusercontent.com/oven-sh/bun/main/docs/public/logo.svg)

</div>

## Why Bun?

```
Node.js Ecosystem              Bun
──────────────────────────     ──────────────────────────
Node.js runtime                Bun runtime (Zig + JavaScriptCore)
npm/yarn/pnpm                  bun install (100x faster)
webpack/vite/esbuild          bun build (built-in bundler)
Jest/Vitest                    bun test (built-in)
ts-node/tsx                    Native TypeScript/JSX
```

## Performance Benchmarks

```
Startup Time:     < 50ms (vs Node ~300ms)
HTTP Server:      70,000+ req/sec
Package Install:  Up to 100x faster than npm
File Operations:  90% Node.js fs compatibility
```

## Installation

```bash
# macOS/Linux
curl -fsSL https://bun.sh/install | bash

# Windows (via Scoop)
scoop install bun

# Check version
bun --version
```

## Package Management

```bash
# Install dependencies (blazing fast)
bun install

# Add a package
bun add express

# Add dev dependency
bun add -d typescript

# Remove package
bun remove lodash

# Run scripts
bun run dev
bun run build
```

## Native TypeScript Support

```typescript
// server.ts - Run directly without config!
import { serve } from "bun";

serve({
  port: 3000,
  fetch(req) {
    const url = new URL(req.url);

    if (url.pathname === "/api/hello") {
      return Response.json({ message: "Hello from Bun!" });
    }

    return new Response("Not Found", { status: 404 });
  },
});

console.log("Server running at http://localhost:3000");
```

```bash
# Run TypeScript directly
bun run server.ts
```

## Built-in Bundler

```typescript
// Build for production
await Bun.build({
  entrypoints: ['./src/index.tsx'],
  outdir: './dist',
  minify: true,
  splitting: true,
  sourcemap: 'external',
  target: 'browser',
});

// Or via CLI
// bun build ./src/index.tsx --outdir ./dist --minify
```

### HTML Import (Bun 1.2+)

```javascript
// Import HTML directly!
import html from "./index.html";

// Bun handles asset bundling automatically
```

### Built-in Tailwind CSS

```javascript
// No config needed - just import CSS
import "./styles.css"; // Tailwind processed automatically
```

## Built-in Test Runner

```typescript
// math.test.ts
import { expect, test, describe } from "bun:test";
import { add, multiply } from "./math";

describe("math operations", () => {
  test("adds two numbers", () => {
    expect(add(2, 3)).toBe(5);
  });

  test("multiplies two numbers", () => {
    expect(multiply(4, 5)).toBe(20);
  });
});
```

```bash
# Run tests
bun test

# Watch mode
bun test --watch

# Coverage
bun test --coverage
```

## File Operations

```typescript
// Reading files
const file = Bun.file("./data.json");
const content = await file.text();
const json = await file.json();

// Writing files
await Bun.write("./output.txt", "Hello, World!");
await Bun.write("./data.json", JSON.stringify({ foo: "bar" }));

// Streaming large files
const writer = Bun.file("./large.txt").writer();
writer.write("chunk 1");
writer.write("chunk 2");
await writer.end();
```

## SQLite Built-in

```typescript
import { Database } from "bun:sqlite";

const db = new Database("mydb.sqlite");

// Create table
db.run(`
  CREATE TABLE IF NOT EXISTS users (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT,
    email TEXT UNIQUE
  )
`);

// Insert
const insert = db.prepare("INSERT INTO users (name, email) VALUES (?, ?)");
insert.run("John", "john@example.com");

// Query
const users = db.query("SELECT * FROM users").all();
console.log(users);
```

## HTTP Server

```typescript
// High-performance HTTP server
Bun.serve({
  port: 3000,

  async fetch(req) {
    const url = new URL(req.url);

    // JSON API
    if (url.pathname === "/api/users") {
      const users = await getUsers();
      return Response.json(users);
    }

    // Static files
    if (url.pathname.startsWith("/static/")) {
      const file = Bun.file(`./public${url.pathname}`);
      return new Response(file);
    }

    // HTML response
    return new Response("<h1>Hello Bun!</h1>", {
      headers: { "Content-Type": "text/html" },
    });
  },

  // WebSocket support
  websocket: {
    message(ws, message) {
      ws.send(`Echo: ${message}`);
    },
  },
});
```

## Node.js Compatibility

```typescript
// Most Node.js APIs work out of the box
import fs from "fs";
import path from "path";
import crypto from "crypto";
import { EventEmitter } from "events";

// Express works too!
import express from "express";
const app = express();

app.get("/", (req, res) => {
  res.send("Express on Bun!");
});

app.listen(3000);
```

## Bun 1.2+ Features

```typescript
// Bytecode caching (faster startup)
// Automatic - no config needed

// FormData handling
const formData = await req.formData();
const file = formData.get("file") as File;

// Binary data support
const blob = new Blob(["Hello"]);
const uint8 = new Uint8Array([1, 2, 3]);
```

## When to Use Bun

| Use Case | Recommendation |
|----------|---------------|
| New projects | Excellent choice |
| API servers | Great performance |
| CLI tools | Fast startup time |
| Build tooling | Built-in bundler |
| Testing | Built-in test runner |
| Legacy Node apps | Test compatibility first |

---

*Learned: December 20, 2025*
*Tags: Bun, JavaScript, Runtime, Bundler, TypeScript*
