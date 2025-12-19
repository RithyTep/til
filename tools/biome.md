# Biome - Fast Linter and Formatter

<div align="center">

![Biome](https://img.shields.io/badge/Biome-60A5FA?style=for-the-badge&logo=biome&logoColor=white)
![Rust](https://img.shields.io/badge/Rust-DEA584?style=for-the-badge&logo=rust&logoColor=black)
![Linter](https://img.shields.io/badge/Tool-Linter_+_Formatter-green?style=for-the-badge)

*10-25x faster than ESLint + Prettier, unified in a single tool.*

</div>

## Why Biome?

```
ESLint + Prettier              Biome
──────────────────────────     ──────────────────────────
127+ npm packages              1 binary (zero deps)
4 config files                 1 config file
10-30 seconds lint             < 1 second lint
Multiple tools to learn        One unified tool
```

## Performance

```
String parsing:   14x faster than ESLint
Array parsing:    7x faster
Object parsing:   6.5x faster
Formatting:       97% Prettier compatible
```

## Installation

```bash
# npm
npm install --save-dev --save-exact @biomejs/biome

# pnpm
pnpm add --save-dev --save-exact @biomejs/biome

# Initialize config
npx @biomejs/biome init
```

## Configuration

```json
// biome.json
{
  "$schema": "https://biomejs.dev/schemas/1.9.0/schema.json",
  "organizeImports": {
    "enabled": true
  },
  "linter": {
    "enabled": true,
    "rules": {
      "recommended": true,
      "complexity": {
        "noForEach": "warn",
        "useFlatMap": "error"
      },
      "suspicious": {
        "noExplicitAny": "error"
      },
      "style": {
        "useConst": "error",
        "noUnusedTemplateLiteral": "warn"
      }
    }
  },
  "formatter": {
    "enabled": true,
    "indentStyle": "space",
    "indentWidth": 2,
    "lineWidth": 100
  },
  "javascript": {
    "formatter": {
      "quoteStyle": "single",
      "trailingCommas": "es5",
      "semicolons": "asNeeded"
    }
  }
}
```

## CLI Commands

```bash
# Format files
npx biome format --write .

# Lint files
npx biome lint .

# Both format and lint
npx biome check --write .

# CI mode (no writes, exit 1 on errors)
npx biome ci .

# Check specific files
npx biome check src/**/*.ts
```

## Migrate from ESLint/Prettier

```bash
# Auto-migrate ESLint config
npx @biomejs/biome migrate eslint --write

# Auto-migrate Prettier config
npx @biomejs/biome migrate prettier --write

# Both at once
npx @biomejs/biome migrate eslint prettier --write
```

### Remove Old Dependencies

```bash
# After migration, remove old tools
npm uninstall eslint prettier eslint-config-prettier \
  @typescript-eslint/eslint-plugin @typescript-eslint/parser \
  eslint-plugin-react eslint-plugin-react-hooks
```

## Editor Integration

### VS Code

```json
// .vscode/settings.json
{
  "editor.defaultFormatter": "biomejs.biome",
  "editor.formatOnSave": true,
  "editor.codeActionsOnSave": {
    "quickfix.biome": "explicit",
    "source.organizeImports.biome": "explicit"
  }
}
```

### package.json Scripts

```json
{
  "scripts": {
    "lint": "biome lint .",
    "format": "biome format --write .",
    "check": "biome check --write .",
    "ci": "biome ci ."
  }
}
```

## Biome 2.0 Features

### Type-Aware Linting

```typescript
// biome.json
{
  "linter": {
    "rules": {
      "correctness": {
        // Catches type-related bugs without full TS compiler
        "noUndeclaredVariables": "error"
      }
    }
  }
}
```

### Plugins (v2.0+)

```json
// biome.json
{
  "plugins": ["@biomejs/plugin-react"]
}
```

## Rule Examples

### Complexity Rules

```typescript
// noForEach: prefer for...of
// Bad
array.forEach(item => console.log(item))

// Good
for (const item of array) {
  console.log(item)
}

// useFlatMap: use flatMap instead of map().flat()
// Bad
array.map(x => [x, x * 2]).flat()

// Good
array.flatMap(x => [x, x * 2])
```

### Suspicious Rules

```typescript
// noExplicitAny: avoid any type
// Bad
function process(data: any) {}

// Good
function process(data: unknown) {}

// noDoubleEquals: use strict equality
// Bad
if (x == null) {}

// Good
if (x === null || x === undefined) {}
```

### Style Rules

```typescript
// useConst: prefer const over let
// Bad
let x = 5 // never reassigned

// Good
const x = 5

// noUnusedTemplateLiteral
// Bad
const str = `hello` // no interpolation

// Good
const str = 'hello'
```

## Ignore Patterns

```json
// biome.json
{
  "files": {
    "ignore": [
      "node_modules",
      "dist",
      "build",
      "*.min.js",
      "generated/**"
    ]
  }
}
```

## Inline Ignores

```typescript
// Ignore next line
// biome-ignore lint/suspicious/noExplicitAny: legacy code
function legacy(data: any) {}

// Ignore specific rule in block
/* biome-ignore lint/complexity/noForEach: performance critical */
array.forEach(processItem)
```

## When to Use Biome

| Scenario | Recommendation |
|----------|---------------|
| New projects | Use Biome |
| React/Next.js | Fully supported |
| Vue/Svelte | Partial support |
| Monorepos | Excellent |
| CI/CD speed | Major improvement |
| HTML/Markdown | Still needs Prettier |

---

*Learned: December 20, 2025*
*Tags: Biome, Linter, Formatter, ESLint, Prettier, Rust*
