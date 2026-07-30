# Repository

Mediate between the domain and data mapping layers, acting like an in-memory collection of domain objects.

## Pressure

Domain objects need persistence, but the domain should not know about SQL, ORMs, or database specifics. Business logic is tangled with data access code. You want to test business logic without a database.

## Solution

Define an interface in the domain layer that speaks the domain's language (findByHost, not findById). Implement it in the infrastructure layer. Domain code depends on the interface; the implementation handles SQL, HTTP, or in-memory storage.

## Implementation

```ts
// Domain-level interface -- speaks the domain's language
interface InventoryRepository {
  findByHost(host: string): Promise<Inventory | null>;
  findByTag(tag: string): Promise<Inventory[]>;
  save(inventory: Inventory): Promise<void>;
  delete(host: string): Promise<void>;
}

// Infrastructure implementation -- handles SQL details
class PostgresInventoryRepository implements InventoryRepository {
  constructor(private pool: Pool) {}

  async findByHost(host: string): Promise<Inventory | null> {
    const result = await this.pool.query(
      "SELECT * FROM inventory WHERE host = $1", [host],
    );
    if (result.rows.length === 0) return null;
    return this.rowToInventory(result.rows[0]);
  }

  async save(inventory: Inventory): Promise<void> {
    // upsert logic
  }

  private rowToInventory(row: any): Inventory {
    return new Inventory(row.host, row.data, new Date(row.scanned_at));
  }
}

// Test implementation -- no database needed
class InMemoryInventoryRepository implements InventoryRepository {
  private items = new Map<string, Inventory>();

  async findByHost(host: string) {
    return this.items.get(host) ?? null;
  }
  async save(inventory: Inventory) {
    this.items.set(inventory.host, inventory);
  }
}
```

## What a Repository is NOT

```ts
// NOT a Repository -- this is a DAL (Data Access Layer)
interface InventoryDal {
  insert(row: InventoryRow): Promise<void>;
  update(id: number, row: Partial<InventoryRow>): Promise<void>;
  deleteById(id: number): Promise<void>;
  findById(id: number): Promise<InventoryRow | null>;
}
```

A proper Repository:
- Takes and returns **domain objects**, not database rows
- Uses domain language in method names (`findByHost`, not `findByHostId`)
- Encapsulates query logic -- a single `findByTag()` may join three tables internally
- The domain does not know whether the implementation uses SQL, REST API, or in-memory map

### Why the distinction matters

```ts
// Domain code using a Repository -- testable without a database
class ScanService {
  constructor(private readonly inventory: InventoryRepository) {}

  async recordFinding(host: string, finding: Finding) {
    const inventory = await this.inventory.findByHost(host);
    if (!inventory) throw new NotFoundError(`no inventory for ${host}`);

    inventory.addFinding(finding);        // domain logic
    inventory.updateRiskScore();           // domain logic
    await this.inventory.save(inventory);  // persistence
  }
}

// Test
const repo = new InMemoryInventoryRepository();
const service = new ScanService(repo);
await service.recordFinding("example.com", finding);
// No database, no network. Test in milliseconds.
```

## The save() contract

Decide on one convention and document it:

| Convention | Behavior |
|-----------|----------|
| `save(entity)` -- insert or update | Repository decides based on existence check |
| `save(entity)` -- insert; `update(entity)` -- update | Explicit at call site |
| `save(entity)` -- always upsert | Simplest; works when identity is stable |

## Common mistakes

1. **Repository returns database types** -- `findByHost()` returns `InventoryRow` instead of `Inventory`. Callers still depend on the DB schema.
2. **Repository that writes but never reads** -- if you only have `save()` and never `findBy*()`, you probably do not need a Repository.
3. **Repository that returns ORM entities** -- ORM entities are infrastructure. Convert them to domain objects at the boundary.
4. **Generic Repository** -- `Repository<T>` with `findById(id)`, `save(entity)`, `delete(id)`. This is a leaky abstraction that forces every entity into the same persistence pattern. Domain-specific methods are more useful.

## Minimal cut

Do not create a Repository interface for every entity on day one. Add the interface when:
- You need to test logic that uses persistence
- You have a second data source (cache, search index, read replica)
- The domain needs to change without changing the persistence layer

## Related patterns

- **Clean / Hexagonal Architecture** -- Repository is the concrete form of a port in Hexagonal Architecture
- **CQRS** -- CQRS often uses separate repositories for reads and writes
- **Unit of Work** -- Coordinates multiple repository saves into a single transaction
