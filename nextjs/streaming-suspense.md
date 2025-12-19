# Streaming with Suspense

Stream UI from server to client progressively using Suspense boundaries.

## Basic Streaming

```tsx
// app/page.tsx
import { Suspense } from 'react';

async function SlowComponent() {
  const data = await fetch('https://api.example.com/slow');
  return <div>{data.title}</div>;
}

export default function Page() {
  return (
    <div>
      <h1>Instant Header</h1>

      <Suspense fallback={<p>Loading data...</p>}>
        <SlowComponent />
      </Suspense>
    </div>
  );
}
```

## Multiple Suspense Boundaries

```tsx
export default function Dashboard() {
  return (
    <div className="grid grid-cols-3 gap-4">
      <Suspense fallback={<CardSkeleton />}>
        <RevenueCard />
      </Suspense>

      <Suspense fallback={<CardSkeleton />}>
        <UsersCard />
      </Suspense>

      <Suspense fallback={<CardSkeleton />}>
        <OrdersCard />
      </Suspense>
    </div>
  );
}
```

## Loading.tsx (Route-Level Streaming)

```tsx
// app/dashboard/loading.tsx
export default function Loading() {
  return <DashboardSkeleton />;
}

// Automatically wraps page.tsx in Suspense!
```

## Streaming with Dynamic Data

```tsx
async function Comments({ postId }) {
  // This fetch is streamed in after initial page load
  const comments = await db.comment.findMany({
    where: { postId }
  });

  return (
    <ul>
      {comments.map(c => <li key={c.id}>{c.text}</li>)}
    </ul>
  );
}

export default function PostPage({ params }) {
  return (
    <article>
      {/* Instant */}
      <h1>Post Title</h1>
      <p>Post content...</p>

      {/* Streamed later */}
      <Suspense fallback={<p>Loading comments...</p>}>
        <Comments postId={params.id} />
      </Suspense>
    </article>
  );
}
```

## Benefits

- ✅ Faster Time to First Byte (TTFB)
- ✅ Progressive rendering
- ✅ No all-or-nothing loading
- ✅ Better perceived performance

---

📅 *Learned: 2024*
🏷️ *Tags: Next.js, Streaming, Suspense, Performance*
