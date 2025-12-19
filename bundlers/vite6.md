# Vite 6 - Environment API for Multi-Runtime

<div align="center">

![Vite](https://img.shields.io/badge/Vite-646CFF?style=for-the-badge&logo=vite&logoColor=white)
![Build](https://img.shields.io/badge/Build_Tool-Fast-green?style=for-the-badge)
![Environment](https://img.shields.io/badge/Feature-Environment_API-orange?style=for-the-badge)

*The most significant Vite release since v2, introducing the Environment API for multi-runtime support.*

</div>

## What's New in Vite 6

```
Vite 5                         Vite 6
──────────────────────────     ──────────────────────────
2 implicit environments        Unlimited environments
Client + SSR only              Client, SSR, Edge, Workers
Sass legacy API                Sass modern API (default)
Runtime API (experimental)     Module Runner API (stable)
```

## The Environment API

Before Vite 6, there were only two implicit environments: `client` and `ssr`. Now you can create as many environments as needed.

```typescript
// vite.config.ts
import { defineConfig } from 'vite'

export default defineConfig({
  environments: {
    // Client environment (browser)
    client: {
      build: {
        outDir: 'dist/client',
        rollupOptions: {
          input: '/src/entry-client.ts',
        },
      },
    },

    // SSR environment (Node.js)
    ssr: {
      build: {
        outDir: 'dist/server',
        rollupOptions: {
          input: '/src/entry-server.ts',
        },
      },
    },

    // Edge environment (Cloudflare Workers)
    edge: {
      dev: {
        createEnvironment(name, config) {
          return createWorkerdDevEnvironment(name, config)
        },
      },
      build: {
        outDir: 'dist/edge',
        rollupOptions: {
          input: '/src/entry-edge.ts',
        },
      },
    },
  },
})
```

## Environment-Aware Plugins

```typescript
// Plugins can now target specific environments
function myPlugin() {
  return {
    name: 'my-plugin',

    // Access environment in hooks
    transform(code, id, options) {
      // options.environment tells you which environment
      if (options.environment.name === 'client') {
        return transformForClient(code)
      }
      if (options.environment.name === 'ssr') {
        return transformForSSR(code)
      }
    },

    // Environment-specific config
    configureServer(server) {
      // Access all environments
      for (const [name, env] of Object.entries(server.environments)) {
        console.log(`Environment: ${name}`)
      }
    },
  }
}
```

## Module Runner API

```typescript
// Create module runner for custom runtime
import { DevEnvironment, createServer } from 'vite'

const server = await createServer()

// Get SSR environment
const ssrEnv = server.environments.ssr as DevEnvironment

// Create module runner
const runner = createModuleRunner(ssrEnv)

// Load and execute module
const mod = await runner.import('/src/entry-server.ts')
```

## React Server Components Example

```typescript
// vite.config.ts - RSC support with Environment API
export default defineConfig({
  environments: {
    // Browser environment
    client: {
      resolve: {
        conditions: ['browser'],
      },
    },

    // React Server Components environment
    rsc: {
      resolve: {
        conditions: ['react-server'],
        noExternal: true,
      },
      dev: {
        optimizeDeps: {
          include: ['react', 'react-dom'],
        },
      },
    },

    // SSR for hydration
    ssr: {
      resolve: {
        conditions: ['node'],
      },
    },
  },
})
```

## Sass Modern API (Default)

```typescript
// Vite 6 uses modern Sass API by default
// No configuration needed for most projects

// If you need legacy API (rare):
export default defineConfig({
  css: {
    preprocessorOptions: {
      scss: {
        api: 'legacy', // Use legacy API
      },
    },
  },
})
```

## Migration from Vite 5

### Breaking Changes

```typescript
// 1. Sass uses modern API by default
// Update your Sass code if using legacy features

// 2. postcss-load-config parameters changed
// Check your postcss.config.js

// 3. resolve.conditions default changed
// Explicit conditions may need updates

// 4. JSON stringify changed
// Use json.stringify option if needed
```

### Plugin Migration

```typescript
// Before (Vite 5)
function oldPlugin() {
  return {
    transform(code, id, { ssr }) {
      if (ssr) {
        // Handle SSR
      }
    },
  }
}

// After (Vite 6)
function newPlugin() {
  return {
    transform(code, id, options) {
      // Use environment name instead of boolean
      const env = options?.environment?.name

      if (env === 'ssr') {
        // Handle SSR
      } else if (env === 'client') {
        // Handle client
      }
    },
  }
}
```

## Dev Server with Environments

```typescript
import { createServer } from 'vite'

const server = await createServer({
  environments: {
    client: {},
    ssr: {},
    workerd: {
      dev: {
        createEnvironment: (name, config) => {
          return createWorkerdEnvironment(name, config)
        },
      },
    },
  },
})

await server.listen()

// All environments run concurrently
// Shared HTTP server, middlewares, config, plugins
```

## Hot Module Replacement

```typescript
// HMR works across all environments
if (import.meta.hot) {
  import.meta.hot.accept((newModule) => {
    // Handle HMR update
  })

  // Environment-aware HMR
  import.meta.hot.on('vite:beforeUpdate', (payload) => {
    console.log('Update in:', payload.updates)
  })
}
```

## Backward Compatibility

```typescript
// SPAs work exactly the same
export default defineConfig({
  // No changes needed for simple SPAs
  plugins: [react()],
})

// Existing SSR setups continue to work
export default defineConfig({
  plugins: [react()],
  ssr: {
    // Still supported
    noExternal: ['some-package'],
  },
})
```

## Framework Author Benefits

```typescript
// Framework can create custom environments
import { defineConfig } from 'vite'
import { myFramework } from 'my-framework'

export default defineConfig({
  plugins: [
    myFramework({
      // Framework automatically configures environments
      // for client, server, edge, etc.
    }),
  ],
})
```

---

*Learned: December 20, 2025*
*Tags: Vite, Build Tool, Environment API, Bundler, Development*
