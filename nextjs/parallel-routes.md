# Parallel Routes for Complex Layouts

<div align="center">

![Next.js](https://img.shields.io/badge/Next.js-000000?style=for-the-badge&logo=next.js&logoColor=white)
![Routing](https://img.shields.io/badge/Routing-Parallel_Routes-blue?style=for-the-badge)

*Parallel routes render multiple pages in the same layout simultaneously.*

</div>

## Directory Structure

```
app/
├── layout.tsx
├── page.tsx
├── @dashboard/
│   └── page.tsx
├── @analytics/
│   └── page.tsx
└── @notifications/
    └── page.tsx
```

## Layout with Parallel Routes

```tsx
// app/layout.tsx
export default function Layout({
  children,
  dashboard,
  analytics,
  notifications,
}: {
  children: React.ReactNode;
  dashboard: React.ReactNode;
  analytics: React.ReactNode;
  notifications: React.ReactNode;
}) {
  return (
    <div className="grid grid-cols-3 gap-4">
      <div>{dashboard}</div>
      <div>{analytics}</div>
      <div>{notifications}</div>
    </div>
  );
}
```

## Conditional Rendering

```tsx
// app/layout.tsx
import { getUser } from '@/lib/auth';

export default async function Layout({
  children,
  admin,
  user,
}) {
  const currentUser = await getUser();

  return (
    <>
      {currentUser.role === 'admin' ? admin : user}
      {children}
    </>
  );
}
```

## Modal with Parallel Routes

```
app/
├── @modal/
│   ├── default.tsx      # Shows nothing normally
│   └── (.)photo/[id]/
│       └── page.tsx     # Intercepts /photo/[id]
├── photo/[id]/
│   └── page.tsx         # Full page view
└── layout.tsx
```

```tsx
// app/layout.tsx
export default function Layout({ children, modal }) {
  return (
    <>
      {children}
      {modal}
    </>
  );
}

// app/@modal/default.tsx
export default function Default() {
  return null; // No modal by default
}
```

## Use Cases

- Dashboards with multiple panels
- Modals that preserve background
- Split views (email client style)
- Conditional content by role

---

*Learned: December 20, 2025*
*Tags: Next.js, Parallel Routes, Layouts*
