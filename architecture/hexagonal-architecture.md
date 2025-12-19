# Hexagonal Architecture (Ports & Adapters)

Framework-agnostic business logic with replaceable infrastructure.

## Core Structure

```
┌─────────────────────────────────────────────────────────────────┐
│                        Driving Side                              │
│   (Primary Adapters - UI, CLI, Tests, API Controllers)          │
│         ▼           ▼           ▼           ▼                   │
│    ┌─────────────────────────────────────────────────┐          │
│    │              PORTS (Interfaces)                 │          │
│    │   ┌─────────────────────────────────────────┐   │          │
│    │   │                                         │   │          │
│    │   │         APPLICATION CORE                │   │          │
│    │   │    ┌─────────────────────────────┐     │   │          │
│    │   │    │      DOMAIN MODEL           │     │   │          │
│    │   │    │   (Entities, Value Objects, │     │   │          │
│    │   │    │    Domain Services)         │     │   │          │
│    │   │    └─────────────────────────────┘     │   │          │
│    │   │                                         │   │          │
│    │   └─────────────────────────────────────────┘   │          │
│    └─────────────────────────────────────────────────┘          │
│         ▼           ▼           ▼           ▼                   │
│   (Secondary Adapters - DB, Cache, External APIs, Messaging)    │
│                        Driven Side                               │
└─────────────────────────────────────────────────────────────────┘
```

## Project Structure

```
src/
├── domain/                    # Pure business logic
│   ├── entities/
│   │   ├── Order.ts
│   │   └── Customer.ts
│   ├── value-objects/
│   │   ├── Money.ts
│   │   └── OrderId.ts
│   ├── events/
│   │   └── OrderEvents.ts
│   └── services/
│       └── PricingService.ts
│
├── application/               # Use cases & orchestration
│   ├── ports/
│   │   ├── driving/          # Primary ports (inbound)
│   │   │   ├── CreateOrderUseCase.ts
│   │   │   └── GetOrderQuery.ts
│   │   └── driven/           # Secondary ports (outbound)
│   │       ├── OrderRepository.ts
│   │       ├── PaymentGateway.ts
│   │       └── EventPublisher.ts
│   ├── services/
│   │   └── OrderApplicationService.ts
│   └── dto/
│       └── OrderDTO.ts
│
└── infrastructure/            # External world adapters
    ├── driving/              # Primary adapters
    │   ├── rest/
    │   │   └── OrderController.ts
    │   ├── graphql/
    │   │   └── OrderResolver.ts
    │   └── cli/
    │       └── OrderCommands.ts
    └── driven/               # Secondary adapters
        ├── persistence/
        │   ├── PostgresOrderRepository.ts
        │   └── RedisOrderCache.ts
        ├── payment/
        │   ├── StripePaymentGateway.ts
        │   └── PayPalPaymentGateway.ts
        └── messaging/
            └── KafkaEventPublisher.ts
```

## Ports - The Contracts

```typescript
// Driving Port (Primary) - What the application offers
interface CreateOrderUseCase {
  execute(command: CreateOrderCommand): Promise<Result<OrderId, OrderError>>;
}

interface GetOrderQuery {
  execute(orderId: string): Promise<Result<OrderDTO, NotFoundError>>;
}

// Driven Port (Secondary) - What the application needs
interface OrderRepository {
  save(order: Order): Promise<void>;
  findById(id: OrderId): Promise<Order | null>;
  findByCustomer(customerId: CustomerId, pagination: Pagination): Promise<Page<Order>>;
  nextId(): OrderId;
}

interface PaymentGateway {
  charge(payment: PaymentRequest): Promise<Result<PaymentConfirmation, PaymentError>>;
  refund(paymentId: PaymentId, amount: Money): Promise<Result<RefundConfirmation, RefundError>>;
}

interface EventPublisher {
  publish<T extends DomainEvent>(event: T): Promise<void>;
  publishAll(events: DomainEvent[]): Promise<void>;
}
```

## Application Service - Orchestration

```typescript
// Application layer - orchestrates domain and infrastructure
class OrderApplicationService implements CreateOrderUseCase, GetOrderQuery {
  constructor(
    private readonly orderRepository: OrderRepository,
    private readonly customerRepository: CustomerRepository,
    private readonly paymentGateway: PaymentGateway,
    private readonly eventPublisher: EventPublisher,
    private readonly pricingService: PricingService
  ) {}

  async execute(command: CreateOrderCommand): Promise<Result<OrderId, OrderError>> {
    // 1. Validate customer exists
    const customer = await this.customerRepository.findById(
      CustomerId.from(command.customerId)
    );
    if (!customer) {
      return err(new OrderError('Customer not found'));
    }

    // 2. Create order (domain logic)
    const orderId = this.orderRepository.nextId();
    const order = Order.create(orderId, customer.id);

    for (const item of command.items) {
      order.addItem(
        ProductId.from(item.productId),
        Quantity.of(item.quantity),
        Money.of(item.price, command.currency)
      );
    }

    // 3. Calculate pricing (domain service)
    const pricing = await this.pricingService.calculateOrderTotal(order, customer);
    order.setPricing(pricing);

    // 4. Process payment (infrastructure)
    const paymentResult = await this.paymentGateway.charge({
      orderId: order.id,
      amount: pricing.total,
      customerId: customer.id,
      paymentMethod: command.paymentMethod,
    });

    if (paymentResult.isErr()) {
      return err(new OrderError('Payment failed', paymentResult.error));
    }

    // 5. Confirm order
    order.confirm(paymentResult.value.paymentId);

    // 6. Persist
    await this.orderRepository.save(order);

    // 7. Publish domain events
    await this.eventPublisher.publishAll(order.domainEvents);

    return ok(order.id);
  }
}
```

