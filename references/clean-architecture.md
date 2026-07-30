# Clean Architecture

Organize code into concentric layers with dependencies pointing inward toward the domain.

## Pressure

Business rules import the database driver, HTTP framework, or vendor SDK, making it impossible to test or reuse the domain without running the infrastructure. The ORM or HTTP framework shapes how business logic is expressed. Infrastructure changes require rewrites of domain code.

## Solution

Organize the application into layers. The innermost layer (Domain) has zero dependencies on infrastructure. Each outer layer depends on the layer inside it. Infrastructure implements interfaces defined by the domain.

```
+-----------------------------------------------------+
|          Frameworks & Drivers                        |
|  (HTTP, Database, Queue, Vendors)                    |
|  +-----------------------------------------------+   |
|  |      Interface Adapters                       |   |
|  |  (Controllers, Presenters, Gateways/Repos)    |   |
|  |  +-----------------------------------------+  |   |
|  |  |   Application Layer                     |  |   |
|  |  |  (Use Cases / Services)                 |  |   |
|  |  |  +-----------------------------------+  |  |   |
|  |  |  |   Domain Layer                    |  |  |   |
|  |  |  |  (Entities, VOs, Domain Services) |  |  |   |
|  |  |  +-----------------------------------+  |  |   |
|  |  +-----------------------------------------+  |   |
|  +-----------------------------------------------+   |
+-----------------------------------------------------+
```

## Implementation

```ts
// Layer 1 -- Domain (innermost). Zero dependencies on infrastructure.
interface OrderRepository {
  save(order: Order): Promise<void>;
  findById(id: string): Promise<Order | null>;
}

class Order {
  constructor(
    public readonly id: string,
    public readonly userId: string,
    private _status: OrderStatus,
    private _items: OrderItem[],
  ) {}

  get status() { return this._status; }
  get total() { return this._items.reduce((sum, i) => sum + i.price * i.qty, 0); }

  fulfill() {
    if (this._status !== "pending")
      throw new Error("only pending orders can be fulfilled");
    this._status = "fulfilled";
  }
  // Domain rules live here -- no import of database, HTTP, or framework types.
}

// Layer 2 -- Application (Use Cases)
class FulfillOrderUseCase {
  constructor(
    private readonly orders: OrderRepository,
    private readonly email: EmailService,
  ) {}

  async execute(orderId: string) {
    const order = await this.orders.findById(orderId);
    if (!order) throw new Error("order not found");
    order.fulfill();
    await this.orders.save(order);
    await this.email.sendConfirmation(order.userId, order.id);
  }
}

// Layer 3 -- Infrastructure (outermost)
class PostgresOrderRepository implements OrderRepository {
  constructor(private readonly db: Database) {}

  async save(order: Order) {
    await this.db.sql`UPDATE orders SET status = ${order.status} WHERE id = ${order.id}`;
  }

  async findById(id: string): Promise<Order | null> {
    const rows = await this.db.sql`SELECT * FROM orders WHERE id = ${id}`;
    if (!rows.length) return null;
    return new Order(rows[0].id, rows[0].user_id, rows[0].status, rows[0].items);
  }
}
```

### The dependency rule

Dependencies point inward, toward the domain. The domain defines interfaces (contracts); infrastructure implements them. The outer layers know about the inner layers, but never the reverse.

- Domain: zero imports of infrastructure code (no database, no HTTP, no framework)
- Application: imports only Domain
- Interface Adapters: imports Application and Domain
- Frameworks/Drivers: implements adapters

### The practical test

You should be able to run the entire application with in-memory implementations and no network. If you cannot, the dependency rule has been violated somewhere -- infrastructure types are leaking into the domain.

## Cost

More indirection and more files. Every feature crosses several layers: Controller to Use Case to Domain to Repository interface to Repository implementation to Database. This is real friction on small codebases. The payoff arrives when infrastructure changes or when the domain needs to be tested without any of it running.

## Minimal cut

You do not need all four layers from day one. Start with:
1. **Domain** -- entities and value objects, pure of infrastructure imports
2. **Repository interfaces** in the domain
3. **Repository implementations** in an `infra/` directory

Add the Use Case layer when application logic (orchestration, transaction management) grows too complex to live in controllers.

## Clean vs Hexagonal

Clean Architecture and Hexagonal Architecture (Ports and Adapters) are the same idea with different diagrams. Both invert the dependency from domain-to-infrastructure to infrastructure-to-domain. Both isolate the domain from frameworks and databases. Pick one vocabulary and be consistent. The differences are cosmetic.

## Related patterns

- **Hexagonal Architecture** -- Same intent, different diagram shape
- **Dependency Injection** -- DI is the mechanism that makes Clean Architecture possible
- **Repository** -- Repository is the persistence port
- **CQRS** -- CQRS is often implemented within a Clean Architecture layer structure
