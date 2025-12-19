# Middleware for Edge Computing

<div align="center">

![Next.js](https://img.shields.io/badge/Next.js-000000?style=for-the-badge&logo=next.js&logoColor=white)
![Edge](https://img.shields.io/badge/Edge-Middleware-orange?style=for-the-badge&logo=vercel&logoColor=white)

*Middleware runs before every request at the Edge, perfect for auth and redirects.*

</div>

## Basic Middleware

```tsx
// middleware.ts (root of project)
import { NextResponse } from 'next/server';
import type { NextRequest } from 'next/server';

export function middleware(request: NextRequest) {
  // Check auth
  const token = request.cookies.get('token');

  if (!token && request.nextUrl.pathname.startsWith('/dashboard')) {
    return NextResponse.redirect(new URL('/login', request.url));
  }

  return NextResponse.next();
}
```

## Matcher Config

```tsx
// Only run on specific paths
export const config = {
  matcher: [
    '/dashboard/:path*',
    '/api/:path*',
    '/((?!_next/static|favicon.ico).*)',
  ],
};
```

## Geo-based Redirects

```tsx
export function middleware(request: NextRequest) {
  const country = request.geo?.country || 'US';

  if (country === 'KH') {
    return NextResponse.redirect(new URL('/km', request.url));
  }

  return NextResponse.next();
}
```

## Adding Headers

```tsx
export function middleware(request: NextRequest) {
  const response = NextResponse.next();

  // Add security headers
  response.headers.set('X-Frame-Options', 'DENY');
  response.headers.set('X-Content-Type-Options', 'nosniff');

  // Add request ID
  response.headers.set('X-Request-Id', crypto.randomUUID());

  return response;
}
```

## Rewriting URLs

```tsx
export function middleware(request: NextRequest) {
  const hostname = request.headers.get('host');

  // Multi-tenant rewrite
  if (hostname === 'acme.example.com') {
    return NextResponse.rewrite(
      new URL('/tenants/acme' + request.nextUrl.pathname, request.url)
    );
  }

  return NextResponse.next();
}
```

## A/B Testing

```tsx
export function middleware(request: NextRequest) {
  const bucket = request.cookies.get('ab-bucket')?.value
    || (Math.random() < 0.5 ? 'a' : 'b');

  const response = NextResponse.rewrite(
    new URL(`/experiments/${bucket}${request.nextUrl.pathname}`, request.url)
  );

  response.cookies.set('ab-bucket', bucket);
  return response;
}
```

---

*Learned: December 20, 2025*
*Tags: Next.js, Middleware, Edge, Authentication*
