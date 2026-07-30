# Copilot Instructions -- Design Patterns

## When to suggest design patterns

Only recommend a design pattern when the codebase has a concrete pressure that the pattern resolves. A hypothetical future need is not a justification.

### Triggers that warrant a pattern recommendation

- A `switch` or `if/else` chain on a type discriminator keeps growing
- A constructor or function call has many optional parameters
- A vendor SDK's types or patterns leak into domain-level code
- Logging, retry, or metrics logic is copy-pasted at multiple call sites
- Work needs to survive process restart or be retried reliably
- A multi-service operation needs distributed rollback
- Read models and write models have different shapes and scaling needs
- Business logic directly depends on infrastructure (database driver, HTTP client)
- A named design pattern is mentioned by the developer

### Symptoms that do NOT warrant a pattern

- A single implementation that might have a second one "someday"
- A simple conditional with two branches
- A constructor with two or three parameters

## Symptom to pattern reference

| Symptom in the code                                                          | Pressure                                                 | Pattern                 | Reference file                                                           |
| ---------------------------------------------------------------------------- | -------------------------------------------------------- | ----------------------- | ------------------------------------------------------------------------ |
| `new ConcreteThing()` scattered across call sites                            | Object creation is duplicated and coupled                | Factory                 | `~/.agents/skills/design-patterns/references/factory.md`                 |
| Constructor with 6+ params, many optional                                    | Construction is fragile and unreadable                   | Builder                 | `~/.agents/skills/design-patterns/references/builder.md`                 |
| Vendor SDK types sprinkling through domain code                              | Vendor lock-in leaking across boundaries                 | Adapter                 | `~/.agents/skills/design-patterns/references/adapter.md`                 |
| Callers must invoke 5 methods in the right sequence                          | Complexity hidden behind a simple operation              | Facade                  | `~/.agents/skills/design-patterns/references/facade.md`                  |
| Dependencies created inside functions/classes                                | Tight coupling to concrete implementations               | Dependency Injection    | `~/.agents/skills/design-patterns/references/dependency-injection.md`    |
| `if (provider === "x") ... else if ...` growing                              | Algorithm selection duplicated across call sites         | Strategy                | `~/.agents/skills/design-patterns/references/strategy.md`                |
| One change must notify several unrelated subsystems                          | Implicit coupling between publishers and subscribers     | Observer (Pub/Sub)      | `~/.agents/skills/design-patterns/references/observer.md`                |
| Work must survive restart, be queued, scheduled                              | Imperative actions cannot be stored or retried           | Command                 | `~/.agents/skills/design-patterns/references/command.md`                 |
| Request passes through auth, validate, authorize, and pipeline keeps growing | Fixed pipeline of handlers with early exit               | Chain of Responsibility | `~/.agents/skills/design-patterns/references/chain-of-responsibility.md` |
| `if (status === "running")` checks everywhere                                | State transitions scattered and ungoverned               | State                   | `~/.agents/skills/design-patterns/references/state.md`                   |
| Work arrives faster than it can be processed                                 | Unbounded concurrency risks resource exhaustion          | Producer-Consumer       | `~/.agents/skills/design-patterns/references/producer-consumer.md`       |
| Domain depends on database driver                                            | Business rules leak into infrastructure concerns         | Clean Architecture      | `~/.agents/skills/design-patterns/references/clean-architecture.md`      |
| Needs multiple entry points (HTTP, CLI, queue)                               | Infrastructure coupling with multiple access paths       | Hexagonal Architecture  | `~/.agents/skills/design-patterns/references/hexagonal-architecture.md`  |
| Reads vastly outnumber writes, different shapes                              | One model optimized for neither reads nor writes         | CQRS                    | `~/.agents/skills/design-patterns/references/cqrs.md`                    |
| History matters for audit or rebuilding read models                          | Current state alone cannot answer "how did we get here?" | Event Sourcing          | `~/.agents/skills/design-patterns/references/event-sourcing.md`          |
| Domain objects need persistence without SQL                                  | Persistence concern tangled with business logic          | Repository              | `~/.agents/skills/design-patterns/references/repository.md`              |
| Multi-service operation needs atomic rollback                                | One service cannot transactionally undo another          | Saga                    | `~/.agents/skills/design-patterns/references/saga.md`                    |
| Event published but DB write rolled back                                     | Dual-write hazard between DB and message broker          | Outbox                  | `~/.agents/skills/design-patterns/references/outbox.md`                  |
| One flaky dependency stalls the entire system                                | Cascading failure from a slow downstream                 | Circuit Breaker         | `~/.agents/skills/design-patterns/references/circuit-breaker.md`         |
| One dependency resource exhaustion starves others                            | Shared resource pools across differing reliability       | Bulkhead                | `~/.agents/skills/design-patterns/references/bulkhead.md`                |

## Language idioms

### TypeScript / JavaScript

- Discriminated unions + exhaustive switch over class hierarchies for closed variant sets
- Function types over single-method interfaces for Strategy, Command, Factory
- `Required<T> & Partial<Options>` for builder patterns with optional params
- Higher-order functions for decorator behavior
- Branded types for domain identifiers (`type UserId = string & { __brand: 'UserId' }`)

### Go

- Consumer-defined interfaces, one or two methods wide
- Func types over interfaces for Strategy, Command
- `func WithRetry(next Store) Store` for decorator composition
- Package-level state + `sync.Once` instead of Singleton pattern
- Pass dependencies explicitly rather than using a global container

## Red flags -- do NOT suggest these

- **Pattern stacking**: Repository -> Service -> Manager -> ORM pass-through
- **Interface with exactly one implementation and no test double** -- delete the interface
- **Singleton as a global variable** -- pass the value explicitly
- **CQRS or Event Sourcing without an audit or read-scaling requirement**
- **Microservices for a small team or codebase under 50K lines**
- **Abstraction over a single vendor with no migration planned**
- **Naming classes after the pattern** (`UserFactoryStrategyImpl`)

## Response format when recommending a pattern

Always frame it as:

1. **Pressure** -- what hurts today in concrete terms
2. **Pattern** -- name + the lighter alternative you considered and rejected
3. **Cost** -- what gets worse (indirection, files to navigate, consistency model changes)
4. **Minimal cut** -- the smallest version worth writing right now
5. **Deferred** -- what you are deliberately not adding yet, and what should trigger it

## Reference

This skill's full content is at `~/.agents/skills/design-patterns/SKILL.md`. Pattern reference files in `~/.agents/skills/design-patterns/references/` contain detailed implementation guidance for each pattern.
