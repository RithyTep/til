# Const Assertions with `as const`

Use `as const` to make TypeScript infer the most specific literal types.

## Problem

```typescript
const colors = {
  primary: "#6366f1",
  secondary: "#a855f7"
};
// Type: { primary: string; secondary: string }

// We lose the literal types!
```

## Solution

```typescript
const colors = {
  primary: "#6366f1",
  secondary: "#a855f7"
} as const;
// Type: { readonly primary: "#6366f1"; readonly secondary: "#a855f7" }

// Now TypeScript knows the exact values!
```

## Arrays

```typescript
// Without as const
const statuses = ["pending", "active", "done"];
// Type: string[]

// With as const
const statuses = ["pending", "active", "done"] as const;
// Type: readonly ["pending", "active", "done"]

type Status = typeof statuses[number];
// Type: "pending" | "active" | "done"
```

## Function Returns

```typescript
function getConfig() {
  return {
    apiUrl: "https://api.example.com",
    timeout: 5000
  } as const;
}

const config = getConfig();
// config.apiUrl is "https://api.example.com", not string
```

## Enum Alternative

```typescript
// Instead of enums, use as const
const Direction = {
  Up: "UP",
  Down: "DOWN",
  Left: "LEFT",
  Right: "RIGHT"
} as const;

type Direction = typeof Direction[keyof typeof Direction];
// Type: "UP" | "DOWN" | "LEFT" | "RIGHT"
```

---

📅 *Learned: 2024*
🏷️ *Tags: TypeScript, Const Assertions, Literals*
