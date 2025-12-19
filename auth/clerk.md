# Clerk - Modern Authentication for React

<div align="center">

![Clerk](https://img.shields.io/badge/Clerk-6C47FF?style=for-the-badge&logo=clerk&logoColor=white)
![Auth](https://img.shields.io/badge/Auth-Authentication-blue?style=for-the-badge)
![Next.js](https://img.shields.io/badge/Next.js-000000?style=for-the-badge&logo=next.js&logoColor=white)

*Drop-in authentication with beautiful UI, social logins, and organization management.*

</div>

## Why Clerk?

```
Auth.js / NextAuth              Clerk
──────────────────────────     ──────────────────────────
DIY UI components              Pre-built, customizable UI
Manual session handling        Automatic session sync
Basic user management          Full user management UI
No organization support        Multi-tenant built-in
Self-hosted only               Managed + self-hosted
```

## Installation

```bash
npm install @clerk/nextjs
```

## Setup

```typescript
// middleware.ts
import { clerkMiddleware } from '@clerk/nextjs/server'

export default clerkMiddleware()

export const config = {
  matcher: [
    '/((?!_next|[^?]*\\.(?:html?|css|js(?!on)|jpe?g|webp|png|gif|svg|ttf|woff2?|ico|csv|docx?|xlsx?|zip|webmanifest)).*)',
    '/(api|trpc)(.*)',
  ],
}
```

```typescript
// app/layout.tsx
import { ClerkProvider } from '@clerk/nextjs'

export default function RootLayout({
  children,
}: {
  children: React.ReactNode
}) {
  return (
    <ClerkProvider>
      <html lang="en">
        <body>{children}</body>
      </html>
    </ClerkProvider>
  )
}
```

```env
# .env.local
NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=pk_...
CLERK_SECRET_KEY=sk_...
```

## Pre-Built Components

```typescript
import {
  SignIn,
  SignUp,
  SignInButton,
  SignUpButton,
  SignOutButton,
  UserButton,
  UserProfile,
} from '@clerk/nextjs'

// Sign In Page - app/sign-in/[[...sign-in]]/page.tsx
export default function SignInPage() {
  return <SignIn />
}

// Sign Up Page - app/sign-up/[[...sign-up]]/page.tsx
export default function SignUpPage() {
  return <SignUp />
}

// Header with auth buttons
function Header() {
  return (
    <header className="flex justify-between p-4">
      <Logo />
      <div className="flex gap-4">
        <SignedOut>
          <SignInButton mode="modal" />
          <SignUpButton mode="modal" />
        </SignedOut>
        <SignedIn>
          <UserButton afterSignOutUrl="/" />
        </SignedIn>
      </div>
    </header>
  )
}
```

## Conditional Rendering

```typescript
import { SignedIn, SignedOut, RedirectToSignIn } from '@clerk/nextjs'

// Show different content based on auth state
function Page() {
  return (
    <>
      <SignedIn>
        <Dashboard />
      </SignedIn>
      <SignedOut>
        <LandingPage />
      </SignedOut>
    </>
  )
}

// Require sign in
function ProtectedPage() {
  return (
    <>
      <SignedIn>
        <SecretContent />
      </SignedIn>
      <SignedOut>
        <RedirectToSignIn />
      </SignedOut>
    </>
  )
}
```

## Protect Routes (Server)

```typescript
// app/dashboard/page.tsx
import { auth } from '@clerk/nextjs/server'
import { redirect } from 'next/navigation'

export default async function DashboardPage() {
  const { userId } = await auth()

  if (!userId) {
    redirect('/sign-in')
  }

  return <Dashboard userId={userId} />
}
```

## Get User Data

```typescript
// Server Component
import { currentUser, auth } from '@clerk/nextjs/server'

export default async function ProfilePage() {
  const user = await currentUser()

  if (!user) return null

  return (
    <div>
      <h1>{user.firstName} {user.lastName}</h1>
      <p>{user.emailAddresses[0].emailAddress}</p>
      <img src={user.imageUrl} alt="Avatar" />
    </div>
  )
}

// Client Component
'use client'
import { useUser, useAuth } from '@clerk/nextjs'

function Profile() {
  const { user, isLoaded, isSignedIn } = useUser()
  const { userId, sessionId, getToken } = useAuth()

  if (!isLoaded) return <Spinner />
  if (!isSignedIn) return <SignIn />

  return <div>Hello, {user.firstName}</div>
}
```

## API Route Protection

```typescript
// app/api/protected/route.ts
import { auth } from '@clerk/nextjs/server'
import { NextResponse } from 'next/server'

export async function GET() {
  const { userId } = await auth()

  if (!userId) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })
  }

  // Fetch user-specific data
  const data = await fetchUserData(userId)

  return NextResponse.json(data)
}
```

## Organizations (Multi-Tenant)

```typescript
import {
  OrganizationSwitcher,
  OrganizationProfile,
  CreateOrganization,
} from '@clerk/nextjs'
import { auth } from '@clerk/nextjs/server'

// Organization switcher in header
function Header() {
  return (
    <header>
      <OrganizationSwitcher />
    </header>
  )
}

// Get current organization
export default async function OrgDashboard() {
  const { userId, orgId, orgRole } = await auth()

  if (!orgId) {
    return <CreateOrganization />
  }

  return (
    <div>
      <p>Organization: {orgId}</p>
      <p>Role: {orgRole}</p>
    </div>
  )
}

// Protect by role
export default async function AdminPage() {
  const { orgRole } = await auth()

  if (orgRole !== 'org:admin') {
    return <p>Admin access required</p>
  }

  return <AdminDashboard />
}
```

## Custom Claims & Metadata

```typescript
// Set public metadata (from backend)
import { clerkClient } from '@clerk/nextjs/server'

await clerkClient.users.updateUser(userId, {
  publicMetadata: {
    role: 'admin',
    plan: 'pro',
  },
})

// Access in components
const { user } = useUser()
const role = user?.publicMetadata?.role

// Access in server
const { sessionClaims } = await auth()
const role = sessionClaims?.metadata?.role
```

## Webhooks

```typescript
// app/api/webhooks/clerk/route.ts
import { Webhook } from 'svix'
import { headers } from 'next/headers'
import { WebhookEvent } from '@clerk/nextjs/server'

export async function POST(req: Request) {
  const WEBHOOK_SECRET = process.env.CLERK_WEBHOOK_SECRET!

  const headerPayload = headers()
  const svix_id = headerPayload.get('svix-id')
  const svix_timestamp = headerPayload.get('svix-timestamp')
  const svix_signature = headerPayload.get('svix-signature')

  const payload = await req.json()
  const body = JSON.stringify(payload)

  const wh = new Webhook(WEBHOOK_SECRET)
  let evt: WebhookEvent

  try {
    evt = wh.verify(body, {
      'svix-id': svix_id!,
      'svix-timestamp': svix_timestamp!,
      'svix-signature': svix_signature!,
    }) as WebhookEvent
  } catch (err) {
    return new Response('Invalid signature', { status: 400 })
  }

  switch (evt.type) {
    case 'user.created':
      await createUserInDB(evt.data)
      break
    case 'user.updated':
      await updateUserInDB(evt.data)
      break
    case 'user.deleted':
      await deleteUserFromDB(evt.data.id)
      break
  }

  return new Response('OK', { status: 200 })
}
```

## Customization

```typescript
// Customize appearance
<ClerkProvider
  appearance={{
    baseTheme: dark,
    variables: {
      colorPrimary: '#6366f1',
      borderRadius: '0.5rem',
    },
    elements: {
      card: 'shadow-xl',
      formButtonPrimary: 'bg-indigo-600 hover:bg-indigo-700',
    },
  }}
>
```

---

*Learned: December 20, 2025*
*Tags: Clerk, Authentication, Next.js, React, Auth*
