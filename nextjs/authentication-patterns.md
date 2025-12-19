# Authentication Patterns in Next.js

<div align="center">

![Next.js](https://img.shields.io/badge/Next.js-000000?style=for-the-badge&logo=next.js&logoColor=white)
![Auth](https://img.shields.io/badge/Security-Authentication-red?style=for-the-badge&logo=auth0&logoColor=white)

*Common patterns for auth in App Router applications.*

</div>

## Middleware Authentication

```tsx
// middleware.ts
import { NextResponse } from 'next/server';
import type { NextRequest } from 'next/server';

export function middleware(request: NextRequest) {
  const token = request.cookies.get('session')?.value;

  // Protect dashboard routes
  if (request.nextUrl.pathname.startsWith('/dashboard')) {
    if (!token) {
      return NextResponse.redirect(new URL('/login', request.url));
    }
  }

  // Redirect logged-in users from login page
  if (request.nextUrl.pathname === '/login' && token) {
    return NextResponse.redirect(new URL('/dashboard', request.url));
  }

  return NextResponse.next();
}

export const config = {
  matcher: ['/dashboard/:path*', '/login'],
};
```

## Server Component Auth Check

```tsx
// lib/auth.ts
import { cookies } from 'next/headers';
import { redirect } from 'next/navigation';

export async function getUser() {
  const token = cookies().get('session')?.value;

  if (!token) {
    return null;
  }

  const user = await verifyToken(token);
  return user;
}

export async function requireAuth() {
  const user = await getUser();

  if (!user) {
    redirect('/login');
  }

  return user;
}
```

## Protected Page

```tsx
// app/dashboard/page.tsx
import { requireAuth } from '@/lib/auth';

export default async function DashboardPage() {
  const user = await requireAuth(); // Redirects if not authenticated

  return (
    <div>
      <h1>Welcome, {user.name}!</h1>
    </div>
  );
}
```

## Login with Server Action

```tsx
// app/login/actions.ts
'use server';

import { cookies } from 'next/headers';
import { redirect } from 'next/navigation';

export async function login(formData: FormData) {
  const email = formData.get('email') as string;
  const password = formData.get('password') as string;

  const user = await authenticate(email, password);

  if (!user) {
    return { error: 'Invalid credentials' };
  }

  const session = await createSession(user.id);

  cookies().set('session', session, {
    httpOnly: true,
    secure: process.env.NODE_ENV === 'production',
    maxAge: 60 * 60 * 24 * 7, // 1 week
    path: '/',
  });

  redirect('/dashboard');
}

export async function logout() {
  cookies().delete('session');
  redirect('/login');
}
```

## Login Form

```tsx
// app/login/page.tsx
import { login } from './actions';

export default function LoginPage() {
  return (
    <form action={login}>
      <input name="email" type="email" required />
      <input name="password" type="password" required />
      <button type="submit">Login</button>
    </form>
  );
}
```

## Client-Side Auth Context

```tsx
// context/AuthContext.tsx
'use client';

import { createContext, useContext } from 'react';

const AuthContext = createContext<User | null>(null);

export function AuthProvider({ user, children }) {
  return (
    <AuthContext.Provider value={user}>
      {children}
    </AuthContext.Provider>
  );
}

export const useAuth = () => useContext(AuthContext);
```

---

📅 *Learned: December 20, 2025*
🏷️ *Tags: Next.js, Authentication, Security, Middleware*
