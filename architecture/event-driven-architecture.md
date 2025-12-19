# Event-Driven Architecture Patterns

Building loosely coupled, scalable systems with events.

## Event Sourcing vs Event-Driven

```
┌─────────────────────────────────────────────────────────────┐
│ Event-Driven: Services communicate via events               │
│ Event Sourcing: State derived from event log               │
└─────────────────────────────────────────────────────────────┘
```

## CQRS + Event Sourcing

```typescript
// Command Side - Write Model
interface Command {
  type: string;
  payload: unknown;
  metadata: { userId: string; timestamp: Date; correlationId: string };
}

interface Event {
  id: string;
  aggregateId: string;
  type: string;
  payload: unknown;
  version: number;
  timestamp: Date;
}

class OrderAggregate {
  private events: Event[] = [];
  private state: OrderState = { status: 'pending', items: [] };

  apply(event: Event): void {
    switch (event.type) {
      case 'OrderCreated':
        this.state = { ...event.payload as OrderState };
        break;
      case 'ItemAdded':
        this.state.items.push(event.payload as OrderItem);
        break;
      case 'OrderConfirmed':
        this.state.status = 'confirmed';
        break;
    }
    this.events.push(event);
  }

  // Rehydrate from event store
  static fromEvents(events: Event[]): OrderAggregate {
    const aggregate = new OrderAggregate();
    events.forEach(e => aggregate.apply(e));
    return aggregate;
  }
}

// Event Store Implementation
class EventStore {
  async append(aggregateId: string, events: Event[], expectedVersion: number): Promise<void> {
    // Optimistic concurrency check
    const currentVersion = await this.getVersion(aggregateId);
    if (currentVersion !== expectedVersion) {
      throw new ConcurrencyError('Aggregate modified by another process');
    }

    await this.db.transaction(async (tx) => {
      for (const event of events) {
        await tx.insert('events', {
          ...event,
          aggregateId,
          version: ++currentVersion,
        });
      }
      // Publish to message broker
      await this.broker.publish('domain-events', events);
    });
  }

  async getEvents(aggregateId: string, fromVersion = 0): Promise<Event[]> {
    return this.db.query(
      'SELECT * FROM events WHERE aggregate_id = ? AND version > ? ORDER BY version',
      [aggregateId, fromVersion]
    );
  }
}
```

## Saga Pattern - Distributed Transactions

```typescript
// Orchestration-based Saga
class OrderSaga {
  private steps: SagaStep[] = [
    {
      execute: async (ctx) => {
        ctx.paymentId = await this.paymentService.reserve(ctx.orderId, ctx.amount);
      },
      compensate: async (ctx) => {
        await this.paymentService.releaseReservation(ctx.paymentId);
      },
    },
    {
      execute: async (ctx) => {
        ctx.inventoryId = await this.inventoryService.reserve(ctx.items);
      },
      compensate: async (ctx) => {
        await this.inventoryService.releaseReservation(ctx.inventoryId);
      },
    },
    {
      execute: async (ctx) => {
        await this.shippingService.scheduleDelivery(ctx.orderId);
      },
      compensate: async (ctx) => {
        await this.shippingService.cancelDelivery(ctx.orderId);
      },
    },
  ];

  async execute(context: SagaContext): Promise<void> {
    const completedSteps: SagaStep[] = [];

    try {
      for (const step of this.steps) {
        await step.execute(context);
        completedSteps.push(step);
      }
    } catch (error) {
      // Compensate in reverse order
      for (const step of completedSteps.reverse()) {
        try {
          await step.compensate(context);
        } catch (compensationError) {
          // Log and alert - manual intervention needed
          await this.alertService.criticalFailure({
            saga: 'OrderSaga',
            step: step.name,
            error: compensationError,
          });
        }
      }
      throw error;
    }
  }
}

// Choreography-based Saga (Event-driven)
class PaymentService {
  @EventHandler('OrderCreated')
  async onOrderCreated(event: OrderCreatedEvent): Promise<void> {
    try {
      await this.processPayment(event.orderId, event.amount);
      await this.eventBus.publish(new PaymentCompletedEvent(event.orderId));
    } catch (error) {
      await this.eventBus.publish(new PaymentFailedEvent(event.orderId, error));
    }
  }

  @EventHandler('OrderCancelled')
  async onOrderCancelled(event: OrderCancelledEvent): Promise<void> {
    await this.refundPayment(event.orderId);
  }
}
```

## Outbox Pattern - Reliable Event Publishing

```typescript
// Transactional Outbox
class OrderService {
  async createOrder(command: CreateOrderCommand): Promise<Order> {
    return this.db.transaction(async (tx) => {
      // 1. Create order
      const order = await tx.insert('orders', command);

      // 2. Write event to outbox (same transaction)
      await tx.insert('outbox', {
        id: uuid(),
        aggregateType: 'Order',
        aggregateId: order.id,
        eventType: 'OrderCreated',
        payload: JSON.stringify(order),
        createdAt: new Date(),
      });

      return order;
    });
  }
}

// Outbox Processor (CDC or Polling)
class OutboxProcessor {
  @Scheduled('*/5 * * * * *') // Every 5 seconds
  async processOutbox(): Promise<void> {
    const events = await this.db.query(
      'SELECT * FROM outbox WHERE processed_at IS NULL ORDER BY created_at LIMIT 100 FOR UPDATE SKIP LOCKED'
    );

    for (const event of events) {
      try {
        await this.messageBroker.publish(event.eventType, event.payload);
        await this.db.update('outbox', { id: event.id }, { processedAt: new Date() });
      } catch (error) {
        await this.db.update('outbox', { id: event.id }, {
          retryCount: event.retryCount + 1,
          lastError: error.message,
        });
      }
    }
  }
}

// Using Debezium for CDC (Change Data Capture)
// docker-compose.yml
/*
debezium:
  image: debezium/connect:2.4
  environment:
    - BOOTSTRAP_SERVERS=kafka:9092
    - GROUP_ID=outbox-connector
    - CONFIG_STORAGE_TOPIC=connect-configs
*/
```

## Dead Letter Queue & Retry Strategies

```typescript
class EventProcessor {
  private readonly maxRetries = 5;
  private readonly backoffMultiplier = 2;

  async processWithRetry<T>(
    event: Event,
    handler: (event: Event) => Promise<T>
  ): Promise<T> {
    let lastError: Error;

    for (let attempt = 1; attempt <= this.maxRetries; attempt++) {
      try {
        return await handler(event);
      } catch (error) {
        lastError = error;

        if (!this.isRetryable(error)) {
          await this.sendToDeadLetterQueue(event, error, 'non-retryable');
          throw error;
        }

        const delay = Math.pow(this.backoffMultiplier, attempt) * 1000;
        await this.sleep(delay);
      }
    }

    // Max retries exceeded
    await this.sendToDeadLetterQueue(event, lastError, 'max-retries-exceeded');
    throw lastError;
  }

  private isRetryable(error: Error): boolean {
    return error instanceof NetworkError ||
           error instanceof TimeoutError ||
           error instanceof ServiceUnavailableError;
  }

  private async sendToDeadLetterQueue(
    event: Event,
    error: Error,
    reason: string
  ): Promise<void> {
    await this.dlq.publish({
      originalEvent: event,
      error: { message: error.message, stack: error.stack },
      reason,
      timestamp: new Date(),
    });
  }
}
```

---

*Learned: December 20, 2025*
*Tags: Architecture, Event Sourcing, CQRS, Saga, Microservices*
