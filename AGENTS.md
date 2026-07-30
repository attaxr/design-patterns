# AGENTS.md -- Design Patterns Skill

This file registers the design-patterns skill for agents that read `AGENTS.md` configuration from the project root. Copy this into your project root or append it to an existing `AGENTS.md`.

## Skill registration

The design-patterns skill is installed at `~/.agents/skills/design-patterns/`. It provides a symptom-driven reference for applying design patterns correctly -- only when the codebase actually has the pressure the pattern resolves.

### Triggers -- agent should invoke this skill when the user:

- Has an `if/else` or `switch` on a type, provider, or status that keeps growing
- Has a constructor with many optional parameters
- Has a vendor SDK shape leaking into domain code
- Has logging, retries, or metrics copy-pasted around calls
- Has work that must be queued, retried, scheduled, or survive a restart
- Has a workflow spanning multiple services that needs rollback
- Has reads and writes that need different models
- Designs module or service boundaries
- Has a flaky dependency where failures cascade
- Frontend state or component structure needs organizing
- Asks how to structure, refactor, or simplify over-engineered code
- Names a pattern -- Strategy, Factory, Builder, Adapter, Repository, CQRS, Event Sourcing, Saga, Outbox, Circuit Breaker, Clean Architecture, Hexagonal

### Reference files

| Pattern                 | File                                    |
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

### Core rule

> Pattern follows pressure. A pattern is justified by a force _already_ present in the codebase, not by a force you imagine arriving later. Applying Strategy to a single implementation, or Repository over one query, buys indirection and pays nothing back.

### Response format

When recommending a pattern, structure the answer like this:

```
Pressure:    what specifically hurts today, in the user's code
Pattern:     the name, and the lighter alternative you rejected
Cost:        what gets worse (indirection, files, consistency model)
Minimal cut: the smallest version worth writing now
Deferred:    the pattern you are deliberately not adding yet, and its trigger
```

### Red flags -- push back on these

- **Pattern stacking** -- Repository wrapping a Service wrapping a Manager wrapping the ORM
- **Interface with one implementation and no test double** -- delete it
- **Singleton as global variable** -- pass the value instead
- **Event Sourcing or CQRS chosen for elegance** -- require stated audit or read-scaling need
- **Microservices to organize a small team** -- modular monolith first
- **Abstraction over one vendor with no exit plan** -- it is just indirection
- **Naming a class after its pattern** (`UserFactoryStrategyImpl`) -- name it for the domain
