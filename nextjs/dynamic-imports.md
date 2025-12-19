# Dynamic Imports for Code Splitting

Lazy load components to reduce initial bundle size.

## Basic Dynamic Import

```tsx
import dynamic from 'next/dynamic';

// Component loads only when rendered
const HeavyChart = dynamic(() => import('@/components/HeavyChart'));

export default function Dashboard() {
  return (
    <div>
      <h1>Dashboard</h1>
      <HeavyChart />
    </div>
  );
}
```

## With Loading State

```tsx
const HeavyEditor = dynamic(
  () => import('@/components/RichTextEditor'),
  {
    loading: () => <p>Loading editor...</p>,
  }
);
```

## Disable SSR

```tsx
// For components that use browser APIs
const MapComponent = dynamic(
  () => import('@/components/Map'),
  { ssr: false }
);

// Useful for:
// - window/document access
// - localStorage
// - Canvas/WebGL
// - Third-party libraries without SSR support
```

## Named Exports

```tsx
// If component is not default export
const SpecificComponent = dynamic(
  () => import('@/components/Library').then(mod => mod.SpecificComponent)
);
```

## Conditional Loading

```tsx
export default function Page({ showComments }) {
  const Comments = dynamic(() => import('@/components/Comments'));

  return (
    <article>
      <h1>Post Title</h1>
      <p>Content...</p>
      {showComments && <Comments />}
    </article>
  );
}
```

## With Suspense

```tsx
import { Suspense } from 'react';
import dynamic from 'next/dynamic';

const DynamicComponent = dynamic(
  () => import('@/components/Heavy'),
  { suspense: true }
);

export default function Page() {
  return (
    <Suspense fallback={<Loading />}>
      <DynamicComponent />
    </Suspense>
  );
}
```

## Import External Libraries

```tsx
// Only load library when needed
async function handleClick() {
  const { format } = await import('date-fns');
  alert(format(new Date(), 'yyyy-MM-dd'));
}

// Or for heavy libraries
const Confetti = dynamic(() => import('react-confetti'), {
  ssr: false,
});
```

---

📅 *Learned: 2024*
🏷️ *Tags: Next.js, Dynamic Import, Code Splitting, Performance*
