# Domain-Driven Design in Practice

Strategic and tactical patterns for complex domains.

## Bounded Contexts & Context Mapping

```
┌─────────────────────────────────────────────────────────────────────┐
│                        E-Commerce Platform                          │
├─────────────────┬─────────────────┬─────────────────────────────────┤
│   Order Context │ Inventory Context│     Shipping Context           │
│   ┌───────────┐ │  ┌────────────┐  │  ┌─────────────┐              │
│   │ Order     │ │  │ Stock      │  │  │ Shipment    │              │
│   │ LineItem  │ │  │ Warehouse  │  │  │ Carrier     │              │
│   │ Customer* │ │  │ Product*   │  │  │ Address*    │              │
│   └───────────┘ │  └────────────┘  │  └─────────────┘              │
│   * = Reference │  * = Own def     │  * = Value Object             │
└─────────────────┴─────────────────┴─────────────────────────────────┘

Context Map Relationships:
- Order → Inventory: Customer/Supplier
- Order → Shipping: Partnership
- Inventory → Shipping: Conformist
```

## Aggregate Design Rules

```typescript
// Rule 1: Reference aggregates by identity only
// Rule 2: Keep aggregates small
// Rule 3: Modify one aggregate per transaction
// Rule 4: Use eventual consistency between aggregates

// ❌ Wrong: Large aggregate, violates boundaries
class Order {
  customer: Customer; // Full object reference
  items: OrderItem[];
  shipping: ShippingDetails;
  payment: PaymentDetails;
  inventory: InventoryReservation[];
}

// ✅ Correct: Small aggregate, references by ID
class Order {
  readonly id: OrderId;
  private customerId: CustomerId;
  private items: OrderItem[];
  private status: OrderStatus;

  constructor(id: OrderId, customerId: CustomerId) {
    this.id = id;
    this.customerId = customerId;
    this.items = [];
    this.status = OrderStatus.Draft;
  }

  addItem(productId: ProductId, quantity: Quantity, price: Money): void {
    this.ensureModifiable();
    const item = new OrderItem(productId, quantity, price);
    this.items.push(item);
    this.addDomainEvent(new ItemAddedToOrder(this.id, item));
  }

  submit(): void {
    this.ensureModifiable();
    if (this.items.length === 0) {
      throw new DomainError('Cannot submit empty order');
    }
    this.status = OrderStatus.Submitted;
    this.addDomainEvent(new OrderSubmitted(this.id, this.customerId, this.total));
  }

  private ensureModifiable(): void {
    if (this.status !== OrderStatus.Draft) {
      throw new DomainError('Order cannot be modified');
    }
  }
}
```

## Value Objects - Immutable Domain Concepts

```typescript
// Money: Classic value object
class Money {
  private constructor(
    readonly amount: number,
    readonly currency: Currency
  ) {
    if (amount < 0) throw new DomainError('Amount cannot be negative');
  }

  static of(amount: number, currency: Currency): Money {
    return new Money(amount, currency);
  }

  add(other: Money): Money {
    this.ensureSameCurrency(other);
    return Money.of(this.amount + other.amount, this.currency);
  }

  multiply(factor: number): Money {
    return Money.of(this.amount * factor, this.currency);
  }

  equals(other: Money): boolean {
    return this.amount === other.amount && this.currency === other.currency;
  }

  private ensureSameCurrency(other: Money): void {
    if (this.currency !== other.currency) {
      throw new DomainError('Currency mismatch');
    }
  }
}

// Address: Complex value object
class Address {
  private constructor(
    readonly street: string,
    readonly city: string,
    readonly state: string,
    readonly postalCode: PostalCode,
    readonly country: CountryCode
  ) {}

  static create(props: AddressProps): Result<Address, ValidationError> {
    const errors: string[] = [];

    if (!props.street?.trim()) errors.push('Street required');
    if (!props.city?.trim()) errors.push('City required');

    const postalCode = PostalCode.create(props.postalCode, props.country);
    if (postalCode.isErr()) errors.push(postalCode.error.message);

    if (errors.length > 0) {
      return err(new ValidationError(errors));
    }

    return ok(new Address(
      props.street.trim(),
      props.city.trim(),
      props.state?.trim() ?? '',
      postalCode.value,
      props.country
    ));
  }

  format(): string {
    return `${this.street}, ${this.city}, ${this.state} ${this.postalCode}`;
  }
}
```

