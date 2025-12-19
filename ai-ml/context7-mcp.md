# Context7 MCP - Real-Time Documentation for AI Coding

Up-to-date, version-specific documentation directly in your AI prompts.

## What is Context7?

```
┌─────────────────────────────────────────────────────────────────┐
│                    CONTEXT7 ARCHITECTURE                         │
│                                                                 │
│   ┌──────────────┐        ┌──────────────┐                     │
│   │  AI Editor   │        │  Context7    │                     │
│   │  (Cursor/    │◄──────►│  MCP Server  │                     │
│   │   Claude)    │  MCP   │              │                     │
│   └──────────────┘        └──────────────┘                     │
│         │                        │                              │
│         │                        ▼                              │
│         ▼                 ┌──────────────┐                     │
│   ┌──────────────┐        │  33,000+     │                     │
│   │   Context    │        │  Libraries   │                     │
│   │   Window     │◄───────│  - React     │                     │
│   │   (Prompt)   │  Docs  │  - Next.js   │                     │
│   └──────────────┘        │  - Prisma    │                     │
│                           │  - And more  │                     │
│                           └──────────────┘                     │
└─────────────────────────────────────────────────────────────────┘
```

## Why Context7?

Traditional AI assistants rely on training data that becomes outdated. Context7 solves this:

| Problem | Context7 Solution |
|---------|-------------------|
| Outdated API references | Fetches current docs from source |
| Hallucinated functions | Provides real, verified APIs |
| Wrong version syntax | Version-specific documentation |
| Missing new features | Always up-to-date |

## Installation

### Claude Code

```bash
claude mcp add context7 -- npx -y @upstash/context7-mcp
```

### Cursor

```json
// ~/.cursor/mcp.json
{
  "mcpServers": {
    "context7": {
      "command": "npx",
      "args": ["-y", "@upstash/context7-mcp"]
    }
  }
}
```

### With API Key (Higher Rate Limits)

```json
{
  "mcpServers": {
    "context7": {
      "command": "npx",
      "args": ["-y", "@upstash/context7-mcp"],
      "env": {
        "CONTEXT7_API_KEY": "your-api-key"
      }
    }
  }
}
```

### HTTP Transport

```bash
claude mcp add --transport http context7 https://mcp.context7.com/mcp
```

## Available Tools

### 1. resolve-library-id

Finds the correct Context7 library ID for a package:

```typescript
// Input
{ "libraryName": "next" }

// Output
[
  { "id": "/vercel/next.js", "name": "Next.js", "version": "15.1.0" },
  { "id": "/vercel/next.js/14", "name": "Next.js 14", "version": "14.2.x" }
]
```

### 2. get-library-docs

Fetches documentation for a specific library:

```typescript
// Input
{
  "libraryId": "/vercel/next.js",
  "topic": "server actions"
}

// Output: Current Server Actions documentation with examples
```

## Usage Patterns

### Basic Usage

Simply add "use context7" to your prompt:

```
Create a Next.js API route with rate limiting. use context7
```

### Specify Library Version

```
Help me migrate from React 18 to React 19 useFormStatus. use context7 /facebook/react/19
```

### Multiple Libraries

```
Build a form with Prisma + Zod validation. use context7
```

## Real-World Examples

### Before Context7

```typescript
// AI might suggest outdated or hallucinated API
import { useForm } from 'react-hook-form';

const form = useForm({
  resolver: zodResolver(schema),
  defaultState: {} // Wrong! It's defaultValues
});
```

### With Context7

```typescript
// Correct, version-specific implementation
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';

const form = useForm({
  resolver: zodResolver(schema),
  defaultValues: {
    email: '',
    password: ''
  }
});
```

## Supported Libraries

Context7 indexes 33,000+ libraries including:

| Category | Libraries |
|----------|-----------|
| Frontend | React, Next.js, Vue, Svelte, Angular |
| Backend | Express, Fastify, Hono, tRPC |
| Database | Prisma, Drizzle, Mongoose, TypeORM |
| AI/ML | LangChain, Vercel AI SDK, OpenAI |
| Testing | Jest, Vitest, Playwright, Cypress |
| Styling | Tailwind CSS, Shadcn/ui, Radix |

## Integration with AI Workflows

### Cursor Composer

```
@context7 How do I implement streaming with Vercel AI SDK 4.1?
```

### Claude Code

```bash
# Context7 automatically provides docs when detected
claude "Create a Prisma schema for a blog with categories. use context7"
```

### VS Code + Continue

```json
{
  "models": [{
    "title": "Claude with Context7",
    "provider": "anthropic",
    "model": "claude-sonnet-4-20250514",
    "mcpServers": ["context7"]
  }]
}
```

## API Key Benefits

| Feature | Free | With API Key |
|---------|------|--------------|
| Rate Limit | 100 req/hour | 10,000 req/hour |
| Private Repos | No | Yes |
| Priority Queue | No | Yes |

Get your API key at [context7.com/dashboard](https://context7.com/dashboard)

## Best Practices

1. **Be Specific**: Include library name and topic in your prompt
2. **Version Matters**: Specify version when working with breaking changes
3. **Combine Libraries**: Context7 handles multi-library queries
4. **Trust the Docs**: Context7 fetches from official sources

---

*Learned: December 20, 2025*
*Tags: Context7, MCP, AI, Documentation, Cursor, Claude Code, Upstash*
