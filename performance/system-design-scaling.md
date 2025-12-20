# System Design & Scaling Patterns

<div align="center">

![System Design](https://img.shields.io/badge/System_Design-Scaling-blue?style=for-the-badge&logo=amazonaws&logoColor=white)
![Pattern](https://img.shields.io/badge/Pattern-Sharding_|_Caching_|_Queues-orange?style=for-the-badge)

*Architecture patterns for handling massive scale.*

![System Design](https://upload.wikimedia.org/wikipedia/commons/e/e9/Client-server-model.svg)

</div>

## Load Balancing Strategies

```
┌─────────────────────────────────────────────────────────────────┐
│                    LOAD BALANCING LAYERS                        │
├─────────────────────────────────────────────────────────────────┤
│  DNS (GeoDNS)     → Global distribution, failover              │
│       ↓                                                         │
│  CDN (Edge)       → Static content, edge compute               │
│       ↓                                                         │
│  L4 (TCP/UDP)     → High performance, connection-level        │
│       ↓                                                         │
│  L7 (HTTP)        → Content-based routing, SSL termination    │
│       ↓                                                         │
│  Service Mesh     → Service-to-service, mTLS                   │
└─────────────────────────────────────────────────────────────────┘
```

```typescript
// Consistent hashing for distributed cache
class ConsistentHash<T> {
  private ring: Map<number, T> = new Map();
  private sortedKeys: number[] = [];
  private virtualNodes: number;

  constructor(nodes: T[], virtualNodes = 150) {
    this.virtualNodes = virtualNodes;
    nodes.forEach(node => this.addNode(node));
  }

  private hash(key: string): number {
    // MurmurHash3 or similar
    let h = 0;
    for (let i = 0; i < key.length; i++) {
      h = Math.imul(31, h) + key.charCodeAt(i) | 0;
    }
    return h >>> 0;
  }

  addNode(node: T): void {
    for (let i = 0; i < this.virtualNodes; i++) {
      const hash = this.hash(`${node}:${i}`);
      this.ring.set(hash, node);
      this.sortedKeys.push(hash);
    }
    this.sortedKeys.sort((a, b) => a - b);
  }

  removeNode(node: T): void {
    for (let i = 0; i < this.virtualNodes; i++) {
      const hash = this.hash(`${node}:${i}`);
      this.ring.delete(hash);
      const idx = this.sortedKeys.indexOf(hash);
      if (idx !== -1) this.sortedKeys.splice(idx, 1);
    }
  }

  getNode(key: string): T {
    if (this.ring.size === 0) throw new Error('No nodes available');

    const hash = this.hash(key);

    // Binary search for first key >= hash
    let left = 0, right = this.sortedKeys.length - 1;
    while (left < right) {
      const mid = Math.floor((left + right) / 2);
      if (this.sortedKeys[mid] < hash) {
        left = mid + 1;
      } else {
        right = mid;
      }
    }

    // Wrap around if hash > all keys
    const idx = this.sortedKeys[left] >= hash ? left : 0;
    return this.ring.get(this.sortedKeys[idx])!;
  }

  // Get N nodes for replication
  getNodes(key: string, count: number): T[] {
    const nodes: T[] = [];
    const seen = new Set<T>();
    const hash = this.hash(key);

    let idx = this.sortedKeys.findIndex(k => k >= hash);
    if (idx === -1) idx = 0;

    while (nodes.length < count && seen.size < this.ring.size / this.virtualNodes) {
      const node = this.ring.get(this.sortedKeys[idx])!;
      if (!seen.has(node)) {
        nodes.push(node);
        seen.add(node);
      }
      idx = (idx + 1) % this.sortedKeys.length;
    }

    return nodes;
  }
}
```

## Caching Architecture

```typescript
// Multi-tier caching
class CacheManager {
  constructor(
    private l1Cache: LRUCache<string, unknown>,  // In-memory
    private l2Cache: Redis,                       // Distributed
    private l3Cache: CDN                          // Edge
  ) {}

  async get<T>(key: string, options?: CacheOptions): Promise<T | null> {
    // L1: In-memory (fastest)
    const l1Value = this.l1Cache.get(key);
    if (l1Value !== undefined) {
      return l1Value as T;
    }

    // L2: Redis (fast, shared)
    const l2Value = await this.l2Cache.get(key);
    if (l2Value !== null) {
      const parsed = JSON.parse(l2Value) as T;
      this.l1Cache.set(key, parsed);
      return parsed;
    }

    return null;
  }

  async set<T>(key: string, value: T, options: CacheOptions): Promise<void> {
    const serialized = JSON.stringify(value);

    // Write to all tiers
    await Promise.all([
      this.l1Cache.set(key, value),
      this.l2Cache.setex(key, options.ttl, serialized),
      options.cacheAtEdge ? this.l3Cache.purge(key) : Promise.resolve(),
    ]);
  }

  async invalidate(pattern: string): Promise<void> {
    // Invalidate across all tiers
    const keys = await this.l2Cache.keys(pattern);

    await Promise.all([
      ...keys.map(k => this.l1Cache.delete(k)),
      this.l2Cache.del(...keys),
      this.l3Cache.purgeByTag(pattern),
    ]);
  }
}

// Cache-aside pattern with thundering herd protection
class SafeCache {
  private locks: Map<string, Promise<unknown>> = new Map();

  async getOrSet<T>(
    key: string,
    fetcher: () => Promise<T>,
    ttl: number
  ): Promise<T> {
    // Try cache first
    const cached = await this.cache.get(key);
    if (cached !== null) {
      return cached as T;
    }

    // Check for existing lock (thundering herd protection)
    const existingLock = this.locks.get(key);
    if (existingLock) {
      return existingLock as Promise<T>;
    }

    // Acquire lock and fetch
    const fetchPromise = fetcher()
      .then(async (value) => {
        await this.cache.set(key, value, ttl);
        return value;
      })
      .finally(() => {
        this.locks.delete(key);
      });

    this.locks.set(key, fetchPromise);
    return fetchPromise;
  }
}

// Write-behind (write-back) caching
class WriteBehindCache {
  private writeBuffer: Map<string, { value: unknown; timestamp: number }> = new Map();
  private flushInterval: NodeJS.Timer;

  constructor(
    private cache: Cache,
    private db: Database,
    flushMs = 5000
  ) {
    this.flushInterval = setInterval(() => this.flush(), flushMs);
  }

  async write(key: string, value: unknown): Promise<void> {
    // Update cache immediately
    await this.cache.set(key, value);

    // Buffer for batch write to DB
    this.writeBuffer.set(key, { value, timestamp: Date.now() });
  }

  private async flush(): Promise<void> {
    if (this.writeBuffer.size === 0) return;

    const entries = Array.from(this.writeBuffer.entries());
    this.writeBuffer.clear();

    // Batch write to database
    await this.db.batchUpsert(
      entries.map(([key, { value }]) => ({ key, value }))
    );
  }
}
```

## Database Sharding

```typescript
// Horizontal sharding with routing
class ShardRouter {
  private shards: DatabaseShard[];
  private hashRing: ConsistentHash<DatabaseShard>;

  constructor(shards: DatabaseShard[]) {
    this.shards = shards;
    this.hashRing = new ConsistentHash(shards);
  }

  // Hash-based sharding
  getShardByHash(key: string): DatabaseShard {
    return this.hashRing.getNode(key);
  }

  // Range-based sharding
  getShardByRange(value: number): DatabaseShard {
    const ranges = [
      { max: 1000000, shard: 0 },
      { max: 5000000, shard: 1 },
      { max: 10000000, shard: 2 },
      { max: Infinity, shard: 3 },
    ];

    const range = ranges.find(r => value < r.max);
    return this.shards[range!.shard];
  }

  // Directory-based sharding
  async getShardByDirectory(entityId: string): Promise<DatabaseShard> {
    const mapping = await this.directory.get(entityId);
    return this.shards[mapping.shardId];
  }

  // Cross-shard query (scatter-gather)
  async queryAllShards<T>(query: Query): Promise<T[]> {
    const results = await Promise.all(
      this.shards.map(shard => shard.query<T>(query))
    );

    return this.mergeResults(results, query.orderBy);
  }
}

// Vitess-style sharding configuration
/*
{
  "keyspaces": {
    "users": {
      "sharded": true,
      "vindexes": {
        "user_hash": {
          "type": "hash"
        },
        "user_region": {
          "type": "lookup",
          "params": {
            "table": "user_region_idx",
            "from": "user_id",
            "to": "region"
          }
        }
      },
      "tables": {
        "users": {
          "column_vindexes": [
            {"column": "id", "name": "user_hash"}
          ]
        },
        "orders": {
          "column_vindexes": [
            {"column": "user_id", "name": "user_hash"}
          ]
        }
      }
    }
  }
}
*/
```

## Queue & Event Processing

```typescript
// Kafka consumer with exactly-once semantics
class ExactlyOnceConsumer {
  constructor(
    private kafka: Kafka,
    private db: Database,
    private topic: string
  ) {}

  async start(): Promise<void> {
    const consumer = this.kafka.consumer({
      groupId: 'order-processor',
      readUncommitted: false,
    });

    await consumer.subscribe({ topic: this.topic });

    await consumer.run({
      eachMessage: async ({ topic, partition, message }) => {
        const idempotencyKey = `${topic}:${partition}:${message.offset}`;

        await this.db.transaction(async (tx) => {
          // Check if already processed
          const existing = await tx.query(
            'SELECT 1 FROM processed_messages WHERE key = $1',
            [idempotencyKey]
          );

          if (existing.rows.length > 0) {
            return; // Already processed
          }

          // Process message
          await this.processMessage(tx, message);

          // Mark as processed
          await tx.query(
            'INSERT INTO processed_messages (key, processed_at) VALUES ($1, NOW())',
            [idempotencyKey]
          );
        });

        // Commit offset after successful processing
        await consumer.commitOffsets([{
          topic,
          partition,
          offset: (BigInt(message.offset) + 1n).toString(),
        }]);
      },
    });
  }

  private async processMessage(tx: Transaction, message: KafkaMessage): Promise<void> {
    const event = JSON.parse(message.value!.toString());
    // Process event within transaction
    await this.handleEvent(tx, event);
  }
}

// Dead letter queue with retry
class DeadLetterQueue {
  private readonly maxRetries = 5;
  private readonly backoffMs = [1000, 5000, 30000, 300000, 3600000];

  async handleFailure(event: Event, error: Error): Promise<void> {
    const retryCount = event.metadata.retryCount ?? 0;

    if (retryCount >= this.maxRetries) {
      await this.sendToDeadLetter(event, error);
      return;
    }

    const delay = this.backoffMs[retryCount];

    await this.queue.sendDelayed(
      'retry-queue',
      {
        ...event,
        metadata: {
          ...event.metadata,
          retryCount: retryCount + 1,
          lastError: error.message,
        },
      },
      delay
    );
  }

  async processDeadLetter(): Promise<void> {
    // Manual review process
    const events = await this.queue.consume('dead-letter');

    for (const event of events) {
      // Alert on-call
      await this.alertService.notify({
        severity: 'high',
        message: `Dead letter event: ${event.type}`,
        context: event,
      });

      // Store for analysis
      await this.analytics.store('dead_letters', event);
    }
  }
}
```

## Backpressure & Flow Control

```typescript
// Reactive streams with backpressure
class BackpressureStream<T> {
  private buffer: T[] = [];
  private readonly maxBuffer: number;
  private paused = false;

  constructor(
    private source: AsyncIterable<T>,
    private processor: (item: T) => Promise<void>,
    options: { maxBuffer: number; highWaterMark: number; lowWaterMark: number }
  ) {
    this.maxBuffer = options.maxBuffer;
  }

  async start(): Promise<void> {
    const iterator = this.source[Symbol.asyncIterator]();

    while (true) {
      // Apply backpressure
      while (this.buffer.length >= this.maxBuffer) {
        await this.waitForDrain();
      }

      const { value, done } = await iterator.next();
      if (done) break;

      this.buffer.push(value);
      this.processBuffer();
    }
  }

  private async processBuffer(): Promise<void> {
    while (this.buffer.length > 0 && !this.paused) {
      const item = this.buffer.shift()!;

      try {
        await this.processor(item);
      } catch (error) {
        // Handle error, potentially re-queue
        await this.handleError(item, error);
      }
    }
  }

  pause(): void {
    this.paused = true;
  }

  resume(): void {
    this.paused = false;
    this.processBuffer();
  }
}

// Token bucket rate limiter for outbound requests
class TokenBucket {
  private tokens: number;
  private lastRefill: number;

  constructor(
    private readonly capacity: number,
    private readonly refillRate: number // tokens per second
  ) {
    this.tokens = capacity;
    this.lastRefill = Date.now();
  }

  async acquire(tokens = 1): Promise<void> {
    this.refill();

    while (this.tokens < tokens) {
      const waitTime = (tokens - this.tokens) / this.refillRate * 1000;
      await new Promise(resolve => setTimeout(resolve, waitTime));
      this.refill();
    }

    this.tokens -= tokens;
  }

  private refill(): void {
    const now = Date.now();
    const elapsed = (now - this.lastRefill) / 1000;
    this.tokens = Math.min(this.capacity, this.tokens + elapsed * this.refillRate);
    this.lastRefill = now;
  }
}
```

---

*Learned: December 20, 2025*
*Tags: System Design, Scaling, Architecture, Performance, Distributed Systems*
