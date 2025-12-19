# Tailwind CSS v4 - Next-Gen Styling

<div align="center">

![Tailwind](https://img.shields.io/badge/Tailwind_CSS-06B6D4?style=for-the-badge&logo=tailwindcss&logoColor=white)
![CSS](https://img.shields.io/badge/CSS-New_Engine-blue?style=for-the-badge)
![Performance](https://img.shields.io/badge/Performance-10x_Faster-green?style=for-the-badge)

*Rebuilt from the ground up with a new high-performance engine and CSS-first configuration.*

</div>

## What's New in v4

```
Tailwind v3                    Tailwind v4
──────────────────────────     ──────────────────────────
JS config file                 CSS-first config
PostCSS plugin                 Standalone + Vite plugin
JS-based engine                Oxide engine (Rust)
Config merging                 CSS @import
Custom colors in JS            CSS variables native
```

## Installation

```bash
# Vite (recommended)
npm install tailwindcss @tailwindcss/vite

# PostCSS
npm install tailwindcss @tailwindcss/postcss

# CLI
npm install tailwindcss @tailwindcss/cli
```

## Vite Setup

```typescript
// vite.config.ts
import { defineConfig } from 'vite'
import tailwindcss from '@tailwindcss/vite'

export default defineConfig({
  plugins: [tailwindcss()],
})
```

```css
/* app.css */
@import "tailwindcss";
```

## CSS-First Configuration

```css
/* app.css */
@import "tailwindcss";

/* Theme customization */
@theme {
  /* Colors */
  --color-primary: #6366f1;
  --color-primary-dark: #4f46e5;
  --color-secondary: #ec4899;

  /* Custom color palette */
  --color-brand-50: #eff6ff;
  --color-brand-100: #dbeafe;
  --color-brand-500: #3b82f6;
  --color-brand-900: #1e3a8a;

  /* Fonts */
  --font-display: "Cal Sans", sans-serif;
  --font-body: "Inter", sans-serif;

  /* Spacing */
  --spacing-128: 32rem;

  /* Border radius */
  --radius-4xl: 2rem;

  /* Shadows */
  --shadow-glow: 0 0 20px rgba(99, 102, 241, 0.5);

  /* Animations */
  --animate-fade-in: fade-in 0.5s ease-out;
}

@keyframes fade-in {
  from { opacity: 0; }
  to { opacity: 1; }
}
```

## Using Custom Theme

```html
<!-- Use custom colors -->
<button class="bg-primary hover:bg-primary-dark text-white">
  Click me
</button>

<!-- Use custom color palette -->
<div class="bg-brand-50 text-brand-900">
  Brand colored section
</div>

<!-- Use custom spacing -->
<div class="w-128">Wide container</div>

<!-- Use custom animation -->
<div class="animate-fade-in">Fading in</div>
```

## Container Queries (Built-in)

```html
<!-- v4 has native container query support -->
<div class="@container">
  <div class="@sm:grid-cols-2 @lg:grid-cols-4">
    <!-- Responds to container, not viewport -->
  </div>
</div>

<!-- Named containers -->
<div class="@container/sidebar">
  <nav class="@lg/sidebar:flex-row">
    Navigation
  </nav>
</div>
```

## CSS Variables Integration

```css
/* Define with CSS variables */
@theme {
  --color-surface: light-dark(white, #1a1a1a);
  --color-text: light-dark(#1a1a1a, white);
}

/* Or use native CSS */
:root {
  --accent: #6366f1;
}

.dark {
  --accent: #818cf8;
}
```

```html
<!-- Use directly in Tailwind classes -->
<div class="bg-[--accent] text-surface">
  Themed content
</div>
```

## New Variant Syntax

```css
/* Custom variants in CSS */
@variant hocus (&:hover, &:focus);
@variant pointer-fine (@media (pointer: fine));
@variant theme-dark (.dark &);
```

```html
<!-- Use custom variants -->
<button class="hocus:bg-blue-600">Hover or Focus</button>
<div class="pointer-fine:text-sm">Fine pointer only</div>
```

## Composing Utilities

```css
/* Create composed utilities */
@utility btn {
  @apply px-4 py-2 rounded-lg font-medium transition-colors;
}

@utility btn-primary {
  @apply btn bg-primary text-white hover:bg-primary-dark;
}

@utility card {
  @apply bg-white dark:bg-gray-800 rounded-xl shadow-lg p-6;
}
```

```html
<button class="btn-primary">Primary Button</button>
<div class="card">Card content</div>
```

## Source Detection

```css
/* Configure which files to scan */
@source "../src/**/*.{js,ts,jsx,tsx}";
@source "../components/**/*.{js,ts,jsx,tsx}";

/* Exclude files */
@source not "../src/**/*.test.{js,ts}";
```

## Plugins in CSS

```css
/* Import plugins */
@plugin "@tailwindcss/typography";
@plugin "@tailwindcss/forms";

/* Plugin with options */
@plugin "@tailwindcss/typography" {
  className: "prose";
}
```

## Dark Mode

```css
/* CSS-based dark mode config */
@theme {
  /* Automatic based on system preference */
  --color-bg: light-dark(#ffffff, #0a0a0a);
  --color-text: light-dark(#171717, #ededed);
}

/* Or class-based */
@variant dark (&:where(.dark, .dark *));
```

```html
<!-- Toggle dark mode -->
<html class="dark">
  <body class="bg-bg text-text">
    Automatically themed
  </body>
</html>
```

## Responsive Breakpoints

```css
/* Custom breakpoints */
@theme {
  --breakpoint-xs: 475px;
  --breakpoint-3xl: 1920px;
}
```

```html
<!-- Use custom breakpoints -->
<div class="xs:block 3xl:max-w-screen-3xl">
  Responsive content
</div>
```

## Migration from v3

```css
/* v3 - tailwind.config.js */
module.exports = {
  theme: {
    extend: {
      colors: {
        primary: '#6366f1',
      },
    },
  },
}

/* v4 - app.css */
@import "tailwindcss";

@theme {
  --color-primary: #6366f1;
}
```

## Performance

```
Build Time:    ~10x faster
Full Rebuild:  < 100ms typical
Incremental:   < 5ms
Bundle Size:   Smaller CSS output
```

## Compatibility

```css
/* Ensure browser support */
@supports (color: oklch(0 0 0)) {
  @theme {
    --color-primary: oklch(0.65 0.25 265);
  }
}
```

---

*Learned: December 20, 2025*
*Tags: Tailwind CSS, CSS, Styling, v4, Design System*
