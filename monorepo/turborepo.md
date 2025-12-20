# Turborepo - High-Performance Monorepo

<div align="center">

![Turborepo](https://img.shields.io/badge/Turborepo-EF4444?style=for-the-badge&logo=turborepo&logoColor=white)
![Monorepo](https://img.shields.io/badge/Monorepo-Build_System-blue?style=for-the-badge)
![Vercel](https://img.shields.io/badge/Vercel-000000?style=for-the-badge&logo=vercel&logoColor=white)

*Incremental builds, remote caching, and parallel execution for monorepos.*

![Turborepo](https://turbo.build/images/docs/pack-benchmark.svg)

</div>

## Why Turborepo?

```
Without Turborepo              With Turborepo
──────────────────────────     ──────────────────────────
Build everything               Build only what changed
Sequential tasks               Parallel execution
No caching                     Smart local + remote cache
Slow CI                        10x faster CI
Manual dependencies            Automatic task ordering
```

## Create New Monorepo

```bash
npx create-turbo@latest my-monorepo
cd my-monorepo
```

## Project Structure

```
my-monorepo/
├── apps/
│   ├── web/               # Next.js app
│   │   └── package.json
│   ├── docs/              # Docs site
│   │   └── package.json
│   └── api/               # Backend API
│       └── package.json
├── packages/
│   ├── ui/                # Shared UI components
│   │   └── package.json
│   ├── config-eslint/     # Shared ESLint config
│   │   └── package.json
│   ├── config-typescript/ # Shared TS config
│   │   └── package.json
│   └── utils/             # Shared utilities
│       └── package.json
├── turbo.json
├── package.json
└── pnpm-workspace.yaml
```

## Root Configuration

```json
// package.json
{
  "name": "my-monorepo",
  "private": true,
  "scripts": {
    "build": "turbo build",
    "dev": "turbo dev",
    "lint": "turbo lint",
    "test": "turbo test",
    "clean": "turbo clean"
  },
  "devDependencies": {
    "turbo": "^2.0.0"
  },
  "packageManager": "pnpm@8.15.0"
}
```

```yaml
# pnpm-workspace.yaml
packages:
  - "apps/*"
  - "packages/*"
```

## Turbo Configuration

```json
// turbo.json
{
  "$schema": "https://turbo.build/schema.json",
  "globalDependencies": ["**/.env.*local"],
  "tasks": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": [".next/**", "!.next/cache/**", "dist/**"]
    },
    "dev": {
      "cache": false,
      "persistent": true
    },
    "lint": {
      "dependsOn": ["^build"]
    },
    "test": {
      "dependsOn": ["build"],
      "inputs": ["src/**/*.tsx", "src/**/*.ts", "test/**/*.ts"]
    },
    "clean": {
      "cache": false
    }
  }
}
```

## Package Dependencies

```json
// apps/web/package.json
{
  "name": "@repo/web",
  "dependencies": {
    "@repo/ui": "workspace:*",
    "@repo/utils": "workspace:*",
    "next": "^14.0.0",
    "react": "^18.0.0"
  },
  "devDependencies": {
    "@repo/config-eslint": "workspace:*",
    "@repo/config-typescript": "workspace:*"
  }
}
```

```json
// packages/ui/package.json
{
  "name": "@repo/ui",
  "main": "./dist/index.js",
  "types": "./dist/index.d.ts",
  "exports": {
    ".": "./dist/index.js",
    "./button": "./dist/button.js"
  },
  "scripts": {
    "build": "tsup src/index.ts --format cjs,esm --dts",
    "dev": "tsup src/index.ts --format cjs,esm --dts --watch"
  }
}
```

## CLI Commands

```bash
# Run all builds
turbo build

# Run specific task
turbo lint

# Run for specific package
turbo build --filter=@repo/web

# Run for package and dependencies
turbo build --filter=@repo/web...

# Run for changed packages
turbo build --filter=[HEAD^1]

# Dry run (show what would run)
turbo build --dry-run

# Force rebuild (ignore cache)
turbo build --force

# Graph visualization
turbo build --graph
```

## Filtering

```bash
# Single package
turbo build --filter=@repo/web

# Multiple packages
turbo build --filter=@repo/web --filter=@repo/docs

# All in directory
turbo build --filter=./apps/*

# Dependencies of package
turbo build --filter=@repo/web^...

# Package and its dependents
turbo build --filter=...@repo/ui

# Changed since commit
turbo build --filter=[HEAD^1]

# Changed in branch
turbo build --filter=[main...my-branch]
```

## Remote Caching

```bash
# Login to Vercel
turbo login

# Link to Vercel team
turbo link

# Enable remote cache
# Set in turbo.json or env
TURBO_TOKEN=xxx
TURBO_TEAM=my-team
```

```json
// turbo.json
{
  "remoteCache": {
    "signature": true
  }
}
```

## Environment Variables

```json
// turbo.json
{
  "globalEnv": ["CI", "NODE_ENV"],
  "tasks": {
    "build": {
      "env": ["API_URL", "DATABASE_URL"],
      "passThroughEnv": ["AWS_SECRET_KEY"]
    }
  }
}
```

## Shared UI Package

```typescript
// packages/ui/src/button.tsx
import { ReactNode } from 'react'

interface ButtonProps {
  children: ReactNode
  variant?: 'primary' | 'secondary'
  onClick?: () => void
}

export function Button({ children, variant = 'primary', onClick }: ButtonProps) {
  return (
    <button
      className={`btn btn-${variant}`}
      onClick={onClick}
    >
      {children}
    </button>
  )
}
```

```typescript
// packages/ui/src/index.ts
export * from './button'
export * from './card'
export * from './input'
```

```typescript
// apps/web/app/page.tsx
import { Button } from '@repo/ui'

export default function Home() {
  return <Button variant="primary">Click me</Button>
}
```

## Shared Config Package

```javascript
// packages/config-eslint/index.js
module.exports = {
  extends: ['next', 'turbo', 'prettier'],
  rules: {
    '@next/next/no-html-link-for-pages': 'off',
  },
}
```

```javascript
// apps/web/.eslintrc.js
module.exports = {
  root: true,
  extends: ['@repo/config-eslint'],
}
```

## CI/CD (GitHub Actions)

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: pnpm/action-setup@v2
        with:
          version: 8

      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'pnpm'

      - run: pnpm install

      - run: pnpm build
        env:
          TURBO_TOKEN: ${{ secrets.TURBO_TOKEN }}
          TURBO_TEAM: ${{ vars.TURBO_TEAM }}

      - run: pnpm lint
      - run: pnpm test
```

## Best Practices

```
1. Keep shared packages small and focused
2. Use workspace:* for internal dependencies
3. Enable remote caching in CI
4. Use --filter for targeted builds
5. Define clear task dependencies
6. Separate config packages (eslint, typescript)
```

---

*Learned: December 20, 2025*
*Tags: Turborepo, Monorepo, Build System, Vercel, pnpm*
