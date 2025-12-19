# Template Literal Types for String Patterns

TypeScript 4.1+ lets you create types from string patterns using template literals.

## Basic Usage

```typescript
type EventName = `on${string}`;

const valid: EventName = "onClick";    // ✅
const invalid: EventName = "click";    // ❌ Error
```

## Real-World Example: API Routes

```typescript
type HTTPMethod = "GET" | "POST" | "PUT" | "DELETE";
type APIVersion = "v1" | "v2";
type Endpoint = `/${APIVersion}/${"users" | "orders" | "products"}`;

// Result: "/v1/users" | "/v1/orders" | "/v1/products" | "/v2/users" | ...

const route: Endpoint = "/v1/users"; // ✅
```

## CSS Units

```typescript
type CSSUnit = "px" | "rem" | "em" | "%";
type CSSValue = `${number}${CSSUnit}`;

const padding: CSSValue = "16px";  // ✅
const margin: CSSValue = "2rem";   // ✅
const bad: CSSValue = "16";        // ❌ Error
```

## Extracting Parts

```typescript
type ExtractRouteParams<T extends string> =
  T extends `${string}:${infer Param}/${infer Rest}`
    ? Param | ExtractRouteParams<`/${Rest}`>
    : T extends `${string}:${infer Param}`
    ? Param
    : never;

type Params = ExtractRouteParams<"/users/:userId/posts/:postId">;
// Result: "userId" | "postId"
```

---

📅 *Learned: 2024*
🏷️ *Tags: TypeScript, Template Literals*
