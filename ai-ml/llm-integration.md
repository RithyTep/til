# LLM Integration Patterns

<div align="center">

![AI](https://img.shields.io/badge/AI-LLM_Integration-purple?style=for-the-badge&logo=openai&logoColor=white)
![Claude](https://img.shields.io/badge/Claude-Anthropic-orange?style=for-the-badge)

*Production patterns for AI/LLM-powered applications.*

</div>

## Structured Output with Function Calling

```typescript
import Anthropic from '@anthropic-ai/sdk';
import { z } from 'zod';

// Define structured output schema
const ProductAnalysisSchema = z.object({
  sentiment: z.enum(['positive', 'negative', 'neutral']),
  score: z.number().min(0).max(1),
  keyTopics: z.array(z.string()),
  actionItems: z.array(z.object({
    priority: z.enum(['high', 'medium', 'low']),
    description: z.string(),
    department: z.string(),
  })),
  summary: z.string(),
});

type ProductAnalysis = z.infer<typeof ProductAnalysisSchema>;

const anthropic = new Anthropic();

async function analyzeReview(review: string): Promise<ProductAnalysis> {
  const response = await anthropic.messages.create({
    model: 'claude-sonnet-4-20250514',
    max_tokens: 1024,
    tools: [{
      name: 'analyze_review',
      description: 'Analyze a product review and extract structured insights',
      input_schema: {
        type: 'object',
        properties: {
          sentiment: { type: 'string', enum: ['positive', 'negative', 'neutral'] },
          score: { type: 'number', minimum: 0, maximum: 1 },
          keyTopics: { type: 'array', items: { type: 'string' } },
          actionItems: {
            type: 'array',
            items: {
              type: 'object',
              properties: {
                priority: { type: 'string', enum: ['high', 'medium', 'low'] },
                description: { type: 'string' },
                department: { type: 'string' },
              },
              required: ['priority', 'description', 'department'],
            },
          },
          summary: { type: 'string' },
        },
        required: ['sentiment', 'score', 'keyTopics', 'actionItems', 'summary'],
      },
    }],
    tool_choice: { type: 'tool', name: 'analyze_review' },
    messages: [{
      role: 'user',
      content: `Analyze this product review:\n\n${review}`,
    }],
  });

  const toolUse = response.content.find(c => c.type === 'tool_use');
  if (!toolUse || toolUse.type !== 'tool_use') {
    throw new Error('No tool response');
  }

  return ProductAnalysisSchema.parse(toolUse.input);
}
```

## RAG (Retrieval-Augmented Generation)

```typescript
import { Pinecone } from '@pinecone-database/pinecone';
import Anthropic from '@anthropic-ai/sdk';

interface Document {
  id: string;
  content: string;
  metadata: Record<string, string>;
}

class RAGPipeline {
  private pinecone: Pinecone;
  private anthropic: Anthropic;
  private index: ReturnType<Pinecone['index']>;

  constructor() {
    this.pinecone = new Pinecone();
    this.anthropic = new Anthropic();
    this.index = this.pinecone.index('knowledge-base');
  }

  async embed(text: string): Promise<number[]> {
    const response = await this.anthropic.messages.create({
      model: 'claude-sonnet-4-20250514',
      max_tokens: 1,
      messages: [{ role: 'user', content: text }],
    });
    // Use embedding model in production (e.g., voyage-3)
    return this.getEmbedding(text);
  }

  async ingest(documents: Document[]): Promise<void> {
    const vectors = await Promise.all(
      documents.map(async (doc) => ({
        id: doc.id,
        values: await this.embed(doc.content),
        metadata: {
          ...doc.metadata,
          content: doc.content.slice(0, 1000), // Store truncated content
        },
      }))
    );

    // Batch upsert
    const batchSize = 100;
    for (let i = 0; i < vectors.length; i += batchSize) {
      await this.index.upsert(vectors.slice(i, i + batchSize));
    }
  }

  async query(question: string, topK = 5): Promise<string> {
    // 1. Retrieve relevant documents
    const queryVector = await this.embed(question);
    const results = await this.index.query({
      vector: queryVector,
      topK,
      includeMetadata: true,
    });

    const context = results.matches
      .map((m, i) => `[${i + 1}] ${m.metadata?.content}`)
      .join('\n\n');

    // 2. Generate response with context
    const response = await this.anthropic.messages.create({
      model: 'claude-sonnet-4-20250514',
      max_tokens: 2048,
      system: `You are a helpful assistant. Answer questions based on the provided context.
If the context doesn't contain relevant information, say so.
Always cite your sources using [1], [2], etc.`,
      messages: [{
        role: 'user',
        content: `Context:\n${context}\n\nQuestion: ${question}`,
      }],
    });

    return response.content[0].type === 'text' ? response.content[0].text : '';
  }
}

// Hybrid search with reranking
class HybridRAG extends RAGPipeline {
  async query(question: string): Promise<string> {
    // Semantic search
    const semanticResults = await this.semanticSearch(question);

    // Keyword search (BM25)
    const keywordResults = await this.keywordSearch(question);

    // Reciprocal Rank Fusion
    const fusedResults = this.reciprocalRankFusion([
      semanticResults,
      keywordResults,
    ]);

    // Rerank with cross-encoder
    const reranked = await this.rerank(question, fusedResults);

    return this.generate(question, reranked.slice(0, 5));
  }

  private reciprocalRankFusion(
    resultSets: string[][],
    k = 60
  ): Array<{ id: string; score: number }> {
    const scores = new Map<string, number>();

    for (const results of resultSets) {
      results.forEach((id, rank) => {
        const rrf = 1 / (k + rank + 1);
        scores.set(id, (scores.get(id) ?? 0) + rrf);
      });
    }

    return Array.from(scores.entries())
      .map(([id, score]) => ({ id, score }))
      .sort((a, b) => b.score - a.score);
  }
}
```

## Streaming with Backpressure

```typescript
import { Anthropic } from '@anthropic-ai/sdk';

async function* streamResponse(
  prompt: string
): AsyncGenerator<string, void, unknown> {
  const anthropic = new Anthropic();

  const stream = await anthropic.messages.stream({
    model: 'claude-sonnet-4-20250514',
    max_tokens: 4096,
    messages: [{ role: 'user', content: prompt }],
  });

  for await (const event of stream) {
    if (event.type === 'content_block_delta' && event.delta.type === 'text_delta') {
      yield event.delta.text;
    }
  }
}

// Server-Sent Events endpoint
app.get('/api/chat/stream', async (req, res) => {
  res.setHeader('Content-Type', 'text/event-stream');
  res.setHeader('Cache-Control', 'no-cache');
  res.setHeader('Connection', 'keep-alive');

  const prompt = req.query.prompt as string;

  try {
    for await (const chunk of streamResponse(prompt)) {
      res.write(`data: ${JSON.stringify({ text: chunk })}\n\n`);

      // Handle client disconnect
      if (res.writableEnded) break;
    }
    res.write('data: [DONE]\n\n');
  } catch (error) {
    res.write(`data: ${JSON.stringify({ error: error.message })}\n\n`);
  } finally {
    res.end();
  }
});

// Client-side consumption
async function consumeStream(prompt: string, onChunk: (text: string) => void) {
  const response = await fetch(`/api/chat/stream?prompt=${encodeURIComponent(prompt)}`);
  const reader = response.body?.getReader();
  const decoder = new TextDecoder();

  while (reader) {
    const { done, value } = await reader.read();
    if (done) break;

    const chunk = decoder.decode(value);
    const lines = chunk.split('\n');

    for (const line of lines) {
      if (line.startsWith('data: ')) {
        const data = line.slice(6);
        if (data === '[DONE]') return;

        try {
          const { text, error } = JSON.parse(data);
          if (error) throw new Error(error);
          if (text) onChunk(text);
        } catch (e) {
          // Skip malformed lines
        }
      }
    }
  }
}
```

## Caching & Rate Limiting

```typescript
import { Redis } from 'ioredis';
import crypto from 'crypto';

class LLMCache {
  constructor(private redis: Redis) {}

  private hashPrompt(prompt: string, model: string): string {
    return crypto
      .createHash('sha256')
      .update(`${model}:${prompt}`)
      .digest('hex');
  }

  async get(prompt: string, model: string): Promise<string | null> {
    const key = `llm:cache:${this.hashPrompt(prompt, model)}`;
    return this.redis.get(key);
  }

  async set(prompt: string, model: string, response: string, ttl = 3600): Promise<void> {
    const key = `llm:cache:${this.hashPrompt(prompt, model)}`;
    await this.redis.setex(key, ttl, response);
  }
}

// Token bucket rate limiter
class RateLimiter {
  constructor(
    private redis: Redis,
    private tokensPerMinute: number = 100000,
    private requestsPerMinute: number = 1000
  ) {}

  async acquire(userId: string, estimatedTokens: number): Promise<boolean> {
    const now = Date.now();
    const minute = Math.floor(now / 60000);
    const tokenKey = `ratelimit:tokens:${userId}:${minute}`;
    const requestKey = `ratelimit:requests:${userId}:${minute}`;

    const [tokens, requests] = await this.redis
      .multi()
      .incrby(tokenKey, estimatedTokens)
      .incr(requestKey)
      .expire(tokenKey, 120)
      .expire(requestKey, 120)
      .exec();

    const currentTokens = tokens?.[1] as number;
    const currentRequests = requests?.[1] as number;

    return currentTokens <= this.tokensPerMinute &&
           currentRequests <= this.requestsPerMinute;
  }
}

// Semantic caching (cache similar prompts)
class SemanticCache {
  constructor(
    private redis: Redis,
    private vectorDB: VectorDB,
    private similarityThreshold = 0.95
  ) {}

  async get(prompt: string): Promise<string | null> {
    const embedding = await this.embed(prompt);
    const results = await this.vectorDB.search(embedding, 1);

    if (results[0]?.score >= this.similarityThreshold) {
      return this.redis.get(`llm:semantic:${results[0].id}`);
    }
    return null;
  }

  async set(prompt: string, response: string): Promise<void> {
    const id = crypto.randomUUID();
    const embedding = await this.embed(prompt);

    await Promise.all([
      this.vectorDB.upsert({ id, values: embedding }),
      this.redis.setex(`llm:semantic:${id}`, 86400, response),
    ]);
  }
}
```

---

*Learned: December 20, 2025*
*Tags: LLM, AI, RAG, Claude, OpenAI, Vector Database*