## Domain Services

```typescript
// When logic doesn't naturally fit in an entity
class PricingService {
  constructor(
    private readonly discountRepository: DiscountRepository,
    private readonly taxService: TaxService
  ) {}

  async calculateOrderTotal(order: Order, customer: Customer): Promise<OrderPricing> {
    const subtotal = order.calculateSubtotal();

    // Get applicable discounts
    const discounts = await this.discountRepository.findApplicable({
      customerId: customer.id,
      customerTier: customer.tier,
      orderValue: subtotal,
      productIds: order.productIds,
    });

    const discountAmount = this.applyDiscounts(subtotal, discounts);
    const taxableAmount = subtotal.subtract(discountAmount);
    const tax = await this.taxService.calculate(taxableAmount, order.shippingAddress);

    return new OrderPricing({
      subtotal,
      discounts,
      discountAmount,
      tax,
      total: taxableAmount.add(tax),
    });
  }

  private applyDiscounts(amount: Money, discounts: Discount[]): Money {
    return discounts.reduce(
      (acc, discount) => acc.add(discount.calculate(amount)),
      Money.zero(amount.currency)
    );
  }
}
```

## Specification Pattern

```typescript
// Encapsulate business rules as first-class objects
interface Specification<T> {
  isSatisfiedBy(candidate: T): boolean;
  and(other: Specification<T>): Specification<T>;
  or(other: Specification<T>): Specification<T>;
  not(): Specification<T>;
  toQuery(): QueryFilter; // For repository queries
}

abstract class CompositeSpecification<T> implements Specification<T> {
  abstract isSatisfiedBy(candidate: T): boolean;
  abstract toQuery(): QueryFilter;

  and(other: Specification<T>): Specification<T> {
    return new AndSpecification(this, other);
  }

  or(other: Specification<T>): Specification<T> {
    return new OrSpecification(this, other);
  }

  not(): Specification<T> {
    return new NotSpecification(this);
  }
}

// Usage
class EligibleForPremiumSpec extends CompositeSpecification<Customer> {
  isSatisfiedBy(customer: Customer): boolean {
    return customer.totalPurchases.isGreaterThan(Money.of(10000, 'USD')) &&
           customer.accountAge.isGreaterThan(Duration.months(6)) &&
           customer.hasNoDisputes();
  }

  toQuery(): QueryFilter {
    return {
      totalPurchases: { $gte: 10000 },
      createdAt: { $lte: subMonths(new Date(), 6) },
      disputeCount: 0,
    };
  }
}

const premiumSpec = new EligibleForPremiumSpec();
const activeSpec = new ActiveCustomerSpec();
const eligibleCustomers = await customerRepo.findSatisfying(
  premiumSpec.and(activeSpec)
);
```

## Anti-Corruption Layer

```typescript
// Protect your domain from external systems
class LegacyOrderAdapter implements OrderPort {
  constructor(private readonly legacyClient: LegacySystemClient) {}

  async getOrder(orderId: OrderId): Promise<Order> {
    const legacyOrder = await this.legacyClient.fetchOrder(orderId.value);
    return this.translateToDomain(legacyOrder);
  }

  private translateToDomain(legacy: LegacyOrderDTO): Order {
    // Map legacy status codes to domain status
    const status = this.mapStatus(legacy.STATUS_CD);

    // Convert legacy money format (cents as string) to Money
    const total = Money.of(
      parseInt(legacy.TOTAL_AMT) / 100,
      this.mapCurrency(legacy.CURRENCY_CD)
    );

    // Handle legacy's nullable customer
    const customerId = legacy.CUST_ID
      ? CustomerId.from(legacy.CUST_ID)
      : CustomerId.guest();

    return Order.reconstitute({
      id: OrderId.from(legacy.ORDER_ID),
      customerId,
      status,
      total,
      createdAt: this.parseDate(legacy.CREATE_DT),
    });
  }

  private mapStatus(legacyCode: string): OrderStatus {
    const mapping: Record<string, OrderStatus> = {
      'A': OrderStatus.Active,
      'P': OrderStatus.Pending,
      'C': OrderStatus.Completed,
      'X': OrderStatus.Cancelled,
    };
    return mapping[legacyCode] ?? OrderStatus.Unknown;
  }
}
```

---

*Learned: December 20, 2025*
*Tags: DDD, Architecture, Domain Modeling, Aggregates, Bounded Context*
