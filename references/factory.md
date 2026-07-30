# Factory

Encapsulate object creation so call sites are decoupled from concrete types.

## Pressure

`new ConcreteThing()` appears in multiple places, or a test cannot substitute a concrete type because the call site calls `new`. The choice of what to build is duplicated across call sites. Adding a new variant means editing every call site instead of one creation point.

## Solution

Move the decision of which concrete type to instantiate behind a single function or interface. Callers receive a product through its interface and never call `new` directly.

## Implementation

### Factory Method -- the most common form

A single function that decides which concrete type to return based on runtime input.

```ts
// Before: every call site picks the provider
function scan(target: string) {
  const provider =
    cfg.provider === "openai"
      ? new OpenAIProvider(cfg.openai)
      : new AnthropicProvider(cfg.anthropic);
  // ...
}

// After: one function owns the decision
function createProvider(kind: ProviderKind, cfg: Config): Provider {
  switch (kind) {
    case "openai":
      return new OpenAIProvider(cfg.openai);
    case "anthropic":
      return new AnthropicProvider(cfg.anthropic);
    case "ollama":
      return new OllamaProvider(cfg.ollama);
  }
}
// Adding a provider touches 1 file, not N call sites.
```

```go
// Go form: a function or a struct holding the factory.
func NewProvider(kind ProviderKind, cfg Config) (Provider, error) {
    switch kind {
    case ProviderOpenAI:
        return openai.New(cfg.OpenAI), nil
    case ProviderAnthropic:
        return anthropic.New(cfg.Anthropic), nil
    default:
        return nil, fmt.Errorf("unknown provider %q", kind)
    }
}
```

### Product type decision: interface or discriminated union

- If callers treat all implementations the same way: use an **interface** (`Provider.generate()`)
- If callers need to handle specific types differently: use a **discriminated union**
- If you started with a union and found every caller uses it the same way: refactor to interface

### Abstract Factory (rarely needed)

A factory that produces a family of related objects that must be used together.

```ts
// When switching DB also switches migration runner and dialect builder.
interface DatabaseFactory {
  createConnection(): Connection;
  createMigrator(): Migrator;
  createQueryBuilder(): QueryBuilder;
}

class PostgresFactory implements DatabaseFactory { /* ... */ }
class SqliteFactory implements DatabaseFactory { /* ... */ }
```

**Cost:** high ceremony. Two levels of indirection before any real work happens. In Go, prefer returning a struct with the family as fields rather than a full interface hierarchy.

```go
type DatabaseComponents struct {
    Conn     *sql.DB
    Dialect  Dialect
    Migrator Migrator
}

func NewDatabaseComponents(driver string) (*DatabaseComponents, error) {
    // switch here
}
```

## When to reach for Factory

- `new Concrete()` appears in 3 or more places
- A test needs to inject a mock and cannot because the call site calls `new`
- The concrete type is determined by config, environment, or feature flag

## Minimal cut

Do not create a factory interface or registry. A single function is enough until you have multiple factories or need runtime registration. An interface wrapper over a switch statement is just indirection.

## Cost

Adds one level of indirection. The concrete type is no longer visible at the call site (this is the point, but readers lose the "what am I getting" at-a-glance).

## Related patterns

- **Builder** -- Factory picks which type. Builder assembles one type incrementally.
- **Strategy** -- Factory creates objects. Strategy selects behavior.
