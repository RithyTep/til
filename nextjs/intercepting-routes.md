# Intercepting Routes for Modals

<div align="center">

![Next.js](https://img.shields.io/badge/Next.js-000000?style=for-the-badge&logo=next.js&logoColor=white)
![Routing](https://img.shields.io/badge/Routing-Intercepting_Routes-purple?style=for-the-badge)

*Intercepting routes show a route as a modal while keeping the URL updated.*

</div>

## Convention

```
(.)  - Same level
(..) - One level up
(..)(..) - Two levels up
(...) - Root level
```

## Photo Modal Example

```
app/
├── feed/
│   └── page.tsx           # Feed with photo links
├── photo/[id]/
│   └── page.tsx           # Full photo page (direct access)
├── @modal/
│   ├── default.tsx        # Empty by default
│   └── (.)photo/[id]/
│       └── page.tsx       # Photo as modal (intercepted)
└── layout.tsx
```

## Feed Page with Links

```tsx
// app/feed/page.tsx
import Link from 'next/link';

export default function Feed() {
  const photos = [1, 2, 3, 4, 5];

  return (
    <div className="grid grid-cols-3 gap-4">
      {photos.map(id => (
        <Link key={id} href={`/photo/${id}`}>
          <img src={`/photos/${id}.jpg`} />
        </Link>
      ))}
    </div>
  );
}
```

## Intercepted Modal

```tsx
// app/@modal/(.)photo/[id]/page.tsx
import { Modal } from '@/components/Modal';

export default function PhotoModal({ params }) {
  return (
    <Modal>
      <img src={`/photos/${params.id}.jpg`} />
      <p>Photo {params.id}</p>
    </Modal>
  );
}
```

## Full Page (Direct Access)

```tsx
// app/photo/[id]/page.tsx
export default function PhotoPage({ params }) {
  return (
    <div className="full-screen">
      <img src={`/photos/${params.id}.jpg`} />
      <h1>Photo {params.id}</h1>
      <p>Full details here...</p>
    </div>
  );
}
```

## Behavior

- **Click from feed** → Modal opens, URL changes to `/photo/1`
- **Direct URL access** → Full page loads
- **Refresh on modal** → Full page loads
- **Back button** → Modal closes, back to feed

---

*Learned: December 20, 2025*
*Tags: Next.js, Intercepting Routes, Modals*
