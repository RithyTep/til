# Partial Prerendering (PPR)

Combine static and dynamic content in a single route (Next.js 14+).

## The Concept

```
┌─────────────────────────────────────┐
│  Static Shell (Instant)             │
│  ┌─────────────────────────────────┐│
│  │ Header, Navigation, Footer      ││
│  │                                 ││
│  │  ┌─────────────┐  ┌───────────┐ ││
│  │  │ Dynamic     │  │ Dynamic   │ ││
│  │  │ (Streamed)  │  │ (Streamed)│ ││
│  │  └─────────────┘  └───────────┘ ││
│  │                                 ││
│  └─────────────────────────────────┘│
└─────────────────────────────────────┘
```

## Enable PPR

```tsx
// next.config.js
module.exports = {
  experimental: {
    ppr: true,
  },
};
```

## Example Page

```tsx
// app/page.tsx
import { Suspense } from 'react';

// Static - prerendered at build time
function StaticHeader() {
  return <header>My App</header>;
}

// Dynamic - streamed at request time
async function DynamicUserInfo() {
  const user = await getCurrentUser(); // Uses cookies
  return <p>Welcome, {user.name}</p>;
}

async function DynamicRecommendations() {
  const recs = await getRecommendations();
  return <ul>{recs.map(r => <li key={r.id}>{r.title}</li>)}</ul>;
}

export default function Page() {
  return (
    <div>
      {/* Static shell - instant */}
      <StaticHeader />
      <h1>Dashboard</h1>

      {/* Dynamic holes - streamed */}
      <Suspense fallback={<p>Loading user...</p>}>
        <DynamicUserInfo />
      </Suspense>

      <Suspense fallback={<p>Loading recommendations...</p>}>
        <DynamicRecommendations />
      </Suspense>

      {/* Static footer - instant */}
      <footer>© 2024</footer>
    </div>
  );
}
```

## What Makes Content Dynamic?

```tsx
// These make a component dynamic:
import { cookies, headers } from 'next/headers';
import { searchParams } from 'next/navigation';

// Reading cookies
const token = cookies().get('token');

// Reading headers
const userAgent = headers().get('user-agent');

// Dynamic fetch
fetch(url, { cache: 'no-store' });
```

## Benefits

- ✅ Instant static shell (like SSG)
- ✅ Fresh dynamic content (like SSR)
- ✅ Best of both worlds
- ✅ Better Core Web Vitals
- ✅ Reduced Time to First Byte

## vs Traditional Approaches

| Approach | Static Parts | Dynamic Parts |
|----------|--------------|---------------|
| SSG | ✅ Fast | ❌ Stale |
| SSR | ❌ Slow | ✅ Fresh |
| ISR | ✅ Fast | ⚠️ Eventually fresh |
| **PPR** | ✅ Fast | ✅ Fresh |

---

📅 *Learned: December 20, 2025*
🏷️ *Tags: Next.js, PPR, Partial Prerendering, Performance*
