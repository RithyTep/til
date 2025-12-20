# Zod 4 - Type-Safe Schema Validation

<div align="center">

![Zod](https://img.shields.io/badge/Zod-3E67B1?style=for-the-badge&logo=zod&logoColor=white)
![TypeScript](https://img.shields.io/badge/TypeScript-3178C6?style=for-the-badge&logo=typescript&logoColor=white)
![Validation](https://img.shields.io/badge/Feature-Schema_Validation-green?style=for-the-badge)

*14x faster parsing, 57% smaller core, with new @zod/mini for lightweight validation.*

![Zod](https://raw.githubusercontent.com/colinhacks/zod/main/logo.svg)

</div>

## What's New in Zod 4

```
Zod 3                          Zod 4
──────────────────────────     ──────────────────────────
Baseline performance           14x faster strings
Standard bundle                57% smaller core
superRefine for custom         .check() method (better)
Single package                 + @zod/mini (~1.9kb)
```

## Performance Improvements

```
String parsing:    14x faster
Array parsing:     7x faster
Object parsing:    6.5x faster
Type instantiation: Faster TypeScript compilation
```

## Installation

```bash
# Full Zod
npm install zod

# Zod Mini (1.9kb, tree-shakable)
npm install @zod/mini
```

## Basic Schemas

```typescript
import { z } from 'zod'

// Primitives
const stringSchema = z.string()
const numberSchema = z.number()
const booleanSchema = z.boolean()
const dateSchema = z.date()
const bigintSchema = z.bigint()

// Literals
const literalSchema = z.literal('hello')
const enumSchema = z.enum(['active', 'inactive', 'pending'])

// Parse and validate
const result = stringSchema.parse('hello') // 'hello'
const result2 = numberSchema.parse('123')  // throws ZodError

// Safe parse (no throw)
const safe = stringSchema.safeParse('hello')
if (safe.success) {
  console.log(safe.data)
} else {
  console.log(safe.error)
}
```

## String Validations

```typescript
const userSchema = z.object({
  email: z.string().email(),
  username: z.string()
    .min(3, 'Username must be at least 3 characters')
    .max(20, 'Username must be at most 20 characters')
    .regex(/^[a-zA-Z0-9_]+$/, 'Only alphanumeric and underscore'),
  website: z.string().url().optional(),
  phone: z.string().regex(/^\+?[1-9]\d{1,14}$/),
  uuid: z.string().uuid(),
  ip: z.string().ip(),
})
```

## Number Validations

```typescript
const productSchema = z.object({
  price: z.number().positive().finite(),
  quantity: z.number().int().min(0).max(1000),
  discount: z.number().min(0).max(100),
  rating: z.number().min(1).max(5).multipleOf(0.5),
})
```

## Object Schemas

```typescript
const userSchema = z.object({
  id: z.string().uuid(),
  name: z.string(),
  email: z.string().email(),
  age: z.number().optional(),
  role: z.enum(['admin', 'user', 'guest']).default('user'),
  metadata: z.record(z.string()), // { [key: string]: string }
})

// Type inference
type User = z.infer<typeof userSchema>

// Partial (all optional)
const partialUser = userSchema.partial()

// Pick specific fields
const loginSchema = userSchema.pick({ email: true })

// Omit fields
const publicUser = userSchema.omit({ role: true })

// Extend
const adminSchema = userSchema.extend({
  permissions: z.array(z.string()),
})

// Merge
const fullSchema = userSchema.merge(z.object({
  createdAt: z.date(),
}))
```

## Array Schemas

```typescript
const tagsSchema = z.array(z.string()).min(1).max(10)

const postsSchema = z.array(z.object({
  title: z.string(),
  content: z.string(),
})).nonempty()

// Tuple (fixed length, different types)
const coordinatesSchema = z.tuple([z.number(), z.number()])
```

## Union & Intersection

```typescript
// Union (OR)
const stringOrNumber = z.union([z.string(), z.number()])
// Shorthand
const stringOrNumber2 = z.string().or(z.number())

// Discriminated union (better error messages)
const resultSchema = z.discriminatedUnion('status', [
  z.object({ status: z.literal('success'), data: z.string() }),
  z.object({ status: z.literal('error'), message: z.string() }),
])

// Intersection (AND)
const withTimestamp = z.object({ createdAt: z.date() })
const userWithTimestamp = userSchema.and(withTimestamp)
```

## Transformations

```typescript
// Transform input
const lowercaseEmail = z.string().email().transform(s => s.toLowerCase())

// Coerce types
const numberFromString = z.coerce.number() // "123" -> 123
const dateFromString = z.coerce.date()     // "2024-01-01" -> Date
const boolFromString = z.coerce.boolean()  // "true" -> true

// Preprocess
const trimmedString = z.preprocess(
  (val) => typeof val === 'string' ? val.trim() : val,
  z.string()
)
```

## Zod 4: .check() Method

```typescript
// New in Zod 4: .check() replaces .superRefine()
// Returns multiple issues at once

const passwordSchema = z.string().check((val, ctx) => {
  const issues = []

  if (val.length < 8) {
    issues.push({ message: 'Password must be at least 8 characters' })
  }
  if (!/[A-Z]/.test(val)) {
    issues.push({ message: 'Password must contain uppercase letter' })
  }
  if (!/[0-9]/.test(val)) {
    issues.push({ message: 'Password must contain number' })
  }

  return issues // Return all issues at once
})

// Cross-field validation
const formSchema = z.object({
  password: z.string().min(8),
  confirmPassword: z.string(),
}).check((data, ctx) => {
  if (data.password !== data.confirmPassword) {
    return [{ path: ['confirmPassword'], message: 'Passwords must match' }]
  }
  return []
})
```

## Async Validation

```typescript
const uniqueEmailSchema = z.string().email().refine(
  async (email) => {
    const exists = await checkEmailExists(email)
    return !exists
  },
  { message: 'Email already taken' }
)

// Use parseAsync
const result = await uniqueEmailSchema.parseAsync('user@example.com')
```

## Error Handling

```typescript
try {
  userSchema.parse(invalidData)
} catch (error) {
  if (error instanceof z.ZodError) {
    // Formatted errors
    console.log(error.format())

    // Flat errors
    console.log(error.flatten())

    // Access issues
    error.issues.forEach(issue => {
      console.log(issue.path, issue.message)
    })
  }
}

// Custom error messages
const schema = z.object({
  email: z.string({
    required_error: 'Email is required',
    invalid_type_error: 'Email must be a string',
  }).email({ message: 'Invalid email format' }),
})
```

## @zod/mini (Zod 4)

```typescript
// Ultra-lightweight validation (~1.9kb gzipped)
import { string, object, email, min } from '@zod/mini'

const userSchema = object({
  email: string([email()]),
  name: string([min(1)]),
})

// Tree-shakable - only import what you need
```

## Real-World Example

```typescript
// API request validation
const createUserRequest = z.object({
  body: z.object({
    email: z.string().email(),
    password: z.string().min(8),
    name: z.string().min(1).max(100),
    preferences: z.object({
      newsletter: z.boolean().default(false),
      theme: z.enum(['light', 'dark']).default('light'),
    }).optional(),
  }),
  query: z.object({
    referrer: z.string().optional(),
  }),
})

// With TypeScript
type CreateUserRequest = z.infer<typeof createUserRequest>

// Usage in API handler
function handler(req: Request) {
  const validated = createUserRequest.parse({
    body: req.body,
    query: req.query,
  })
  // validated.body.email is typed as string
}
```

---

*Learned: December 20, 2025*
*Tags: Zod, TypeScript, Validation, Schema, Type Safety*
