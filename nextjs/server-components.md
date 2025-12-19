# React Server Components in Next.js

Server Components run only on the server, reducing JavaScript sent to the client.

## Default Behavior (App Router)

```tsx
// app/page.tsx - This is a Server Component by default!
async function Page() {
  // Can fetch data directly - no useEffect needed
  const data = await fetch('https://api.example.com/data');
  const posts = await data.json();

  return (
    <ul>
      {posts.map(post => (
        <li key={post.id}>{post.title}</li>
      ))}
    </ul>
  );
}

export default Page;
```

## When to Use Server vs Client Components

| Server Components | Client Components |
|-------------------|-------------------|
| Fetch data | onClick, onChange handlers |
| Access backend resources | useState, useEffect |
| Keep secrets on server | Browser APIs (localStorage) |
| Large dependencies | Interactivity |

## Making a Client Component

```tsx
'use client'; // Add this directive at the top

import { useState } from 'react';

export function Counter() {
  const [count, setCount] = useState(0);

  return (
    <button onClick={() => setCount(c => c + 1)}>
      Count: {count}
    </button>
  );
}
```

## Mixing Server and Client

```tsx
// app/page.tsx (Server Component)
import { Counter } from './Counter'; // Client Component

async function Page() {
  const data = await fetchData(); // Server-side fetch

  return (
    <div>
      <h1>{data.title}</h1>      {/* Server rendered */}
      <Counter />                 {/* Client interactive */}
    </div>
  );
}
```

## Benefits

- ✅ Zero JavaScript for static content
- ✅ Direct database/API access
- ✅ Secrets stay on server
- ✅ Smaller bundle size
- ✅ Better SEO

---

📅 *Learned: 2024*
🏷️ *Tags: Next.js, React Server Components, App Router*
