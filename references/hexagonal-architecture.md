# Hexagonal Architecture (Ports and Adapters)

Isolate the domain core from external concerns by defining ports (interfaces) that adapters implement.

## Pressure

The domain must be callable from multiple entry points (HTTP, CLI, queue consumer, test harness) without duplicating logic. Infrastructure must be swappable without touching domain code. The domain currently imports the database driver, rendering it untestable without a real database.

## Solution

The domain defines ports (interfaces) that represent what the application needs from the outside world. Adapters connect those ports to concrete infrastructure. The domain never imports an adapter -- it only depends on its own ports.

```
HTTP Adapter  -->  |                |  <-- Postgres Adapter
CLI Adapter   -->  |   Domain Core  |  <-- S3 Adapter
Queue Adapter -->  |   (Ports)      |  <-- Redis Adapter
Test Harness  -->  |                |  <-- InMemory Adapter
```

## Implementation

```ts
// Port -- defined in the domain. The domain defines what it needs.
interface InventoryRepository {
  findByHost(host: string): Promise<Inventory | null>;
  save(inventory: Inventory): Promise<void>;
}

// Adapter -- infrastructure implements the port
class PostgresInventoryRepository implements InventoryRepository {
  constructor(private pool: Pool) {}

  async findByHost(host: string): Promise<Inventory | null> {
    const result = await this.pool.query(
      "SELECT * FROM inventory WHERE host = $1", [host],
    );
    if (result.rows.length === 0) return null;
    return this.rowToInventory(result.rows[0]);
  }

  async save(inventory: Inventory): Promise<void> { /* upsert logic */ }

  private rowToInventory(row: any): Inventory {
    return new Inventory(row.host, row.data, new Date(row.scanned_at));
  }
}

// Another adapter -- same port, different implementation
class InMemoryInventoryRepository implements InventoryRepository {
  private items = new Map<string, Inventory>();

  async findByHost(host: string) { return this.items.get(host) ?? null; }
  async save(inventory: Inventory) { this.items.set(inventory.host, inventory); }
}
```

### Multiple entry points

The same use case is callable from any entry point because the use case depends on ports, not on transport.

```ts
// HTTP entry point
router.post("/scan", async (req, res) => {
  const useCase = new RunScanUseCase(inventoryRepo);  // port
  await useCase.execute(req.body.target);
  res.json({ ok: true });
});

// CLI entry point
async function main() {
  const [, , target] = process.argv;
  const useCase = new RunScanUseCase(inventoryRepo);  // same port
  await useCase.execute(target);
}

// Queue entry point
consumer.on("message", async (msg) => {
  const useCase = new RunScanUseCase(inventoryRepo);  // same port
  await useCase.execute(msg.body.target);
});
```

### The two sides of the hexagon

| Side | Ports (domain defines) | Adapters (infrastructure provides) |
|------|----------------------|-----------------------------------|
| **Driving (left)** | ApplicationService interface | HTTP controller, CLI handler, queue consumer |
| **Driven (right)** | Repository interface, EmailService interface | Postgres repo, SendGrid adapter, Redis cache |

Driving adapters call the domain. Driven adapters are called by the domain.

## The practical test

Can you run the entire domain with in-memory adapters and no network? If all adapters are swappable, the answer is yes.

## Hexagonal vs Clean

They are the same idea with different diagrams. Both invert the dependency from domain-to-infrastructure to infrastructure-to-domain. The differences are cosmetic. Pick one vocabulary and be consistent.

| Aspect | Clean Architecture | Hexagonal |
|--------|-------------------|-----------|
| Core concept | Layers with dependency rule | Ports and Adapters |
| Diagram | Concentric circles | Hexagon with ports |
| Terminology | Use Cases, Entities, Presenters | Ports, Adapters, Domain |
| Focus | Layer isolation | Interface-driven boundaries |

## Cost

- Every external dependency needs a port interface, increasing file count
- The indirection makes it harder to follow "what actually happens" by reading code linearly
- Requires discipline to prevent ports from leaking infrastructure concerns

## Minimal cut

Identify one external dependency (database, email, file storage). Define a port for it in the domain. Move the implementation to an adapter in an `infra/` directory. When a second entry point or a second implementation for the same port exists, the pattern has paid off.

## Related patterns

- **Clean Architecture** -- Same intent, different diagram shape
- **Adapter** -- Adapters are the concrete implementation of ports
- **Dependency Injection** -- DI wires adapters to the domain core
- **Repository** -- Repository is the persistence port pattern
