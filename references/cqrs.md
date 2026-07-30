# CQRS -- Command Query Responsibility Segregation

Separate the model that reads data from the model that writes data.

## Pressure

Reads and writes have different shapes or wildly different volumes. A single model is optimized for neither -- writes need normalized data for consistency, reads need denormalized data for speed. One change to the write model risks breaking the read paths. When reads vastly outnumber writes, scaling them together wastes resources.

## Solution

Split the model into two: Commands (writes) and Queries (reads). They can use different storage, different schemas, and different code paths. A synchronization mechanism (projection) keeps the read side up to date.

```
Write Side (Commands)      Read Side (Queries)
+------------------+       +------------------+
|  PlaceOrder      |       |  ActiveOrderDTO  |
|  CancelOrder     |       |  OrderHistoryDTO |
|  UpdateShipping  |       |  CustomerSummary |
+--------+---------+       +--------^---------+
         |                          |
         v                          |
  +----------+        +-----------------------+
  |  Write DB |------->|  Sync / Projection   |
  | (Normalized)     | |  (async update)      |
  +----------+        +-----------------------+
```

## Implementation

### Light version (same store, separate services)

```ts
// Write model -- optimized for writes
class OrderCommandService {
  constructor(private readonly db: Database) {}

  async placeOrder(userId: string, items: ItemDto[]) {
    await this.db.transaction(async (tx) => {
      const orderId = await tx
        .insertInto("orders")
        .values({ user_id: userId })
        .returning("id");
      for (const item of items) {
        await tx
          .insertInto("order_items")
          .values({ order_id: orderId, ...item });
      }
    });
  }
}

// Read model -- query-specific DTO, can be denormalized
class OrderQueryService {
  constructor(private readonly db: Database) {}

  async getActiveOrders(userId: string): Promise<ActiveOrderDTO[]> {
    return this.db.sql`
      SELECT o.id, o.created_at, o.status,
             json_agg(json_build_object('name', p.name, 'qty', oi.qty)) as items
      FROM orders o
      JOIN order_items oi ON oi.order_id = o.id
      JOIN products p ON p.id = oi.product_id
      WHERE o.user_id = ${userId} AND o.status = 'active'
      GROUP BY o.id
    `;
  }
}
```

### Full CQRS (separate stores)

```ts
// Command writes to write-optimized store
class PlaceOrderCommand {
  constructor(private readonly writeDb: Database) {}
  async execute(cmd: { userId: string; items: ItemDto[] }) {
    // Write to normalized tables
  }
}

// A projection syncs data to the read store
class OrderProjection {
  constructor(
    private readonly writeDb: Database,
    private readonly readDb: Database,
  ) {}

  async onOrderPlaced(event: OrderPlacedEvent) {
    const denormalized = await this.buildReadModel(event);
    await this.readDb.upsert("orders", denormalized);
  }
}

// Query reads from read-optimized store
class OrderQueryService {
  constructor(private readonly readDb: Database) {}
  async getActiveOrders(userId: string) {
    return this.readDb.find("orders", { userId, status: "active" });
  }
}
```

## The tell

- A single model has methods for both `placeOrder()` and `getActiveOrders()` that use completely different query patterns
- You are doing complex joins on every read to denormalize data that could be stored in query shape
- Read throughput is limited by write-model table structure

## Cost

- **Two models to keep in sync** -- the projection can fall behind, meaning users see stale data
- **Eventual consistency is a product decision** -- decide what "I saved it but do not see it" should look like in the UI before building this
- **More code** -- the sync between write and read stores adds infrastructure

## Lighter first steps (before full CQRS)

1. **Read replicas** -- same model, different database instance for reads
2. **Materialized views** -- denormalized read shapes built in the database
3. **Separate query methods** -- different methods on the same repository
4. **Separate query services** -- different classes, same database

The term "CQRS" is often used for all of these. Be explicit about which level you mean.

## Related patterns

- **Event Sourcing** -- CQRS and Event Sourcing are often paired. Events become the write model; projections build read models.
- **Repository** -- Repositories can be split into command repositories and query repositories.
