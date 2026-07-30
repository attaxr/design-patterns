# CLAUDE.md -- Design Patterns Skill Config

Copy this file to your project root as `CLAUDE.md` or append its contents to an existing one.
It causes Claude to load the design-patterns skill and reference its guidance automatically.

## Design patterns -- when to consult

Before recommending any design pattern, check whether the codebase actually has the
pressure that pattern resolves. Use the symptom-to-pattern table below.

### Symptom to pattern

| Symptom                                                  | Pattern                   | Reference                                                                    |
| -------------------------------------------------------- | ------------------------- | ---------------------------------------------------------------------------- |
| `if (provider === "openai") ... else if ...` and growing | Strategy                  | `references/strategy.md`                                                     |
| Constructor with 6+ params, many optional                | Builder / options object  | `references/builder.md`                                                      |
| `new ConcreteThing()` scattered across call sites        | Factory                   | `references/factory.md`                                                      |
| Vendor SDK shape leaking into domain code                | Adapter / Port            | `references/adapter.md`                                                      |
| Callers must call 5 methods in sequence                  | Facade                    | `references/facade.md`                                                       |
| Dependencies created inside functions/classes            | Dependency Injection      | `references/dependency-injection.md`                                         |
| Logging, retries, metrics copy-pasted                    | Decorator / Middleware    | `references/adapter.md` (see related)                                        |
| Work must survive restart or be retried                  | Command + queue           | `references/command.md`                                                      |
| Multi-service operation needs rollback                   | Saga                      | `references/saga.md`                                                         |
| Event published but DB write rolled back                 | Outbox                    | `references/outbox.md`                                                       |
| Reads >> writes, different shapes                        | CQRS                      | `references/cqrs.md`                                                         |
| Business rules import the database driver                | Clean / Hexagonal         | `references/clean-architecture.md` or `references/hexagonal-architecture.md` |
| Domain objects need persistence without SQL              | Repository                | `references/repository.md`                                                   |
| Flaky dependency stalls the system                       | Circuit Breaker           | `references/circuit-breaker.md`                                              |
| Resource contention between dependencies                 | Bulkhead                  | `references/bulkhead.md`                                                     |
| Work arrives faster than processed                       | Producer-Consumer         | `references/producer-consumer.md`                                            |
| Prop drilling or global-ish UI state                     | Provider / Signals / Flux | See frontend-functional patterns                                             |
| `if (status === "running")` checks everywhere            | State                     | `references/state.md`                                                        |
| One change notifies unrelated subsystems                 | Observer (Pub/Sub)        | `references/observer.md`                                                     |
| Request pipeline keeps growing                           | Chain of Responsibility   | `references/chain-of-responsibility.md`                                      |
| History matters for audit                                | Event Sourcing            | `references/event-sourcing.md`                                               |

### Core principle

**Pattern follows pressure.** Do not recommend a pattern unless you can concretely
finish this sentence: "Today, changing X requires editing Y places / testing X requires Z,
and this pattern removes that."

### Response structure

```
Pressure:    what hurts today, in the user's code
Pattern:     the name, and the lighter alternative you rejected
Cost:        what gets worse (indirection, files, consistency)
Minimal cut: the smallest version worth writing now
Deferred:    the pattern you are deliberately not adding yet, and its trigger
```

### Language idioms (abbreviated)

**TypeScript:**

- Prefer discriminated unions + exhaustive `switch` over class hierarchies
- Strategy, Command, Factory map to function types, not classes
- Decorator maps to higher-order function or Proxy

**Go:**

- Interfaces at the consumer, one or two methods wide
- Strategy and Command map to func types; Observer maps to channels
- Decorator maps to `func WithRetry(next Store) Store`
- Skip Singleton -- use `sync.Once` or pass explicitly
- Skip Abstract Factory -- no idiomatic Go form

### Red flags

- Pattern stacking (pass-through layers)
- Interface with one implementation and no test double
- Singleton as a disguised global variable
- Event Sourcing or CQRS chosen for elegance (real cost)
- Microservices for a small team's code
- Abstraction over one vendor with no exit plan
- Naming a class after its pattern (`UserFactoryStrategyImpl`)

### Skill location

The full skill lives at `~/.agents/skills/design-patterns/SKILL.md`. Reference files
in `~/.agents/skills/design-patterns/references/` contain detailed pattern descriptions
with code examples.

| Pattern                 | Reference file                          |
| ----------------------- | --------------------------------------- |
| Factory                 | `references/factory.md`                 |
| Builder                 | `references/builder.md`                 |
| Adapter                 | `references/adapter.md`                 |
| Facade                  | `references/facade.md`                  |
| Dependency Injection    | `references/dependency-injection.md`    |
| Strategy                | `references/strategy.md`                |
| Observer (Pub/Sub)      | `references/observer.md`                |
| Command                 | `references/command.md`                 |
| Chain of Responsibility | `references/chain-of-responsibility.md` |
| State                   | `references/state.md`                   |
| Producer-Consumer       | `references/producer-consumer.md`       |
| Clean Architecture      | `references/clean-architecture.md`      |
| Hexagonal Architecture  | `references/hexagonal-architecture.md`  |
| Repository              | `references/repository.md`              |
| CQRS                    | `references/cqrs.md`                    |
| Event Sourcing          | `references/event-sourcing.md`          |
| Saga                    | `references/saga.md`                    |
| Outbox                  | `references/outbox.md`                  |
| Circuit Breaker         | `references/circuit-breaker.md`         |
| Bulkhead                | `references/bulkhead.md`                |
