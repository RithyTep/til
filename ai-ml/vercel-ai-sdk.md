# Vercel AI SDK 4.1 - Building Production AI Applications

Unified API for LLMs with streaming, tool calling, and React integration.

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                    VERCEL AI SDK STACK                           │
│                                                                 │
│   ┌──────────────────────────────────────────────────────────┐ │
│   │                    AI SDK UI                              │ │
│   │  useChat()  useCompletion()  useObject()  useAssistant() │ │
│   └──────────────────────────────────────────────────────────┘ │
│                            │                                    │
│                            ▼                                    │
│   ┌──────────────────────────────────────────────────────────┐ │
│   │                   AI SDK Core                             │ │
│   │   generateText()  streamText()  generateObject()         │ │
│   │   streamObject()  embed()  embedMany()                   │ │
│   └──────────────────────────────────────────────────────────┘ │
│                            │                                    │
│                            ▼                                    │
│   ┌──────────────────────────────────────────────────────────┐ │
│   │                  Provider Layer                           │ │
│   │  OpenAI  Anthropic  Google  Mistral  Cohere  Together    │ │
│   └──────────────────────────────────────────────────────────┘ │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## Installation

```bash
npm install ai @ai-sdk/openai @ai-sdk/anthropic
```

## Basic Usage

### Text Generation

```typescript
import { generateText } from 'ai';
import { openai } from '@ai-sdk/openai';

const { text } = await generateText({
  model: openai('gpt-4o'),
  prompt: 'Explain quantum computing in one paragraph.',
});

console.log(text);
```

### Streaming Text

```typescript
import { streamText } from 'ai';
import { anthropic } from '@ai-sdk/anthropic';

const result = streamText({
  model: anthropic('claude-sonnet-4-20250514'),
  prompt: 'Write a poem about TypeScript.',
});

for await (const chunk of result.textStream) {
  process.stdout.write(chunk);
}
```

## React Integration

### useChat Hook

```tsx
// app/chat/page.tsx
'use client';

import { useChat } from 'ai/react';

export default function ChatPage() {
  const { messages, input, handleInputChange, handleSubmit, isLoading } =
    useChat({
      api: '/api/chat',
    });

  return (
    <div className="flex flex-col h-screen">
      <div className="flex-1 overflow-y-auto p-4 space-y-4">
        {messages.map((m) => (
          <div
            key={m.id}
            className={`p-3 rounded-lg ${
              m.role === 'user' ? 'bg-blue-100 ml-auto' : 'bg-gray-100'
            } max-w-[80%]`}
          >
            {m.content}
          </div>
        ))}
      </div>

      <form onSubmit={handleSubmit} className="p-4 border-t">
        <input
          value={input}
          onChange={handleInputChange}
          placeholder="Type a message..."
          disabled={isLoading}
          className="w-full p-2 border rounded"
        />
      </form>
    </div>
  );
}
```

### API Route

```typescript
// app/api/chat/route.ts
import { streamText } from 'ai';
import { openai } from '@ai-sdk/openai';

export async function POST(req: Request) {
  const { messages } = await req.json();

  const result = streamText({
    model: openai('gpt-4o'),
    messages,
    system: 'You are a helpful assistant.',
  });

  return result.toDataStreamResponse();
}
```

## Tool Calling

### Define Tools

```typescript
import { streamText, tool } from 'ai';
import { openai } from '@ai-sdk/openai';
import { z } from 'zod';

const result = streamText({
  model: openai('gpt-4o'),
  tools: {
    weather: tool({
      description: 'Get the weather for a location',
      parameters: z.object({
        location: z.string().describe('City name'),
        unit: z.enum(['celsius', 'fahrenheit']).default('celsius'),
      }),
      execute: async ({ location, unit }) => {
        // Call weather API
        const data = await fetchWeather(location);
        return {
          temperature: unit === 'celsius' ? data.tempC : data.tempF,
          condition: data.condition,
          location,
        };
      },
    }),

    searchDatabase: tool({
      description: 'Search the product database',
      parameters: z.object({
        query: z.string(),
        limit: z.number().default(10),
      }),
      execute: async ({ query, limit }) => {
        const results = await db.product.findMany({
          where: { name: { contains: query } },
          take: limit,
        });
        return results;
      },
    }),
  },
  prompt: 'What is the weather in Tokyo and show me 5 electronics products?',
});
```

