# Server Actions for Form Handling

Server Actions let you run server code directly from forms without API routes.

## Basic Server Action

```tsx
// app/actions.ts
'use server';

export async function createPost(formData: FormData) {
  const title = formData.get('title') as string;
  const content = formData.get('content') as string;

  await db.post.create({
    data: { title, content }
  });

  revalidatePath('/posts');
}
```

## Using in a Form

```tsx
// app/new-post/page.tsx
import { createPost } from './actions';

export default function NewPostPage() {
  return (
    <form action={createPost}>
      <input name="title" placeholder="Title" required />
      <textarea name="content" placeholder="Content" required />
      <button type="submit">Create Post</button>
    </form>
  );
}
```

## With Validation & Error Handling

```tsx
'use server';

import { z } from 'zod';

const schema = z.object({
  email: z.string().email(),
  password: z.string().min(8),
});

export async function signup(formData: FormData) {
  const result = schema.safeParse({
    email: formData.get('email'),
    password: formData.get('password'),
  });

  if (!result.success) {
    return { error: result.error.flatten() };
  }

  // Create user...
  return { success: true };
}
```

## Pending State with useFormStatus

```tsx
'use client';

import { useFormStatus } from 'react-dom';

function SubmitButton() {
  const { pending } = useFormStatus();

  return (
    <button disabled={pending}>
      {pending ? 'Submitting...' : 'Submit'}
    </button>
  );
}
```

## Optimistic Updates

```tsx
'use client';

import { useOptimistic } from 'react';

function TodoList({ todos, addTodo }) {
  const [optimisticTodos, addOptimisticTodo] = useOptimistic(
    todos,
    (state, newTodo) => [...state, newTodo]
  );

  async function handleSubmit(formData) {
    const title = formData.get('title');
    addOptimisticTodo({ title, pending: true });
    await addTodo(formData);
  }

  return (
    <form action={handleSubmit}>
      {/* ... */}
    </form>
  );
}
```

---

📅 *Learned: 2024*
🏷️ *Tags: Next.js, Server Actions, Forms*
