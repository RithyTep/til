# React 19 - Compiler, Actions, and New Hooks

<div align="center">

![React](https://img.shields.io/badge/React-19-61DAFB?style=for-the-badge&logo=react&logoColor=white)
![TypeScript](https://img.shields.io/badge/TypeScript-5.7-3178C6?style=for-the-badge&logo=typescript&logoColor=white)

*The most significant React release since Hooks, with automatic optimizations and server-first patterns.*

[![React Docs](https://img.shields.io/badge/Docs-react.dev-blue)](https://react.dev)

</div>

## React Compiler (React Forget)

```
┌─────────────────────────────────────────────────────────────────┐
│                    REACT COMPILER FLOW                           │
│                                                                 │
│   ┌──────────────┐        ┌──────────────┐                     │
│   │  Your Code   │        │   Compiler   │                     │
│   │  (No memo)   │───────►│  Analyzes    │                     │
│   └──────────────┘        │  Dependencies│                     │
│                           └──────────────┘                     │
│                                  │                              │
│                                  ▼                              │
│                           ┌──────────────┐                     │
│                           │  Optimized   │                     │
│                           │  Output with │                     │
│                           │  Auto-Memo   │                     │
│                           └──────────────┘                     │
│                                                                 │
│   Before: useMemo, useCallback everywhere                      │
│   After:  Compiler handles memoization automatically           │
│                                                                 │
│   Result: 25-40% fewer re-renders without code changes         │
└─────────────────────────────────────────────────────────────────┘
```

### Before React 19

```tsx
// Manual memoization everywhere
function ProductList({ products, onSelect }) {
  const sortedProducts = useMemo(
    () => products.sort((a, b) => a.price - b.price),
    [products]
  );

  const handleSelect = useCallback(
    (id) => onSelect(id),
    [onSelect]
  );

  return sortedProducts.map(p => (
    <Product key={p.id} product={p} onSelect={handleSelect} />
  ));
}

const Product = memo(({ product, onSelect }) => (
  <div onClick={() => onSelect(product.id)}>
    {product.name} - ${product.price}
  </div>
));
```

### After React 19

```tsx
// Compiler handles optimization automatically
function ProductList({ products, onSelect }) {
  const sortedProducts = products.sort((a, b) => a.price - b.price);

  return sortedProducts.map(p => (
    <Product key={p.id} product={p} onSelect={onSelect} />
  ));
}

function Product({ product, onSelect }) {
  return (
    <div onClick={() => onSelect(product.id)}>
      {product.name} - ${product.price}
    </div>
  );
}
// No useMemo, useCallback, or memo needed
```

## The `use` Hook

Read resources (Promises, Context) during render:

```tsx
import { use, Suspense } from 'react';

// Fetch function returns a Promise
async function fetchUser(id: string) {
  const res = await fetch(`/api/users/${id}`);
  return res.json();
}

function UserProfile({ userId }: { userId: string }) {
  // use() unwraps the Promise during render
  const user = use(fetchUser(userId));

  return (
    <div>
      <h1>{user.name}</h1>
      <p>{user.email}</p>
    </div>
  );
}

// Parent handles loading state
function App() {
  return (
    <Suspense fallback={<Loading />}>
      <UserProfile userId="123" />
    </Suspense>
  );
}
```

### use with Context

```tsx
import { use, createContext } from 'react';

const ThemeContext = createContext('light');

function ThemedButton() {
  // Can call use() conditionally (unlike useContext)
  if (someCondition) {
    const theme = use(ThemeContext);
    return <button className={theme}>Click</button>;
  }
  return <button>Default</button>;
}
```

## Actions API

### useActionState

Handle server-driven state with loading and errors:

```tsx
import { useActionState } from 'react';

async function createUser(prevState: State, formData: FormData) {
  'use server';

  const email = formData.get('email');

  try {
    const user = await db.user.create({ data: { email } });
    return { success: true, user };
  } catch (error) {
    return { success: false, error: 'Email already exists' };
  }
}

function SignupForm() {
  const [state, formAction, isPending] = useActionState(createUser, null);

  return (
    <form action={formAction}>
      <input name="email" type="email" disabled={isPending} />
      <button disabled={isPending}>
        {isPending ? 'Creating...' : 'Sign Up'}
      </button>
      {state?.error && <p className="error">{state.error}</p>}
      {state?.success && <p className="success">Welcome!</p>}
    </form>
  );
}
```

### useFormStatus

Access form submission state from child components:

```tsx
import { useFormStatus } from 'react-dom';

function SubmitButton() {
  const { pending, data, method, action } = useFormStatus();

  return (
    <button disabled={pending}>
      {pending ? 'Submitting...' : 'Submit'}
    </button>
  );
}

function ContactForm() {
  return (
    <form action={submitContact}>
      <input name="message" />
      <SubmitButton /> {/* Automatically knows form state */}
    </form>
  );
}
```

### useOptimistic

Instant UI updates before server confirmation:

```tsx
import { useOptimistic } from 'react';

function TodoList({ todos, addTodo }) {
  const [optimisticTodos, addOptimisticTodo] = useOptimistic(
    todos,
    (state, newTodo) => [...state, { ...newTodo, pending: true }]
  );

  async function handleSubmit(formData: FormData) {
    const title = formData.get('title');

    // Instantly show in UI
    addOptimisticTodo({ id: crypto.randomUUID(), title });

    // Server action runs in background
    await addTodo(formData);
  }

  return (
    <form action={handleSubmit}>
      <input name="title" />
      <button>Add</button>
      <ul>
        {optimisticTodos.map(todo => (
          <li key={todo.id} style={{ opacity: todo.pending ? 0.5 : 1 }}>
            {todo.title}
          </li>
        ))}
      </ul>
    </form>
  );
}
```

## Server Components (Stable)

Zero JavaScript sent to client:

```tsx
// app/products/page.tsx - Server Component by default
async function ProductsPage() {
  // Direct database access - no API needed
  const products = await db.product.findMany({
    include: { category: true }
  });

  return (
    <div>
      <h1>Products</h1>
      {products.map(p => (
        <ProductCard key={p.id} product={p} />
      ))}
      {/* Client interactivity where needed */}
      <AddToCartButton />
    </div>
  );
}

// components/AddToCartButton.tsx
'use client';

function AddToCartButton() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(c => c + 1)}>Add ({count})</button>;
}
```

## Document Metadata

Native support for `<title>`, `<meta>`, `<link>`:

```tsx
function BlogPost({ post }) {
  return (
    <article>
      <title>{post.title} | My Blog</title>
      <meta name="description" content={post.excerpt} />
      <meta property="og:image" content={post.image} />
      <link rel="canonical" href={`https://blog.com/${post.slug}`} />

      <h1>{post.title}</h1>
      <p>{post.content}</p>
    </article>
  );
}
// React hoists these to <head> automatically
```

## ref as a Prop

No more forwardRef boilerplate:

```tsx
// Before React 19
const Input = forwardRef((props, ref) => (
  <input ref={ref} {...props} />
));

// React 19
function Input({ ref, ...props }) {
  return <input ref={ref} {...props} />;
}
```

## React 19.2 Updates

```tsx
// New useId prefix for View Transitions compatibility
// Changed from :r: to _r_ for CSS selector validity

function Component() {
  const id = useId(); // Returns _r_1_, _r_2_, etc.
  return <div id={id}>...</div>;
}
```

## Migration Checklist

| Task | Status |
|------|--------|
| Remove unnecessary useMemo/useCallback | Compiler handles it |
| Replace forwardRef with ref prop | Native support |
| Migrate to useActionState for forms | New pattern |
| Add useOptimistic for instant feedback | Better UX |
| Use Server Components by default | Performance |

---

*Learned: December 20, 2025*
*Tags: React 19, Hooks, Server Components, Actions, Compiler, useOptimistic*
