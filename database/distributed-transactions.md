# Distributed Transactions & Consistency

<div align="center">

![Database](https://img.shields.io/badge/Database-Distributed_Transactions-blue?style=for-the-badge)
![Pattern](https://img.shields.io/badge/Pattern-2PC_|_Saga_|_Outbox-orange?style=for-the-badge)

*Patterns for data consistency in distributed systems.*

![Distributed](https://upload.wikimedia.org/wikipedia/commons/8/87/Two-Phase_Commit_Protocol-Positive.svg)

</div>

## Two-Phase Commit (2PC)

```typescript
// Classic 2PC implementation
interface TransactionCoordinator {
  prepare(): Promise<boolean>;
  commit(): Promise<void>;
  rollback(): Promise<void>;
}

class TwoPhaseCommit {
  constructor(private participants: TransactionCoordinator[]) {}

  async execute<T>(operation: () => Promise<T>): Promise<T> {
    const preparedParticipants: TransactionCoordinator[] = [];

    try {
      // Phase 1: Prepare
      for (const participant of this.participants) {
        const canCommit = await participant.prepare();
        if (!canCommit) {
          throw new Error('Participant cannot commit');
        }
        preparedParticipants.push(participant);
      }

      // Execute the actual operation
      const result = await operation();

      // Phase 2: Commit
      await Promise.all(
        preparedParticipants.map(p => p.commit())
      );

      return result;
    } catch (error) {
      // Rollback all prepared participants
      await Promise.allSettled(
        preparedParticipants.map(p => p.rollback())
      );
      throw error;
    }
  }
}

// Participant implementation
class DatabaseParticipant implements TransactionCoordinator {
  private transaction: Transaction | null = null;

  async prepare(): Promise<boolean> {
    try {
      this.transaction = await this.db.beginTransaction();
      // Acquire locks, validate constraints
      await this.transaction.query('SELECT FOR UPDATE...');
      return true;
    } catch {
      return false;
    }
  }

  async commit(): Promise<void> {
    await this.transaction?.commit();
    this.transaction = null;
  }

  async rollback(): Promise<void> {
    await this.transaction?.rollback();
    this.transaction = null;
  }
}
```

## Saga Pattern with Compensation

```typescript
// Choreography-based saga
interface SagaStep<TContext> {
  name: string;
  execute(context: TContext): Promise<void>;
  compensate(context: TContext): Promise<void>;
}

class SagaOrchestrator<TContext> {
  constructor(private steps: SagaStep<TContext>[]) {}

  async execute(initialContext: TContext): Promise<void> {
    const completedSteps: SagaStep<TContext>[] = [];
    const context = { ...initialContext };

    for (const step of this.steps) {
      try {
        await this.executeWithRetry(step, context);
        completedSteps.push(step);
      } catch (error) {
        // Compensate in reverse order
        await this.compensate(completedSteps.reverse(), context);
        throw new SagaFailedError(step.name, error);
      }
    }
  }

  private async executeWithRetry(
    step: SagaStep<TContext>,
    context: TContext,
    maxRetries = 3
  ): Promise<void> {
    let lastError: Error;

    for (let attempt = 1; attempt <= maxRetries; attempt++) {
      try {
        await step.execute(context);
        return;
      } catch (error) {
        lastError = error;
        if (!this.isRetryable(error)) throw error;
        await this.delay(Math.pow(2, attempt) * 100);
      }
    }

    throw lastError!;
  }

  private async compensate(
    steps: SagaStep<TContext>[],
    context: TContext
  ): Promise<void> {
    for (const step of steps) {
      try {
        await step.compensate(context);
      } catch (error) {
        // Log and continue - compensation must be best-effort
        console.error(`Compensation failed for ${step.name}:`, error);
        await this.alertOps(step.name, error);
      }
    }
  }
}

// Order saga example
const orderSaga = new SagaOrchestrator<OrderContext>([
  {
    name: 'reserve-inventory',
    execute: async (ctx) => {
      ctx.reservationId = await inventoryService.reserve(ctx.items);
    },
    compensate: async (ctx) => {
      if (ctx.reservationId) {
        await inventoryService.cancelReservation(ctx.reservationId);
      }
    },
  },
  {
    name: 'process-payment',
    execute: async (ctx) => {
      ctx.paymentId = await paymentService.charge(ctx.customerId, ctx.amount);
    },
    compensate: async (ctx) => {
      if (ctx.paymentId) {
        await paymentService.refund(ctx.paymentId);
      }
    },
  },
  {
    name: 'create-shipment',
    execute: async (ctx) => {
      ctx.shipmentId = await shippingService.createShipment(ctx.orderId);
    },
    compensate: async (ctx) => {
      if (ctx.shipmentId) {
        await shippingService.cancelShipment(ctx.shipmentId);
      }
    },
  },
]);
```

## Transactional Outbox Pattern

```typescript
// Reliable event publishing with database transaction
class TransactionalOutbox {
  async executeWithEvents<T>(
    operation: (tx: Transaction) => Promise<{ result: T; events: DomainEvent[] }>
  ): Promise<T> {
    return this.db.transaction(async (tx) => {
      const { result, events } = await operation(tx);

      // Write events to outbox in same transaction
      for (const event of events) {
        await tx.insert('outbox', {
          id: event.id,
          aggregate_type: event.aggregateType,
          aggregate_id: event.aggregateId,
          event_type: event.type,
          payload: JSON.stringify(event.payload),
          created_at: new Date(),
          published: false,
        });
      }

      return result;
    });
  }
}

// Outbox poller (or use CDC like Debezium)
class OutboxPoller {
  private running = true;

  async start(): Promise<void> {
    while (this.running) {
      try {
        await this.processOutbox();
      } catch (error) {
        console.error('Outbox processing error:', error);
      }
      await this.delay(1000);
    }
  }

  private async processOutbox(): Promise<void> {
    const events = await this.db.query(`
      SELECT * FROM outbox
      WHERE published = false
      ORDER BY created_at
      LIMIT 100
      FOR UPDATE SKIP LOCKED
    `);

    for (const event of events) {
      try {
        await this.messageBroker.publish(event.event_type, event.payload);
        await this.db.update('outbox', { id: event.id }, { published: true });
      } catch (error) {
        // Will retry on next poll
        console.error(`Failed to publish event ${event.id}:`, error);
      }
    }
  }
}
```

## Read-Your-Writes Consistency

```typescript
// Session-based consistency token
class ConsistencyToken {
  constructor(
    public readonly timestamp: number,
    public readonly nodeId: string
  ) {}

  static fromHeader(header: string): ConsistencyToken | null {
    try {
      const [timestamp, nodeId] = header.split(':');
      return new ConsistencyToken(parseInt(timestamp), nodeId);
    } catch {
      return null;
    }
  }

  toHeader(): string {
    return `${this.timestamp}:${this.nodeId}`;
  }
}

// Middleware for read-your-writes
class ConsistencyMiddleware {
  constructor(
    private readonly replicaSelector: ReplicaSelector,
    private readonly maxWait = 5000
  ) {}

  async ensureConsistency(
    token: ConsistencyToken | null,
    operation: (replica: DatabaseReplica) => Promise<unknown>
  ): Promise<unknown> {
    if (!token) {
      // No consistency requirement, use any replica
      return operation(this.replicaSelector.selectRandom());
    }

    // Wait for replica to catch up
    const replica = await this.waitForReplica(token);
    return operation(replica);
  }

  private async waitForReplica(token: ConsistencyToken): Promise<DatabaseReplica> {
    const startTime = Date.now();

    while (Date.now() - startTime < this.maxWait) {
      const replica = this.replicaSelector.selectRandom();
      const replicaPosition = await replica.getReplicationPosition();

      if (replicaPosition >= token.timestamp) {
        return replica;
      }

      await this.delay(100);
    }

    // Fallback to primary if replica doesn't catch up
    return this.replicaSelector.selectPrimary();
  }
}

// Usage in API
app.post('/orders', async (req, res) => {
  const order = await orderService.create(req.body);

  // Include consistency token in response
  const token = new ConsistencyToken(Date.now(), primaryNodeId);
  res.setHeader('X-Consistency-Token', token.toHeader());

  return res.json(order);
});

app.get('/orders/:id', async (req, res) => {
  const token = ConsistencyToken.fromHeader(
    req.headers['x-consistency-token'] as string
  );

  const order = await consistencyMiddleware.ensureConsistency(
    token,
    (replica) => replica.query('SELECT * FROM orders WHERE id = $1', [req.params.id])
  );

  return res.json(order);
});
```

## Conflict Resolution (CRDTs)

```typescript
// Last-Writer-Wins Register
class LWWRegister<T> {
  constructor(
    public value: T,
    public timestamp: number,
    public nodeId: string
  ) {}

  merge(other: LWWRegister<T>): LWWRegister<T> {
    if (other.timestamp > this.timestamp) {
      return other;
    }
    if (other.timestamp === this.timestamp && other.nodeId > this.nodeId) {
      return other;
    }
    return this;
  }
}

// G-Counter (Grow-only counter)
class GCounter {
  private counts: Map<string, number> = new Map();

  constructor(private nodeId: string) {}

  increment(amount = 1): void {
    const current = this.counts.get(this.nodeId) ?? 0;
    this.counts.set(this.nodeId, current + amount);
  }

  value(): number {
    let total = 0;
    for (const count of this.counts.values()) {
      total += count;
    }
    return total;
  }

  merge(other: GCounter): GCounter {
    const merged = new GCounter(this.nodeId);
    const allNodes = new Set([...this.counts.keys(), ...other.counts.keys()]);

    for (const node of allNodes) {
      merged.counts.set(
        node,
        Math.max(this.counts.get(node) ?? 0, other.counts.get(node) ?? 0)
      );
    }

    return merged;
  }
}

// LWW-Element-Set (Last-Writer-Wins Set)
class LWWSet<T> {
  private addSet: Map<T, number> = new Map();
  private removeSet: Map<T, number> = new Map();

  add(element: T): void {
    this.addSet.set(element, Date.now());
  }

  remove(element: T): void {
    this.removeSet.set(element, Date.now());
  }

  has(element: T): boolean {
    const addTime = this.addSet.get(element);
    const removeTime = this.removeSet.get(element);

    if (!addTime) return false;
    if (!removeTime) return true;
    return addTime > removeTime;
  }

  merge(other: LWWSet<T>): LWWSet<T> {
    const merged = new LWWSet<T>();

    // Merge add sets (keep latest timestamp)
    for (const [element, time] of this.addSet) {
      merged.addSet.set(element, Math.max(time, other.addSet.get(element) ?? 0));
    }
    for (const [element, time] of other.addSet) {
      merged.addSet.set(element, Math.max(time, this.addSet.get(element) ?? 0));
    }

    // Merge remove sets
    for (const [element, time] of this.removeSet) {
      merged.removeSet.set(element, Math.max(time, other.removeSet.get(element) ?? 0));
    }
    for (const [element, time] of other.removeSet) {
      merged.removeSet.set(element, Math.max(time, this.removeSet.get(element) ?? 0));
    }

    return merged;
  }
}
```

---

*Learned: December 20, 2025*
*Tags: Distributed Systems, Transactions, Saga, CRDT, Consistency*
