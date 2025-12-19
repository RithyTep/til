# View Transitions API - Smooth Page Animations

<div align="center">

![CSS](https://img.shields.io/badge/CSS-View_Transitions-264de4?style=for-the-badge&logo=css3&logoColor=white)
![Animation](https://img.shields.io/badge/Animation-Native-green?style=for-the-badge)
![Browser](https://img.shields.io/badge/Chrome-Supported-blue?style=for-the-badge)

*Native browser API for smooth transitions between page states and navigations.*

</div>

## What Are View Transitions?

```
Traditional SPA                View Transitions
──────────────────────────     ──────────────────────────
Abrupt content change          Smooth crossfade
Custom animation libs          Native browser API
Complex state management       Simple API
Performance overhead           GPU-accelerated
```

## Basic Same-Document Transition

```typescript
// Wrap DOM changes in startViewTransition
function updateContent() {
  document.startViewTransition(() => {
    // DOM changes happen here
    document.getElementById('content').innerHTML = newContent
  })
}

// With async operations
async function loadNewPage() {
  const transition = document.startViewTransition(async () => {
    const html = await fetchPage('/new-page')
    document.body.innerHTML = html
  })

  // Wait for transition to complete
  await transition.finished
}
```

## CSS Customization

```css
/* Default transition (crossfade) */
::view-transition-old(root),
::view-transition-new(root) {
  animation-duration: 0.3s;
}

/* Custom animation for old view */
::view-transition-old(root) {
  animation: slide-out 0.3s ease-in forwards;
}

/* Custom animation for new view */
::view-transition-new(root) {
  animation: slide-in 0.3s ease-out forwards;
}

@keyframes slide-out {
  to {
    transform: translateX(-100%);
    opacity: 0;
  }
}

@keyframes slide-in {
  from {
    transform: translateX(100%);
    opacity: 0;
  }
}
```

## Named Transitions

```html
<!-- Give elements unique transition names -->
<header style="view-transition-name: header;">
  Site Header
</header>

<main style="view-transition-name: main-content;">
  Page Content
</main>

<img
  src="hero.jpg"
  style="view-transition-name: hero-image;"
/>
```

```css
/* Animate specific elements differently */
::view-transition-old(header),
::view-transition-new(header) {
  animation: none; /* Header stays static */
}

::view-transition-old(main-content) {
  animation: fade-out 0.2s ease-out;
}

::view-transition-new(main-content) {
  animation: fade-in 0.3s ease-in;
}

::view-transition-old(hero-image),
::view-transition-new(hero-image) {
  animation-duration: 0.5s;
  /* Browser morphs between old and new positions */
}
```

## Next.js App Router

```typescript
// app/layout.tsx
export default function RootLayout({ children }) {
  return (
    <html>
      <head>
        <meta name="view-transition" content="same-origin" />
      </head>
      <body>{children}</body>
    </html>
  )
}
```

```css
/* app/globals.css */
@view-transition {
  navigation: auto;
}

::view-transition-old(root) {
  animation: fade-out 0.2s ease-out;
}

::view-transition-new(root) {
  animation: fade-in 0.3s ease-in;
}
```

## React Implementation

```typescript
'use client'
import { useRouter } from 'next/navigation'

export function TransitionLink({ href, children }) {
  const router = useRouter()

  const handleClick = async (e: React.MouseEvent) => {
    e.preventDefault()

    // Check for browser support
    if (!document.startViewTransition) {
      router.push(href)
      return
    }

    document.startViewTransition(() => {
      router.push(href)
    })
  }

  return (
    <a href={href} onClick={handleClick}>
      {children}
    </a>
  )
}
```

## Shared Element Transitions

```typescript
// Product list page
function ProductCard({ product }) {
  return (
    <Link href={`/products/${product.id}`}>
      <img
        src={product.image}
        style={{ viewTransitionName: `product-${product.id}` }}
      />
      <h2 style={{ viewTransitionName: `title-${product.id}` }}>
        {product.name}
      </h2>
    </Link>
  )
}

// Product detail page
function ProductDetail({ product }) {
  return (
    <div>
      <img
        src={product.image}
        style={{ viewTransitionName: `product-${product.id}` }}
      />
      <h1 style={{ viewTransitionName: `title-${product.id}` }}>
        {product.name}
      </h1>
    </div>
  )
}
```

```css
/* Shared elements morph automatically */
::view-transition-group(product-*) {
  animation-duration: 0.4s;
  animation-timing-function: ease-in-out;
}
```

## Direction-Based Transitions

```typescript
// Track navigation direction
let navigationType = 'forward'

window.addEventListener('popstate', () => {
  navigationType = 'back'
})

function navigate(url: string) {
  navigationType = 'forward'

  document.startViewTransition(() => {
    document.documentElement.dataset.transition = navigationType
    // Navigate...
  })
}
```

```css
/* Forward navigation */
[data-transition="forward"]::view-transition-old(root) {
  animation: slide-to-left 0.3s ease-out;
}
[data-transition="forward"]::view-transition-new(root) {
  animation: slide-from-right 0.3s ease-out;
}

/* Back navigation */
[data-transition="back"]::view-transition-old(root) {
  animation: slide-to-right 0.3s ease-out;
}
[data-transition="back"]::view-transition-new(root) {
  animation: slide-from-left 0.3s ease-out;
}
```

## Reduce Motion Support

```css
@media (prefers-reduced-motion: reduce) {
  ::view-transition-old(root),
  ::view-transition-new(root) {
    animation: none;
  }

  ::view-transition-group(*) {
    animation-duration: 0.01ms;
  }
}
```

## Feature Detection

```typescript
function supportsViewTransitions(): boolean {
  return 'startViewTransition' in document
}

// Polyfill-like fallback
async function transitionTo(updateFn: () => void | Promise<void>) {
  if (supportsViewTransitions()) {
    await document.startViewTransition(updateFn).finished
  } else {
    await updateFn()
  }
}
```

## Multi-Page Applications (MPA)

```html
<!-- Enable for cross-document navigations -->
<head>
  <meta name="view-transition" content="same-origin" />
</head>
```

```css
/* CSS triggers transitions for page navigations */
@view-transition {
  navigation: auto;
}
```

## Advanced: Custom Transition Classes

```typescript
const transition = document.startViewTransition(() => {
  // DOM changes
})

// Add class during transition
transition.ready.then(() => {
  document.documentElement.classList.add('transitioning')
})

transition.finished.then(() => {
  document.documentElement.classList.remove('transitioning')
})
```

## Browser Support

| Browser | Support |
|---------|---------|
| Chrome 111+ | Full |
| Edge 111+ | Full |
| Safari 18+ | Full |
| Firefox | In development |

---

*Learned: December 20, 2025*
*Tags: View Transitions, CSS, Animation, SPA, Navigation*
