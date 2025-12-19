# Next.js Caching Strategies

<div align="center">

![Next.js](https://img.shields.io/badge/Next.js-000000?style=for-the-badge&logo=next.js&logoColor=white)
![Cache](https://img.shields.io/badge/Performance-Caching_Layers-green?style=for-the-badge)

*Understanding the four caching layers in Next.js 14+.*

</div>

## 1. Request Memoization

```tsx
// Same fetch is automatically deduped in one render
async function Page() {
  const user = await getUser(); // Fetch 1
  return <Profile user={user} />;
}

async function Profile({ user }) {
  const user = await getUser(); // Same request - reused!
  return <h1>{user.name}</h1>;
}
```

## 2. Data Cache (fetch)

```tsx
// Cached indefinitely (default)
fetch('https://api.example.com/data');

// Revalidate every 60 seconds
fetch('https://api.example.com/data', {
  next: { revalidate: 60 }
});

// No caching
fetch('https://api.example.com/data', {
  cache: 'no-store'
});

// Cache with tags for on-demand revalidation
fetch('https://api.example.com/posts', {
  next: { tags: ['posts'] }
});
```

## 3. Full Route Cache

```tsx
// Static by default (cached at build time)
export default function Page() {
  return <h1>Static Page</h1>;
}

// Dynamic (no caching)
export const dynamic = 'force-dynamic';

// Revalidate every hour
export const revalidate = 3600;
```

## 4. Router Cache (Client-side)

```tsx
// Prefetched routes are cached for 30s (dynamic) or 5min (static)
<Link href="/about" prefetch={true}>About</Link>

// Disable prefetch
<Link href="/about" prefetch={false}>About</Link>
```

## On-Demand Revalidation

```tsx
// app/actions.ts
'use server';

import { revalidatePath, revalidateTag } from 'next/cache';

export async function updatePost() {
  await db.post.update(...);

  // Revalidate specific path
  revalidatePath('/posts');

  // Revalidate by tag
  revalidateTag('posts');
}
```

## Opting Out of Caching

```tsx
// Page level
export const dynamic = 'force-dynamic';
export const revalidate = 0;

// Fetch level
fetch(url, { cache: 'no-store' });

// Using cookies/headers makes route dynamic
import { cookies, headers } from 'next/headers';
```

---

*Learned: December 20, 2025*
*Tags: Next.js, Caching, Performance, Revalidation*
