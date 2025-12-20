# Edge Runtime 2025 - Cloudflare Workers vs Vercel Edge

<div align="center">

![Cloudflare](https://img.shields.io/badge/Cloudflare_Workers-F38020?style=for-the-badge&logo=cloudflare&logoColor=white)
![Vercel](https://img.shields.io/badge/Vercel_Edge-000000?style=for-the-badge&logo=vercel&logoColor=white)

*Ultra-low latency serverless at the edge.*

![Cloudflare Workers](https://raw.githubusercontent.com/cloudflare/cloudflare-docs/production/static/cf-logo-v.svg)

[![Cloudflare Workers](https://img.shields.io/badge/workers.cloudflare.com-orange)](https://workers.cloudflare.com)
[![Vercel Edge](https://img.shields.io/badge/vercel.com-black)](https://vercel.com/docs/functions/edge-functions)

</div>

## Edge Computing Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    EDGE vs TRADITIONAL                           │
│                                                                 │
│   Traditional (Single Region)                                   │
│   ┌─────────┐                     ┌─────────┐                  │
│   │  User   │─────── 200ms ──────►│ Server  │                  │
│   │ (Tokyo) │                     │ (US)    │                  │
│   └─────────┘                     └─────────┘                  │
│                                                                 │
│   Edge (Global Distribution)                                    │
│   ┌─────────┐                     ┌─────────┐                  │
│   │  User   │─────── 10ms ───────►│ Edge    │                  │
│   │ (Tokyo) │                     │ (Tokyo) │                  │
│   └─────────┘                     └─────────┘                  │
│                                                                 │
│   Cold Start Comparison                                         │
│   ├── Cloudflare Workers   <1ms   (V8 isolates)                │
│   ├── Vercel Edge          ~5ms   (V8 isolates)                │
│   ├── AWS Lambda           ~100ms (containers)                 │
│   └── Traditional Server   ~500ms (cold boot)                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## Cloudflare Workers

V8 isolates across 330+ cities globally.

### Basic Worker

```typescript
// src/index.ts
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const url = new URL(request.url);

    // API routing
    if (url.pathname.startsWith('/api/')) {
      return handleAPI(request, env);
    }

    // Static assets
    return env.ASSETS.fetch(request);
  },
};

async function handleAPI(request: Request, env: Env): Promise<Response> {
  const url = new URL(request.url);

  switch (url.pathname) {
    case '/api/user':
      return Response.json({ user: 'Alice', region: request.cf?.colo });

    case '/api/kv':
      const value = await env.MY_KV.get('key');
      return Response.json({ value });

    default:
      return new Response('Not Found', { status: 404 });
  }
}

interface Env {
  MY_KV: KVNamespace;
  ASSETS: Fetcher;
}
```

### Durable Objects (Stateful Edge)

```typescript
// Durable Object for real-time collaboration
export class DocumentRoom implements DurableObject {
  private sessions: Map<string, WebSocket> = new Map();
  private document: string = '';

  constructor(private state: DurableObjectState) {}

  async fetch(request: Request): Promise<Response> {
    const url = new URL(request.url);

    if (url.pathname === '/websocket') {
      const pair = new WebSocketPair();
      const [client, server] = Object.values(pair);

      server.accept();
      const sessionId = crypto.randomUUID();
      this.sessions.set(sessionId, server);

      server.addEventListener('message', (event) => {
        this.document = event.data as string;
        // Broadcast to all connected clients
        this.broadcast(sessionId, this.document);
      });

      server.addEventListener('close', () => {
        this.sessions.delete(sessionId);
      });

      return new Response(null, { status: 101, webSocket: client });
    }

    return new Response('Document Room', { status: 200 });
  }

  private broadcast(excludeId: string, message: string) {
    for (const [id, socket] of this.sessions) {
      if (id !== excludeId) {
        socket.send(message);
      }
    }
  }
}
```

### D1 Database (SQLite at Edge)

```typescript
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    // Query D1 database
    const { results } = await env.DB.prepare(
      'SELECT * FROM users WHERE active = ?'
    )
      .bind(true)
      .all();

    return Response.json(results);
  },
};

interface Env {
  DB: D1Database;
}
```

### R2 Storage

```typescript
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const url = new URL(request.url);
    const key = url.pathname.slice(1);

    if (request.method === 'PUT') {
      await env.BUCKET.put(key, request.body);
      return new Response('Uploaded');
    }

    if (request.method === 'GET') {
      const object = await env.BUCKET.get(key);
      if (!object) return new Response('Not Found', { status: 404 });

      return new Response(object.body, {
        headers: { 'Content-Type': object.httpMetadata?.contentType || '' },
      });
    }

    return new Response('Method Not Allowed', { status: 405 });
  },
};
```

## Vercel Edge Functions

Integrated with Next.js and Vercel platform.

### Edge Middleware

```typescript
// middleware.ts
import { NextResponse } from 'next/server';
import type { NextRequest } from 'next/server';

export function middleware(request: NextRequest) {
  const country = request.geo?.country || 'US';
  const city = request.geo?.city || 'Unknown';

  // Geolocation-based routing
  if (country === 'CN') {
    return NextResponse.redirect(new URL('/cn', request.url));
  }

  // A/B testing
  const bucket = request.cookies.get('ab-bucket')?.value ||
    (Math.random() > 0.5 ? 'a' : 'b');

  const response = NextResponse.next();
  response.cookies.set('ab-bucket', bucket);
  response.headers.set('x-geo-city', city);

  return response;
}

export const config = {
  matcher: ['/((?!api|_next/static|favicon.ico).*)'],
};
```

### Edge API Route

```typescript
// app/api/edge/route.ts
import { NextRequest } from 'next/server';

export const runtime = 'edge';

export async function GET(request: NextRequest) {
  const { searchParams } = new URL(request.url);
  const query = searchParams.get('q');

  // Edge-compatible operations
  const results = await fetch(`https://api.example.com/search?q=${query}`);

  return Response.json({
    results: await results.json(),
    edge: true,
    region: process.env.VERCEL_REGION,
  });
}
```

### Edge with KV Storage

```typescript
// app/api/cache/route.ts
import { kv } from '@vercel/kv';

export const runtime = 'edge';

export async function GET(request: NextRequest) {
  const { searchParams } = new URL(request.url);
  const key = searchParams.get('key');

  // Read from Vercel KV (Redis)
  const cached = await kv.get(key);
  if (cached) {
    return Response.json({ data: cached, source: 'cache' });
  }

  // Fetch and cache
  const data = await fetchExpensiveData();
  await kv.set(key, data, { ex: 3600 }); // 1 hour TTL

  return Response.json({ data, source: 'origin' });
}
```

## Platform Comparison

| Feature | Cloudflare Workers | Vercel Edge |
|---------|-------------------|-------------|
| Global PoPs | 330+ | 100+ |
| Cold Start | <1ms | ~5ms |
| Max Execution | 30s (paid) | 25s |
| Memory Limit | 128MB | 128MB |
| Free Tier | 100k req/day | 100k req/month |
| Native Storage | KV, R2, D1 | KV (Redis) |
| Durable Objects | Yes | No |
| WebSockets | Yes | Limited |
| Framework | Hono, Remix | Next.js native |

## Use Case Decision Matrix

| Scenario | Recommendation |
|----------|---------------|
| Next.js app | Vercel Edge |
| Real-time collaboration | Cloudflare Durable Objects |
| Image transformation | Cloudflare Workers |
| Auth/JWT validation | Either |
| A/B testing | Either |
| Global database | Cloudflare D1 |
| Object storage | Cloudflare R2 |
| Redis caching | Vercel KV |

## Performance Patterns

### Edge Caching

```typescript
// Cloudflare Worker with Cache API
export default {
  async fetch(request: Request): Promise<Response> {
    const cache = caches.default;
    const cacheKey = new Request(request.url, request);

    // Check cache
    let response = await cache.match(cacheKey);
    if (response) {
      return response;
    }

    // Fetch from origin
    response = await fetch(request);

    // Cache successful responses
    if (response.ok) {
      const cached = response.clone();
      cached.headers.set('Cache-Control', 'public, max-age=3600');
      await cache.put(cacheKey, cached);
    }

    return response;
  },
};
```

### Edge Authentication

```typescript
// JWT validation at edge
import { jwtVerify } from 'jose';

export async function middleware(request: NextRequest) {
  const token = request.headers.get('Authorization')?.replace('Bearer ', '');

  if (!token) {
    return Response.json({ error: 'Unauthorized' }, { status: 401 });
  }

  try {
    const secret = new TextEncoder().encode(process.env.JWT_SECRET);
    const { payload } = await jwtVerify(token, secret);

    // Add user info to request headers
    const requestHeaders = new Headers(request.headers);
    requestHeaders.set('x-user-id', payload.sub as string);

    return NextResponse.next({
      request: { headers: requestHeaders },
    });
  } catch {
    return Response.json({ error: 'Invalid token' }, { status: 401 });
  }
}
```

---

*Learned: December 20, 2025*
*Tags: Edge Computing, Cloudflare Workers, Vercel Edge, Serverless, Performance*
