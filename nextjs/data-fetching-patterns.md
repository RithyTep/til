# Data Fetching Patterns in Next.js

Best practices for fetching data in App Router applications.

## Server Component Fetch

```tsx
// Fetching in Server Components (recommended)
async function PostsPage() {
  // Automatically deduped, cached
  const posts = await fetch('https://api.example.com/posts');
  const data = await posts.json();

  return (
    <ul>
      {data.map(post => <li key={post.id}>{post.title}</li>)}
    </ul>
  );
}
```

## Parallel Data Fetching

```tsx
// ❌ Sequential - slow
async function Page() {
  const user = await getUser();
  const posts = await getPosts(); // Waits for user
  const comments = await getComments(); // Waits for posts
}

// ✅ Parallel - fast
async function Page() {
  const [user, posts, comments] = await Promise.all([
    getUser(),
    getPosts(),
    getComments(),
  ]);
}
```

## Preload Pattern

```tsx
// lib/data.ts
import { cache } from 'react';

// Cache the function
export const getUser = cache(async (id: string) => {
  const res = await fetch(`/api/users/${id}`);
  return res.json();
});

// Preload function
export const preloadUser = (id: string) => {
  void getUser(id);
};
```

```tsx
// app/user/[id]/page.tsx
import { getUser, preloadUser } from '@/lib/data';

export default async function UserPage({ params }) {
  // Start fetching early
  preloadUser(params.id);

  // ... other work ...

  // Data is likely ready by now
  const user = await getUser(params.id);
  return <Profile user={user} />;
}
```

## Sequential Fetching (When Needed)

```tsx
// When data depends on previous fetch
async function Page({ params }) {
  // Must get user first to know their posts
  const user = await getUser(params.id);
  const posts = await getPosts(user.id);

  return <UserPosts user={user} posts={posts} />;
}
```

## Fetch with Revalidation

```tsx
// Time-based revalidation
const data = await fetch('https://api.example.com/data', {
  next: { revalidate: 3600 }, // Revalidate every hour
});

// On-demand revalidation with tags
const posts = await fetch('https://api.example.com/posts', {
  next: { tags: ['posts'] },
});

// In server action
import { revalidateTag } from 'next/cache';

export async function createPost() {
  await db.post.create({ ... });
  revalidateTag('posts'); // Bust cache
}
```

## Database Queries (ORM)

```tsx
// Direct database access in Server Components
import { db } from '@/lib/db';

async function UsersPage() {
  const users = await db.user.findMany({
    include: { posts: true },
  });

  return <UserList users={users} />;
}
```

## Client-Side Fetching (When Needed)

```tsx
'use client';

import useSWR from 'swr';

const fetcher = (url: string) => fetch(url).then(r => r.json());

function LiveData() {
  const { data, error, isLoading } = useSWR('/api/live', fetcher, {
    refreshInterval: 1000, // Poll every second
  });

  if (isLoading) return <p>Loading...</p>;
  if (error) return <p>Error!</p>;
  return <p>Live: {data.value}</p>;
}
```

---

📅 *Learned: December 20, 2025*
🏷️ *Tags: Next.js, Data Fetching, Server Components, SWR*