## Adapters - Pluggable Infrastructure

```typescript
// Primary Adapter - REST Controller
@Controller('/orders')
class OrderController {
  constructor(
    private readonly createOrder: CreateOrderUseCase,
    private readonly getOrder: GetOrderQuery
  ) {}

  @Post('/')
  async create(@Body() body: CreateOrderRequest): Promise<ApiResponse<OrderResponse>> {
    const result = await this.createOrder.execute({
      customerId: body.customerId,
      items: body.items,
      currency: body.currency,
      paymentMethod: body.paymentMethod,
    });

    if (result.isErr()) {
      throw new HttpException(result.error.message, HttpStatus.BAD_REQUEST);
    }

    return { id: result.value.toString() };
  }
}

// Secondary Adapter - PostgreSQL Repository
class PostgresOrderRepository implements OrderRepository {
  constructor(private readonly prisma: PrismaClient) {}

  async save(order: Order): Promise<void> {
    await this.prisma.order.upsert({
      where: { id: order.id.value },
      create: this.toPersistence(order),
      update: this.toPersistence(order),
    });
  }

  async findById(id: OrderId): Promise<Order | null> {
    const data = await this.prisma.order.findUnique({
      where: { id: id.value },
      include: { items: true },
    });
    return data ? this.toDomain(data) : null;
  }

  private toDomain(data: OrderData): Order {
    return Order.reconstitute({
      id: OrderId.from(data.id),
      customerId: CustomerId.from(data.customerId),
      status: data.status as OrderStatus,
      items: data.items.map(this.itemToDomain),
      createdAt: data.createdAt,
    });
  }

  private toPersistence(order: Order): OrderData {
    return {
      id: order.id.value,
      customerId: order.customerId.value,
      status: order.status,
      total: order.total.amount,
      currency: order.total.currency,
      createdAt: order.createdAt,
    };
  }
}

// Secondary Adapter - Stripe Payment Gateway
class StripePaymentGateway implements PaymentGateway {
  constructor(private readonly stripe: Stripe) {}

  async charge(request: PaymentRequest): Promise<Result<PaymentConfirmation, PaymentError>> {
    try {
      const paymentIntent = await this.stripe.paymentIntents.create({
        amount: request.amount.toCents(),
        currency: request.amount.currency.toLowerCase(),
        customer: request.customerId.value,
        payment_method: request.paymentMethod,
        confirm: true,
      });

      return ok({
        paymentId: PaymentId.from(paymentIntent.id),
        status: 'confirmed',
        confirmedAt: new Date(),
      });
    } catch (error) {
      return err(new PaymentError(error.message, error.code));
    }
  }
}
```

## Dependency Injection Composition Root

```typescript
// Wire everything together at the composition root
function createContainer(): Container {
  const container = new Container();

  // Infrastructure
  container.bind(PrismaClient).toSelf().inSingletonScope();
  container.bind(Stripe).toConstantValue(new Stripe(config.stripeKey));
  container.bind(Kafka).toConstantValue(new Kafka(config.kafka));

  // Repositories (Driven Ports → Adapters)
  container.bind<OrderRepository>('OrderRepository')
    .to(PostgresOrderRepository);
  container.bind<CustomerRepository>('CustomerRepository')
    .to(PostgresCustomerRepository);

  // External Services (Driven Ports → Adapters)
  container.bind<PaymentGateway>('PaymentGateway')
    .to(StripePaymentGateway);
  container.bind<EventPublisher>('EventPublisher')
    .to(KafkaEventPublisher);

  // Domain Services
  container.bind(PricingService).toSelf();

  // Use Cases (Application Services)
  container.bind<CreateOrderUseCase>('CreateOrderUseCase')
    .to(OrderApplicationService);

  // Controllers (Driving Adapters)
  container.bind(OrderController).toSelf();

  return container;
}

// For testing - swap implementations
function createTestContainer(): Container {
  const container = createContainer();

  // Replace with test doubles
  container.rebind<OrderRepository>('OrderRepository')
    .to(InMemoryOrderRepository);
  container.rebind<PaymentGateway>('PaymentGateway')
    .to(FakePaymentGateway);
  container.rebind<EventPublisher>('EventPublisher')
    .to(SpyEventPublisher);

  return container;
}
```

---

*Learned: 2024*
*Tags: Architecture, Hexagonal, Ports and Adapters, Clean Architecture*