### Multi-step Tool Execution

```typescript
import { generateText } from 'ai';

const { text, toolCalls, toolResults } = await generateText({
  model: openai('gpt-4o'),
  tools: { weather, searchDatabase },
  maxSteps: 5, // Allow multiple tool calls
  prompt: 'Plan a trip to Tokyo - check weather and find hotels.',
  onStepFinish: ({ text, toolCalls, toolResults }) => {
    console.log('Step completed:', { text, toolCalls, toolResults });
  },
});
```

## Structured Output

### Generate Objects

```typescript
import { generateObject } from 'ai';
import { openai } from '@ai-sdk/openai';
import { z } from 'zod';

const ProductSchema = z.object({
  name: z.string(),
  description: z.string(),
  price: z.number(),
  features: z.array(z.string()),
  category: z.enum(['electronics', 'clothing', 'home']),
});

const { object } = await generateObject({
  model: openai('gpt-4o'),
  schema: ProductSchema,
  prompt: 'Generate a product listing for a wireless mouse.',
});

// object is fully typed as z.infer<typeof ProductSchema>
console.log(object.name, object.price);
```

### Stream Objects

```typescript
import { streamObject } from 'ai';

const result = streamObject({
  model: openai('gpt-4o'),
  schema: ProductSchema,
  prompt: 'Generate a product listing for a mechanical keyboard.',
});

for await (const partialObject of result.partialObjectStream) {
  // Receive partial objects as they stream
  console.log(partialObject);
}

const finalObject = await result.object;
```

## Smooth Streaming (AI SDK 4.1)

```typescript
import { streamText, smoothStream } from 'ai';

const result = streamText({
  model: openai('gpt-4o'),
  prompt: 'Explain machine learning.',
  experimental_transform: smoothStream({
    chunking: 'word', // 'character' | 'word' | 'line'
    delayInMs: 20,
  }),
});

// Smoother streaming experience for users
for await (const chunk of result.textStream) {
  process.stdout.write(chunk);
}
```

## Provider Switching

```typescript
import { generateText } from 'ai';
import { openai } from '@ai-sdk/openai';
import { anthropic } from '@ai-sdk/anthropic';
import { google } from '@ai-sdk/google';

// Same code works with any provider
async function chat(provider: 'openai' | 'anthropic' | 'google', prompt: string) {
  const models = {
    openai: openai('gpt-4o'),
    anthropic: anthropic('claude-sonnet-4-20250514'),
    google: google('gemini-2.0-flash'),
  };

  const { text } = await generateText({
    model: models[provider],
    prompt,
  });

  return text;
}
```

## Embeddings

```typescript
import { embed, embedMany } from 'ai';
import { openai } from '@ai-sdk/openai';

// Single embedding
const { embedding } = await embed({
  model: openai.embedding('text-embedding-3-small'),
  value: 'What is TypeScript?',
});

// Batch embeddings
const { embeddings } = await embedMany({
  model: openai.embedding('text-embedding-3-small'),
  values: ['TypeScript basics', 'React hooks', 'Next.js routing'],
});

// Use for RAG, similarity search, etc.
```

## RAG Pattern

```typescript
import { generateText, embed } from 'ai';
import { openai } from '@ai-sdk/openai';

async function ragQuery(question: string) {
  // 1. Embed the question
  const { embedding } = await embed({
    model: openai.embedding('text-embedding-3-small'),
    value: question,
  });

  // 2. Find relevant documents
  const documents = await vectorStore.similaritySearch(embedding, 5);

  // 3. Generate answer with context
  const { text } = await generateText({
    model: openai('gpt-4o'),
    system: `Answer based on the following context:\n${documents.map(d => d.content).join('\n\n')}`,
    prompt: question,
  });

  return text;
}
```

## Error Handling

```typescript
import { generateText, APICallError, InvalidPromptError } from 'ai';

try {
  const { text } = await generateText({
    model: openai('gpt-4o'),
    prompt: userInput,
  });
} catch (error) {
  if (error instanceof APICallError) {
    console.error('API Error:', error.message, error.statusCode);
  } else if (error instanceof InvalidPromptError) {
    console.error('Invalid prompt:', error.message);
  } else {
    throw error;
  }
}
```

---

*Learned: December 20, 2025*
*Tags: Vercel AI SDK, LLM, Streaming, RAG, Tool Calling, React, TypeScript*
