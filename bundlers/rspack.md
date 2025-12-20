# Rspack - Rust-Powered Webpack Replacement

<div align="center">

![Rspack](https://img.shields.io/badge/Rspack-FFA500?style=for-the-badge&logo=webpack&logoColor=white)
![Rust](https://img.shields.io/badge/Rust-DEA584?style=for-the-badge&logo=rust&logoColor=black)
![Bundler](https://img.shields.io/badge/Bundler-23x_Faster-green?style=for-the-badge)

*Drop-in webpack replacement with 23x faster builds, powered by Rust.*

![Rspack](https://raw.githubusercontent.com/web-infra-dev/rspack/main/website/public/logo.png)

</div>

## Why Rspack?

```
Webpack                        Rspack
──────────────────────────     ──────────────────────────
JavaScript-based               Rust-based (fast!)
~60s build (large app)         ~3s build (same app)
Complex config needed          Webpack-compatible config
Mature ecosystem               Uses webpack ecosystem
Sequential processing          Parallel processing
```

## Performance

```
Build Time:           23x faster than webpack
Dev Startup:          10x faster
HMR:                  Near-instant
Memory Usage:         Lower footprint
Plugin Compatibility: 80%+ of top 50 webpack plugins
```

## Who Uses It

- TikTok
- Douyin
- Lark
- Microsoft
- Amazon
- Discord

## Installation

```bash
# New project
npm create rspack@latest

# Or add to existing project
npm install -D @rspack/core @rspack/cli
```

## Basic Configuration

```javascript
// rspack.config.js
const path = require('path')

module.exports = {
  entry: './src/index.js',

  output: {
    path: path.resolve(__dirname, 'dist'),
    filename: '[name].[contenthash].js',
  },

  module: {
    rules: [
      {
        test: /\.tsx?$/,
        use: 'builtin:swc-loader',
        type: 'javascript/auto',
      },
      {
        test: /\.css$/,
        type: 'css',
      },
    ],
  },

  resolve: {
    extensions: ['.ts', '.tsx', '.js', '.jsx'],
  },
}
```

## TypeScript Configuration

```typescript
// rspack.config.ts
import { defineConfig } from '@rspack/cli'
import path from 'path'

export default defineConfig({
  entry: {
    main: './src/index.tsx',
  },

  output: {
    path: path.resolve(__dirname, 'dist'),
    filename: '[name].[contenthash].js',
    clean: true,
  },

  module: {
    rules: [
      {
        test: /\.(ts|tsx)$/,
        use: {
          loader: 'builtin:swc-loader',
          options: {
            jsc: {
              parser: {
                syntax: 'typescript',
                tsx: true,
              },
              transform: {
                react: {
                  runtime: 'automatic',
                },
              },
            },
          },
        },
      },
    ],
  },

  resolve: {
    extensions: ['.ts', '.tsx', '.js', '.jsx'],
    alias: {
      '@': path.resolve(__dirname, 'src'),
    },
  },
})
```

## Built-in Features

### SWC Loader (Blazing Fast)

```javascript
module: {
  rules: [
    {
      test: /\.jsx?$/,
      use: {
        loader: 'builtin:swc-loader',
        options: {
          jsc: {
            parser: {
              syntax: 'ecmascript',
              jsx: true,
            },
            transform: {
              react: {
                runtime: 'automatic',
                development: process.env.NODE_ENV === 'development',
                refresh: true, // React Fast Refresh
              },
            },
          },
        },
      },
    },
  ],
}
```

### CSS Handling

```javascript
module: {
  rules: [
    // Native CSS support
    {
      test: /\.css$/,
      type: 'css',
    },

    // CSS Modules
    {
      test: /\.module\.css$/,
      type: 'css/module',
    },

    // SCSS
    {
      test: /\.scss$/,
      use: ['sass-loader'],
      type: 'css',
    },

    // PostCSS
    {
      test: /\.css$/,
      use: ['postcss-loader'],
      type: 'css',
    },
  ],
}
```

### Asset Handling

```javascript
module: {
  rules: [
    // Images
    {
      test: /\.(png|jpg|gif|svg)$/,
      type: 'asset',
      parser: {
        dataUrlCondition: {
          maxSize: 8 * 1024, // 8kb
        },
      },
    },

    // Fonts
    {
      test: /\.(woff|woff2|eot|ttf|otf)$/,
      type: 'asset/resource',
    },
  ],
}
```

## Webpack Plugin Compatibility

```javascript
// Most webpack plugins work directly
const HtmlWebpackPlugin = require('html-webpack-plugin')
const { DefinePlugin } = require('@rspack/core')

module.exports = {
  plugins: [
    new HtmlWebpackPlugin({
      template: './src/index.html',
    }),

    new DefinePlugin({
      'process.env.API_URL': JSON.stringify(process.env.API_URL),
    }),
  ],
}
```

### Rspack Built-in Plugins

```javascript
const rspack = require('@rspack/core')

module.exports = {
  plugins: [
    // HTML generation
    new rspack.HtmlRspackPlugin({
      template: './src/index.html',
    }),

    // Copy static files
    new rspack.CopyRspackPlugin({
      patterns: [{ from: 'public' }],
    }),

    // Progress indicator
    new rspack.ProgressPlugin({}),

    // Bundle analyzer
    new rspack.BundleAnalyzerPlugin({}),
  ],
}
```

## Migration from Webpack

### Step 1: Install

```bash
npm install -D @rspack/core @rspack/cli
npm uninstall webpack webpack-cli
```

### Step 2: Rename Config

```bash
# Rename webpack.config.js to rspack.config.js
mv webpack.config.js rspack.config.js
```

### Step 3: Update Scripts

```json
{
  "scripts": {
    "dev": "rspack serve",
    "build": "rspack build"
  }
}
```

### Step 4: Replace Loaders

```javascript
// Before (webpack)
{
  test: /\.tsx?$/,
  use: 'ts-loader',
}

// After (rspack)
{
  test: /\.tsx?$/,
  use: 'builtin:swc-loader',
}
```

## Dev Server

```javascript
module.exports = {
  devServer: {
    port: 3000,
    hot: true,
    open: true,
    historyApiFallback: true,

    proxy: {
      '/api': {
        target: 'http://localhost:8080',
        changeOrigin: true,
      },
    },
  },
}
```

## Production Optimization

```javascript
module.exports = {
  mode: 'production',

  optimization: {
    minimize: true,
    splitChunks: {
      chunks: 'all',
      cacheGroups: {
        vendor: {
          test: /[\\/]node_modules[\\/]/,
          name: 'vendors',
          chunks: 'all',
        },
      },
    },
  },

  output: {
    filename: '[name].[contenthash].js',
    chunkFilename: '[name].[contenthash].chunk.js',
    clean: true,
  },
}
```

## Next.js Integration

```javascript
// next.config.js
/** @type {import('next').NextConfig} */
const nextConfig = {
  experimental: {
    rspackBundler: true, // Enable Rspack
  },
}

module.exports = nextConfig
```

## CLI Commands

```bash
# Development server
rspack serve

# Production build
rspack build

# Build with specific config
rspack build --config rspack.prod.config.js

# Analyze bundle
rspack build --analyze
```

---

*Learned: December 20, 2025*
*Tags: Rspack, Bundler, Webpack, Rust, Build Tool*
