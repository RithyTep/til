# Use `satisfies` for Better Type Inference

The `satisfies` operator (TypeScript 4.9+) validates that an expression matches a type while keeping the most specific type inferred.

## Problem

```typescript
// Using type annotation loses specific types
const colors: Record<string, string> = {
  red: "#ff0000",
  green: "#00ff00",
};

colors.red.toUpperCase(); // ✅ Works
colors.blue; // ✅ No error - but it doesn't exist!
```

## Solution

```typescript
// Using satisfies keeps literal types
const colors = {
  red: "#ff0000",
  green: "#00ff00",
} satisfies Record<string, string>;

colors.red.toUpperCase(); // ✅ Works
colors.blue; // ❌ Error: Property 'blue' does not exist
```

## Real-World Example

```typescript
type Route = {
  path: string;
  component: string;
  children?: Route[];
};

const routes = {
  home: { path: "/", component: "HomePage" },
  about: { path: "/about", component: "AboutPage" },
  contact: { path: "/contact", component: "ContactPage" },
} satisfies Record<string, Route>;

// Now TypeScript knows exactly which routes exist
routes.home.path; // ✅ Autocomplete works!
routes.notExist; // ❌ Error
```

## When to Use

- ✅ Configuration objects
- ✅ Route definitions
- ✅ Theme objects
- ✅ Any object where you want both validation AND inference

---

📅 *Learned: December 20, 2025*
🏷️ *Tags: TypeScript, Type Safety*
