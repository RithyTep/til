# Turbopack - Next.js Rust-Powered Bundler

<div align="center">

![Next.js](https://img.shields.io/badge/Next.js-000000?style=for-the-badge&logo=next.js&logoColor=white)
![Turbopack](https://img.shields.io/badge/Turbopack-000000?style=for-the-badge&logo=vercel&logoColor=white)
![Rust](https://img.shields.io/badge/Rust-DEA584?style=for-the-badge&logo=rust&logoColor=black)

*Up to 10x faster development builds with Turbopack, now the default in Next.js 16.*

![Turbopack](https://upload.wikimedia.org/wikipedia/commons/5/5e/Vercel_logo_black.svg)

</div>

## Turbopack Timeline

```
Next.js 15.0 (Oct 2024)    Turbopack Dev stable
Next.js 15.3 (Apr 2025)    Turbopack Build alpha
Next.js 15.5 (Aug 2025)    Turbopack Build beta
Next.js 16.0 (Oct 2025)    Turbopack default for all
```

## Performance Gains

```
Local Server Startup:    76.7% faster
Fast Refresh:            96.3% faster
Initial Route Compile:   45.8% faster
Production Builds:       2-5x faster
```

## Enable Turbopack

```bash
# Development (Next.js 15+)
next dev --turbo

# Production build (Next.js 15.3+)
next build --turbopack

# Next.js 16+ (default - no flag needed)
next dev
next build
```

## Configuration

```typescript
// next.config.ts
import type { NextConfig } from 'next'

const config: NextConfig = {
  // Turbopack specific options
  turbopack: {
    // Custom webpack loaders (if needed)
    rules: {
      '*.svg': {
        loaders: ['@svgr/webpack'],
        as: '*.js',
      },
    },

    // Resolve aliases
    resolveAlias: {
      '@components': './src/components',
      '@utils': './src/utils',
    },

    // Module resolution extensions
    resolveExtensions: ['.tsx', '.ts', '.jsx', '.js', '.json'],
  },
}

export default config
```

## What's Different in Next.js 16

### Turbopack as Default

```bash
# No --turbo flag needed anymore
next dev    # Uses Turbopack automatically
next build  # Uses Turbopack automatically

# Opt-out if needed (not recommended)
next dev --no-turbopack
```

### Node.js Middleware Runtime

```typescript
// middleware.ts
// Now supports Node.js runtime (not just Edge)
import { NextResponse } from 'next/server'
import type { NextRequest } from 'next/server'

export const config = {
  runtime: 'nodejs', // New in 15.5+
}

export function middleware(request: NextRequest) {
  // Can now use Node.js APIs
  return NextResponse.next()
}
```

### Typed Routes (Stable)

```typescript
// Type-safe links - compiler validates paths
import Link from 'next/link'

// Valid route
<Link href="/posts/123">Post</Link>

// TypeScript error if route doesn't exist
<Link href="/invalid-route">Bad</Link> // Error!
```

### Type Generation

```bash
# Generate types without full build
next typegen
```

```typescript
// Auto-generated types for routes
import type { PageProps, LayoutProps } from 'next'

export default function Page({ params, searchParams }: PageProps) {
  // params and searchParams fully typed
  return <div>{params.id}</div>
}
```

## Turbopack vs Webpack Comparison

| Feature | Webpack | Turbopack |
|---------|---------|-----------|
| Language | JavaScript | Rust |
| Cold Start | Slower | 76% faster |
| HMR Speed | Good | 96% faster |
| Memory Usage | Higher | 30% lower |
| Incremental | File-level | Function-level |

## Migration from Webpack

```typescript
// Most configs work automatically
// next.config.ts

const config: NextConfig = {
  // Webpack config still works
  webpack: (config) => {
    // Falls back to webpack if Turbopack doesn't support
    return config
  },

  // Turbopack-specific overrides
  turbopack: {
    rules: {
      // Custom loaders
    },
  },
}
```

## Supported Features

### Fully Supported
- TypeScript/JavaScript compilation
- CSS/CSS Modules/Tailwind CSS
- Image optimization
- Server Components/Actions
- App Router/Pages Router
- API Routes
- Middleware
- i18n

### Migration Notes

```typescript
// Custom webpack loaders need Turbopack equivalents
// next.config.ts
const config: NextConfig = {
  turbopack: {
    rules: {
      // SVG as React component
      '*.svg': {
        loaders: ['@svgr/webpack'],
        as: '*.js',
      },
      // Raw file imports
      '*.txt': {
        loaders: ['raw-loader'],
      },
    },
  },
}
```

## Debugging Turbopack

```bash
# Verbose output
TURBOPACK_DEBUG=1 next dev --turbo

# Check bundle analysis
next build --turbopack --debug
```

## Best Practices

```typescript
// 1. Use dynamic imports for code splitting
const HeavyComponent = dynamic(() => import('./HeavyComponent'))

// 2. Leverage parallel routes
// app/@modal/page.tsx - loads in parallel

// 3. Use route groups for organization
// app/(marketing)/page.tsx
// app/(dashboard)/page.tsx

// 4. Streaming with Suspense
<Suspense fallback={<Loading />}>
  <SlowComponent />
</Suspense>
```

---

*Learned: December 20, 2025*
*Tags: Next.js, Turbopack, Bundler, Rust, Performance*
