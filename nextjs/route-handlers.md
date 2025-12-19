# Route Handlers (API Routes)

Create API endpoints using the App Router's route handlers.

## Basic Route Handler

```tsx
// app/api/users/route.ts
import { NextResponse } from 'next/server';

export async function GET() {
  const users = await db.user.findMany();
  return NextResponse.json(users);
}

export async function POST(request: Request) {
  const body = await request.json();
  const user = await db.user.create({ data: body });
  return NextResponse.json(user, { status: 201 });
}
```

## Dynamic Routes

```tsx
// app/api/users/[id]/route.ts
export async function GET(
  request: Request,
  { params }: { params: { id: string } }
) {
  const user = await db.user.findUnique({
    where: { id: params.id }
  });

  if (!user) {
    return NextResponse.json(
      { error: 'User not found' },
      { status: 404 }
    );
  }

  return NextResponse.json(user);
}

export async function DELETE(
  request: Request,
  { params }: { params: { id: string } }
) {
  await db.user.delete({ where: { id: params.id } });
  return new NextResponse(null, { status: 204 });
}
```

## Query Parameters

```tsx
// GET /api/search?q=hello&page=1
export async function GET(request: Request) {
  const { searchParams } = new URL(request.url);
  const query = searchParams.get('q');
  const page = parseInt(searchParams.get('page') || '1');

  const results = await search(query, page);
  return NextResponse.json(results);
}
```

## Headers & Cookies

```tsx
import { cookies, headers } from 'next/headers';

export async function GET() {
  // Read headers
  const headersList = headers();
  const auth = headersList.get('authorization');

  // Read cookies
  const cookieStore = cookies();
  const token = cookieStore.get('token');

  // Set cookies in response
  const response = NextResponse.json({ success: true });
  response.cookies.set('session', 'abc123', {
    httpOnly: true,
    secure: true,
    maxAge: 60 * 60 * 24, // 1 day
  });

  return response;
}
```

## Streaming Response

```tsx
export async function GET() {
  const encoder = new TextEncoder();
  const stream = new ReadableStream({
    async start(controller) {
      for (let i = 0; i < 10; i++) {
        controller.enqueue(encoder.encode(`data: ${i}\n\n`));
        await new Promise(r => setTimeout(r, 1000));
      }
      controller.close();
    },
  });

  return new Response(stream, {
    headers: { 'Content-Type': 'text/event-stream' },
  });
}
```

---

📅 *Learned: December 20, 2025*
🏷️ *Tags: Next.js, API Routes, Route Handlers*
