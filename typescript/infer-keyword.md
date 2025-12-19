# Using `infer` in Conditional Types

The `infer` keyword lets you extract types from within other types.

## Basic Example

```typescript
// Extract return type of a function
type ReturnType<T> = T extends (...args: any[]) => infer R ? R : never;

type Fn = () => string;
type Result = ReturnType<Fn>; // string
```

## Extract Array Element Type

```typescript
type ElementType<T> = T extends (infer E)[] ? E : never;

type Numbers = ElementType<number[]>;  // number
type Strings = ElementType<string[]>;  // string
```

## Extract Promise Value

```typescript
type Awaited<T> = T extends Promise<infer V> ? V : T;

type A = Awaited<Promise<string>>;  // string
type B = Awaited<Promise<number>>;  // number
type C = Awaited<string>;           // string (not a promise)
```

## Extract Function Parameters

```typescript
type Parameters<T> = T extends (...args: infer P) => any ? P : never;

type Fn = (name: string, age: number) => void;
type Params = Parameters<Fn>;  // [string, number]
```

## Real-World: Extract Event Payload

```typescript
type EventMap = {
  click: { x: number; y: number };
  submit: { data: FormData };
  error: { message: string };
};

type EventPayload<K extends keyof EventMap> = EventMap[K];

function emit<K extends keyof EventMap>(
  event: K,
  payload: EventPayload<K>
) { ... }

emit("click", { x: 10, y: 20 });  // ✅ Type-safe!
```

---

📅 *Learned: December 20, 2025*
🏷️ *Tags: TypeScript, Conditional Types, Generics*
