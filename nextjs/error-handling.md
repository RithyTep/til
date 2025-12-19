# Error Handling in Next.js

<div align="center">

![Next.js](https://img.shields.io/badge/Next.js-000000?style=for-the-badge&logo=next.js&logoColor=white)
![Errors](https://img.shields.io/badge/UX-Error_Handling-red?style=for-the-badge)

*Handle errors gracefully with error boundaries and not-found pages.*

</div>

## error.tsx (Error Boundary)

```tsx
// app/dashboard/error.tsx
'use client'; // Must be a Client Component

import { useEffect } from 'react';

export default function Error({
  error,
  reset,
}: {
  error: Error & { digest?: string };
  reset: () => void;
}) {
  useEffect(() => {
    // Log error to reporting service
    console.error(error);
  }, [error]);

  return (
    <div className="error-container">
      <h2>Something went wrong!</h2>
      <p>{error.message}</p>
      <button onClick={reset}>Try again</button>
    </div>
  );
}
```

## global-error.tsx (Root Error)

```tsx
// app/global-error.tsx
'use client';

export default function GlobalError({
  error,
  reset,
}: {
  error: Error;
  reset: () => void;
}) {
  return (
    <html>
      <body>
        <h2>Something went wrong!</h2>
        <button onClick={reset}>Try again</button>
      </body>
    </html>
  );
}
```

## not-found.tsx

```tsx
// app/not-found.tsx
import Link from 'next/link';

export default function NotFound() {
  return (
    <div>
      <h2>404 - Page Not Found</h2>
      <p>Could not find the requested resource</p>
      <Link href="/">Return Home</Link>
    </div>
  );
}

// Trigger programmatically
import { notFound } from 'next/navigation';

async function Page({ params }) {
  const post = await getPost(params.id);

  if (!post) {
    notFound(); // Shows not-found.tsx
  }

  return <article>{post.content}</article>;
}
```

## Route Segment Not Found

```tsx
// app/posts/[id]/not-found.tsx
export default function PostNotFound() {
  return (
    <div>
      <h2>Post Not Found</h2>
      <p>The post you're looking for doesn't exist.</p>
    </div>
  );
}
```

## API Route Error Handling

```tsx
// app/api/users/route.ts
import { NextResponse } from 'next/server';

export async function GET() {
  try {
    const users = await db.user.findMany();
    return NextResponse.json(users);
  } catch (error) {
    console.error('Database error:', error);
    return NextResponse.json(
      { error: 'Internal Server Error' },
      { status: 500 }
    );
  }
}
```

## Error Hierarchy

```
app/
├── global-error.tsx    # Catches root layout errors
├── error.tsx           # Catches app-level errors
├── dashboard/
│   ├── error.tsx       # Catches dashboard errors
│   └── settings/
│       └── error.tsx   # Catches settings errors
```

---

*Learned: December 20, 2025*
*Tags: Next.js, Error Handling, Error Boundaries*
