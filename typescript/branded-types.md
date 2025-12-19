# Branded Types for Type-Safe IDs

Branded types prevent mixing up IDs of different entities even though they're all strings.

## Problem

```typescript
function getUser(userId: string) { ... }
function getOrder(orderId: string) { ... }

const userId = "user_123";
const orderId = "order_456";

// Oops! Wrong ID passed - but TypeScript doesn't catch it
getUser(orderId); // ✅ No error, but this is a bug!
```

## Solution: Branded Types

```typescript
// Create branded types
type UserId = string & { readonly __brand: "UserId" };
type OrderId = string & { readonly __brand: "OrderId" };

// Helper functions to create branded values
const createUserId = (id: string): UserId => id as UserId;
const createOrderId = (id: string): OrderId => id as OrderId;

function getUser(userId: UserId) { ... }
function getOrder(orderId: OrderId) { ... }

const userId = createUserId("user_123");
const orderId = createOrderId("order_456");

getUser(userId);  // ✅ Works
getUser(orderId); // ❌ Error: Argument of type 'OrderId' is not assignable to 'UserId'
```

## Reusable Brand Utility

```typescript
// Generic brand utility
type Brand<T, B> = T & { readonly __brand: B };

type UserId = Brand<string, "UserId">;
type OrderId = Brand<string, "OrderId">;
type Email = Brand<string, "Email">;
type Currency = Brand<number, "Currency">;
```

## Real-World Use Cases

- User IDs vs Order IDs vs Product IDs
- Email addresses vs regular strings
- Currency amounts vs regular numbers
- Validated vs unvalidated data

---

📅 *Learned: December 20, 2025*
🏷️ *Tags: TypeScript, Type Safety, Domain Modeling*
