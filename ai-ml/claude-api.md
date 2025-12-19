# Claude API - Building with Anthropic's AI

<div align="center">

![Claude](https://img.shields.io/badge/Claude-191919?style=for-the-badge&logo=anthropic&logoColor=white)
![API](https://img.shields.io/badge/API-Anthropic-orange?style=for-the-badge)
![AI](https://img.shields.io/badge/AI-LLM-blue?style=for-the-badge)

*Build intelligent applications with Claude's advanced reasoning and 200K context window.*

</div>

## Why Claude?

```
GPT-4                          Claude 3.5/Opus
──────────────────────────     ──────────────────────────
128K context                   200K context
Function calling               Tool use (similar)
JSON mode                      Structured outputs
Vision                         Vision + PDF analysis
Code interpreter               Artifacts (Claude.ai)
```

## Installation

```bash
npm install @anthropic-ai/sdk
```

## Basic Usage

```typescript
import Anthropic from '@anthropic-ai/sdk'

const anthropic = new Anthropic({
  apiKey: process.env.ANTHROPIC_API_KEY,
})

const message = await anthropic.messages.create({
  model: 'claude-sonnet-4-20250514',
  max_tokens: 1024,
  messages: [
    { role: 'user', content: 'Explain quantum computing in simple terms.' }
  ],
})

console.log(message.content[0].text)
```

## Models

```typescript
// Available models (as of 2025)
const models = {
  'claude-sonnet-4-20250514': 'Best balance of speed and intelligence',
  'claude-opus-4-20250514': 'Most capable, complex tasks',
  'claude-3-5-haiku-20241022': 'Fastest, simple tasks',
}

// Model selection based on task
function selectModel(task: 'simple' | 'balanced' | 'complex') {
  switch (task) {
    case 'simple': return 'claude-3-5-haiku-20241022'
    case 'balanced': return 'claude-sonnet-4-20250514'
    case 'complex': return 'claude-opus-4-20250514'
  }
}
```

## Streaming Responses

```typescript
import Anthropic from '@anthropic-ai/sdk'

const anthropic = new Anthropic()

// Stream for real-time output
const stream = await anthropic.messages.stream({
  model: 'claude-sonnet-4-20250514',
  max_tokens: 1024,
  messages: [{ role: 'user', content: 'Write a short story.' }],
})

for await (const event of stream) {
  if (event.type === 'content_block_delta' &&
      event.delta.type === 'text_delta') {
    process.stdout.write(event.delta.text)
  }
}

// Or use helper
const stream2 = anthropic.messages.stream({...})
stream2.on('text', (text) => process.stdout.write(text))
await stream2.finalMessage()
```

## System Prompts

```typescript
const message = await anthropic.messages.create({
  model: 'claude-sonnet-4-20250514',
  max_tokens: 1024,
  system: `You are a helpful coding assistant.
    - Always provide TypeScript examples
    - Include error handling
    - Explain your reasoning`,
  messages: [
    { role: 'user', content: 'How do I fetch data in React?' }
  ],
})
```

## Multi-Turn Conversations

```typescript
const conversationHistory: Anthropic.MessageParam[] = []

async function chat(userMessage: string) {
  conversationHistory.push({
    role: 'user',
    content: userMessage,
  })

  const response = await anthropic.messages.create({
    model: 'claude-sonnet-4-20250514',
    max_tokens: 1024,
    system: 'You are a helpful assistant.',
    messages: conversationHistory,
  })

  const assistantMessage = response.content[0].text

  conversationHistory.push({
    role: 'assistant',
    content: assistantMessage,
  })

  return assistantMessage
}

// Usage
await chat('What is TypeScript?')
await chat('How does it compare to JavaScript?') // Has context
```

## Vision (Image Analysis)

```typescript
import fs from 'fs'

// From file
const imageData = fs.readFileSync('image.png').toString('base64')

const message = await anthropic.messages.create({
  model: 'claude-sonnet-4-20250514',
  max_tokens: 1024,
  messages: [
    {
      role: 'user',
      content: [
        {
          type: 'image',
          source: {
            type: 'base64',
            media_type: 'image/png',
            data: imageData,
          },
        },
        {
          type: 'text',
          text: 'Describe this image in detail.',
        },
      ],
    },
  ],
})

// From URL
const messageFromUrl = await anthropic.messages.create({
  model: 'claude-sonnet-4-20250514',
  max_tokens: 1024,
  messages: [
    {
      role: 'user',
      content: [
        {
          type: 'image',
          source: {
            type: 'url',
            url: 'https://example.com/image.jpg',
          },
        },
        { type: 'text', text: 'What do you see?' },
      ],
    },
  ],
})
```

## Tool Use (Function Calling)

```typescript
const tools: Anthropic.Tool[] = [
  {
    name: 'get_weather',
    description: 'Get current weather for a location',
    input_schema: {
      type: 'object',
      properties: {
        location: {
          type: 'string',
          description: 'City name, e.g., "San Francisco, CA"',
        },
        unit: {
          type: 'string',
          enum: ['celsius', 'fahrenheit'],
          description: 'Temperature unit',
        },
      },
      required: ['location'],
    },
  },
]

const message = await anthropic.messages.create({
  model: 'claude-sonnet-4-20250514',
  max_tokens: 1024,
  tools,
  messages: [
    { role: 'user', content: 'What is the weather in Tokyo?' }
  ],
})

// Handle tool use
if (message.stop_reason === 'tool_use') {
  const toolUse = message.content.find(block => block.type === 'tool_use')

  if (toolUse && toolUse.name === 'get_weather') {
    const weather = await fetchWeather(toolUse.input.location)

    // Continue conversation with tool result
    const finalResponse = await anthropic.messages.create({
      model: 'claude-sonnet-4-20250514',
      max_tokens: 1024,
      tools,
      messages: [
        { role: 'user', content: 'What is the weather in Tokyo?' },
        { role: 'assistant', content: message.content },
        {
          role: 'user',
          content: [
            {
              type: 'tool_result',
              tool_use_id: toolUse.id,
              content: JSON.stringify(weather),
            },
          ],
        },
      ],
    })
  }
}
```

## Structured Output

```typescript
// Force JSON output with system prompt
const message = await anthropic.messages.create({
  model: 'claude-sonnet-4-20250514',
  max_tokens: 1024,
  system: `Return responses as valid JSON only. No markdown, no explanation.`,
  messages: [
    {
      role: 'user',
      content: `Extract info from: "John Doe, 30 years old, engineer at Google"
        Format: { "name": string, "age": number, "job": string, "company": string }`
    }
  ],
})

const data = JSON.parse(message.content[0].text)
```

## Error Handling

```typescript
import Anthropic, { APIError, RateLimitError } from '@anthropic-ai/sdk'

try {
  const message = await anthropic.messages.create({...})
} catch (error) {
  if (error instanceof RateLimitError) {
    // Retry with exponential backoff
    await sleep(error.headers['retry-after'] * 1000)
  } else if (error instanceof APIError) {
    console.error(`API Error: ${error.status} - ${error.message}`)
  } else {
    throw error
  }
}
```

## Next.js API Route

```typescript
// app/api/chat/route.ts
import Anthropic from '@anthropic-ai/sdk'
import { NextRequest } from 'next/server'

const anthropic = new Anthropic()

export async function POST(req: NextRequest) {
  const { message } = await req.json()

  const stream = await anthropic.messages.stream({
    model: 'claude-sonnet-4-20250514',
    max_tokens: 1024,
    messages: [{ role: 'user', content: message }],
  })

  return new Response(stream.toReadableStream(), {
    headers: { 'Content-Type': 'text/event-stream' },
  })
}
```

## Best Practices

```typescript
// 1. Use appropriate model for task
// 2. Set reasonable max_tokens
// 3. Use system prompts for consistent behavior
// 4. Handle rate limits gracefully
// 5. Stream for better UX on long responses
// 6. Cache responses when appropriate
```

---

*Learned: December 20, 2025*
*Tags: Claude, Anthropic, API, LLM, AI*
