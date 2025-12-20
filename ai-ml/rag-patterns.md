# RAG Patterns - Retrieval Augmented Generation

<div align="center">

![RAG](https://img.shields.io/badge/RAG-Retrieval_Augmented-purple?style=for-the-badge)
![AI](https://img.shields.io/badge/AI-LLM-blue?style=for-the-badge)
![Vector](https://img.shields.io/badge/Vector-Database-green?style=for-the-badge)

*Ground LLM responses in your own data for accurate, up-to-date answers.*

![RAG](https://upload.wikimedia.org/wikipedia/commons/0/04/ChatGPT_logo.svg)

</div>

## What is RAG?

```
Traditional LLM                RAG Pipeline
──────────────────────────     ──────────────────────────
Static knowledge               Dynamic knowledge
Hallucinations                 Grounded responses
No private data                Your own documents
Outdated info                  Always current
```

## Basic RAG Pipeline

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   Query     │───▶│   Embed     │───▶│   Search    │
└─────────────┘    └─────────────┘    └─────────────┘
                                            │
┌─────────────┐    ┌─────────────┐    ┌─────▼───────┐
│   Answer    │◀───│   Generate  │◀───│   Context   │
└─────────────┘    └─────────────┘    └─────────────┘
```

## Step 1: Document Processing

```typescript
// Chunk documents for embedding
interface Chunk {
  id: string
  content: string
  metadata: {
    source: string
    page?: number
    section?: string
  }
}

function chunkDocument(text: string, options = {
  chunkSize: 500,
  overlap: 50,
}): Chunk[] {
  const chunks: Chunk[] = []
  const sentences = text.split(/(?<=[.!?])\s+/)

  let currentChunk = ''
  let chunkIndex = 0

  for (const sentence of sentences) {
    if ((currentChunk + sentence).length > options.chunkSize) {
      chunks.push({
        id: `chunk-${chunkIndex}`,
        content: currentChunk.trim(),
        metadata: { source: 'document.pdf' },
      })

      // Keep overlap
      const words = currentChunk.split(' ')
      currentChunk = words.slice(-options.overlap).join(' ') + ' '
      chunkIndex++
    }
    currentChunk += sentence + ' '
  }

  if (currentChunk.trim()) {
    chunks.push({
      id: `chunk-${chunkIndex}`,
      content: currentChunk.trim(),
      metadata: { source: 'document.pdf' },
    })
  }

  return chunks
}
```

## Step 2: Generate Embeddings

```typescript
import OpenAI from 'openai'

const openai = new OpenAI()

async function embedText(text: string): Promise<number[]> {
  const response = await openai.embeddings.create({
    model: 'text-embedding-3-small',
    input: text,
  })
  return response.data[0].embedding
}

async function embedChunks(chunks: Chunk[]) {
  const embeddings = await Promise.all(
    chunks.map(async (chunk) => ({
      ...chunk,
      embedding: await embedText(chunk.content),
    }))
  )
  return embeddings
}
```

## Step 3: Vector Storage (Pinecone)

```typescript
import { Pinecone } from '@pinecone-database/pinecone'

const pinecone = new Pinecone({
  apiKey: process.env.PINECONE_API_KEY!,
})

const index = pinecone.index('documents')

// Store embeddings
async function storeEmbeddings(chunks: ChunkWithEmbedding[]) {
  const vectors = chunks.map((chunk) => ({
    id: chunk.id,
    values: chunk.embedding,
    metadata: {
      content: chunk.content,
      ...chunk.metadata,
    },
  }))

  await index.upsert(vectors)
}

// Search similar
async function searchSimilar(query: string, topK = 5) {
  const queryEmbedding = await embedText(query)

  const results = await index.query({
    vector: queryEmbedding,
    topK,
    includeMetadata: true,
  })

  return results.matches.map((match) => ({
    content: match.metadata?.content as string,
    score: match.score,
    source: match.metadata?.source as string,
  }))
}
```

## Step 4: Generate Response

```typescript
import Anthropic from '@anthropic-ai/sdk'

const anthropic = new Anthropic()

async function generateRAGResponse(query: string) {
  // 1. Retrieve relevant context
  const context = await searchSimilar(query, 5)

  // 2. Format context for prompt
  const contextText = context
    .map((c, i) => `[${i + 1}] ${c.content}`)
    .join('\n\n')

  // 3. Generate response with context
  const response = await anthropic.messages.create({
    model: 'claude-sonnet-4-20250514',
    max_tokens: 1024,
    system: `You are a helpful assistant. Answer questions based on the provided context.
      If the context doesn't contain relevant information, say so.
      Always cite your sources using [1], [2], etc.`,
    messages: [
      {
        role: 'user',
        content: `Context:\n${contextText}\n\nQuestion: ${query}`,
      },
    ],
  })

  return {
    answer: response.content[0].text,
    sources: context,
  }
}
```

## Advanced: Hybrid Search

```typescript
// Combine semantic + keyword search
async function hybridSearch(query: string) {
  // Semantic search (embeddings)
  const semanticResults = await searchSimilar(query, 10)

  // Keyword search (BM25)
  const keywordResults = await bm25Search(query, 10)

  // Reciprocal Rank Fusion
  const combined = reciprocalRankFusion([
    semanticResults,
    keywordResults,
  ])

  return combined.slice(0, 5)
}

function reciprocalRankFusion(
  resultSets: SearchResult[][],
  k = 60
): SearchResult[] {
  const scores = new Map<string, number>()

  for (const results of resultSets) {
    results.forEach((result, rank) => {
      const score = 1 / (k + rank + 1)
      const current = scores.get(result.id) || 0
      scores.set(result.id, current + score)
    })
  }

  return Array.from(scores.entries())
    .sort((a, b) => b[1] - a[1])
    .map(([id]) => findResultById(id))
}
```

## Advanced: Query Expansion

```typescript
// Generate multiple search queries
async function expandQuery(query: string): Promise<string[]> {
  const response = await anthropic.messages.create({
    model: 'claude-3-5-haiku-20241022',
    max_tokens: 200,
    system: 'Generate 3 alternative search queries. Return as JSON array.',
    messages: [{ role: 'user', content: query }],
  })

  const alternatives = JSON.parse(response.content[0].text)
  return [query, ...alternatives]
}

async function multiQuerySearch(query: string) {
  const queries = await expandQuery(query)

  const allResults = await Promise.all(
    queries.map((q) => searchSimilar(q, 3))
  )

  // Deduplicate and rank
  return deduplicateResults(allResults.flat())
}
```

## Advanced: Reranking

```typescript
// Rerank results with cross-encoder
async function rerankResults(
  query: string,
  results: SearchResult[]
): Promise<SearchResult[]> {
  const response = await anthropic.messages.create({
    model: 'claude-3-5-haiku-20241022',
    max_tokens: 500,
    system: `Score each document's relevance to the query (0-10).
      Return JSON: [{"id": "...", "score": N}, ...]`,
    messages: [
      {
        role: 'user',
        content: `Query: ${query}\n\nDocuments:\n${
          results.map((r, i) => `[${i}] ${r.content}`).join('\n')
        }`,
      },
    ],
  })

  const scores = JSON.parse(response.content[0].text)

  return results
    .map((r, i) => ({ ...r, relevanceScore: scores[i].score }))
    .sort((a, b) => b.relevanceScore - a.relevanceScore)
}
```

## Full RAG Pipeline

```typescript
class RAGPipeline {
  async ingest(documents: Document[]) {
    // 1. Chunk documents
    const chunks = documents.flatMap((doc) =>
      chunkDocument(doc.content, { chunkSize: 500 })
    )

    // 2. Generate embeddings
    const embedded = await embedChunks(chunks)

    // 3. Store in vector DB
    await storeEmbeddings(embedded)
  }

  async query(question: string) {
    // 1. Expand query
    const queries = await expandQuery(question)

    // 2. Hybrid search
    const results = await hybridSearch(queries)

    // 3. Rerank
    const reranked = await rerankResults(question, results)

    // 4. Generate response
    return generateRAGResponse(question, reranked.slice(0, 5))
  }
}
```

## Vector Databases Comparison

| Database | Best For |
|----------|----------|
| Pinecone | Managed, scale |
| Weaviate | Hybrid search |
| Qdrant | Self-hosted |
| Chroma | Local development |
| pgvector | Postgres users |

---

*Learned: December 20, 2025*
*Tags: RAG, LLM, Vector Database, Embeddings, AI*
