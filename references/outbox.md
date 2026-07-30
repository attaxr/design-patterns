# Outbox

Write events to the same database transaction as the state change, then have a background process publish them.

## Pressure

Your application writes to a database and then publishes an event (or sends a notification). If the publish fails after the DB write commits, the event is lost. If the publish succeeds before the DB write commits, readers see an event for state that does not exist yet. This is the dual-write problem.

## Solution

Write the event to the same database transaction as the state change. A background process reads pending events from the outbox table and publishes them. The event is never lost because it is committed in the same transaction as the data it describes.

```
BEGIN TX
  -> update state (INSERT/UPDATE orders)
  -> INSERT into outbox table (event_type, payload, status='pending')
COMMIT TX
  -> (background) SELECT * FROM outbox WHERE status='pending'
  -> publish to broker
  -> UPDATE outbox SET status='published'
```

## Implementation

```ts
// Before: dual-write bug -- race condition between DB commit and publish
async function createOrder(userId: string, items: CartItem[]) {
  const order = await db.orders.create({ userId, items });
  // If this throws or the process crashes here, the event is lost.
  await eventBus.publish("order.created", { orderId: order.id });
  return order;
}
```

```ts
// After: Transactional Outbox

interface OutboxEvent {
  id: string;
  aggregateType: string;
  aggregateId: string;
  eventType: string;
  payload: unknown;
  status: "pending" | "published" | "failed";
  createdAt: Date;
}

class OrderService {
  constructor(
    private readonly db: Database,
    private readonly outbox: OutboxRepository,
  ) {}

  async createOrder(userId: string, items: CartItem[]) {
    // Everything in one transaction
    return this.db.transaction(async (tx) => {
      const order = await tx.orders.create({ userId, items });

      await this.outbox.append(tx, {
        aggregateType: "order",
        aggregateId: order.id,
        eventType: "order.created",
        payload: { orderId: order.id, userId, items },
        status: "pending",
        createdAt: new Date(),
      });

      return order;
    });
  }
}

// Background publisher
class OutboxPublisher {
  private readonly BATCH_SIZE = 50;
  private readonly POLL_INTERVAL_MS = 1000;

  constructor(private readonly outboxRepo: OutboxRepository) {}

  start() {
    setInterval(() => this.publishBatch(), this.POLL_INTERVAL_MS);
  }

  private async publishBatch() {
    const events = await this.outboxRepo.fetchPending(this.BATCH_SIZE);
    for (const event of events) {
      try {
        await eventBus.publish(event.eventType, event.payload);
        await this.outboxRepo.markPublished(event.id);
      } catch (err) {
        console.error(`Failed to publish event ${event.id}:`, err);
      }
    }
  }
}

// Repository
interface OutboxRepository {
  append(tx: Transaction, event: Omit<OutboxEvent, "id">): Promise<void>;
  fetchPending(limit: number): Promise<OutboxEvent[]>;
  markPublished(id: string): Promise<void>;
}

class PostgresOutboxRepository implements OutboxRepository {
  async append(tx: Transaction, event: Omit<OutboxEvent, "id">) {
    await tx.sql`
      INSERT INTO outbox (aggregate_type, aggregate_id, event_type, payload, status, created_at)
      VALUES (${event.aggregateType}, ${event.aggregateId}, ${event.eventType},
              ${JSON.stringify(event.payload)}, 'pending', NOW())
    `;
  }

  async fetchPending(limit: number): Promise<OutboxEvent[]> {
    // Use SKIP LOCKED to avoid multiple workers picking the same events
    return this.pool.sql`
      SELECT * FROM outbox
      WHERE status = 'pending'
      ORDER BY created_at
      LIMIT ${limit}
      FOR UPDATE SKIP LOCKED
    `;
  }

  async markPublished(id: string): Promise<void> {
    await this.pool.sql`UPDATE outbox SET status = 'published' WHERE id = ${id}`;
  }
}
```

### Outbox table schema (SQL)

```sql
CREATE TABLE outbox (
    id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    aggregate_type TEXT NOT NULL,
    aggregate_id  TEXT NOT NULL,
    event_type    TEXT NOT NULL,
    payload       JSONB NOT NULL,
    status        TEXT NOT NULL DEFAULT 'pending',
    created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_outbox_status ON outbox (status, created_at);
```

## When you need this

Always when both these conditions are true:
- You write to a database AND publish an event / send a notification in the same operation
- Data consistency between the write and the publish matters

If you only publish events and never write to a database, you do not need an outbox. If you only write to a database and never publish events, you also do not need one.

## Variants

- **Transactional outbox with Kafka/RabbitMQ/SQS** -- same pattern, different destination
- **Outbox with Debezium (CDC)** -- instead of polling, use change data capture to stream outbox rows from the WAL
- **Outbox with scheduled cleanup** -- DELETE published events older than a retention period

## Common mistakes

1. **Outbox poller runs in-process without crash recovery** -- if the process dies, in-flight events are lost. Use a separate process or ensure the poller can resume after restart.
2. **No cleanup** -- the outbox table grows indefinitely. Add periodic cleanup of published events.
3. **Publishing order not guaranteed** -- if order matters, publish one at a time rather than in parallel batches.
4. **Duplicate publication** -- at-least-once delivery is the norm. Consumers must be idempotent.

## Minimal cut

Write the event to the same transaction as the state change. Have a cron job or background worker that polls for pending events and publishes them. Add the Outbox table, the publisher, and the cleanup job in one iteration.

## Related patterns

- **Inbox / Idempotency** -- The consumer side of the same problem. Outbox ensures at-least-once publish; Inbox ensures at-most-once processing.
- **Command** -- Outbox for events, Command for commands. Both need reliable delivery.
- **Event Sourcing** -- Outbox reliably publishes events from a normal DB. Event Sourcing uses events as the primary store.
