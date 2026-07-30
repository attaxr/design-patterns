---
name: design-patterns
description: Use when choosing a design pattern for a coding problem. Use when an if/else or switch on a type, provider, or status keeps growing. Use when a constructor takes many optional parameters. Use when a vendor SDK shape leaks into domain code. Use when logging, retries, or metrics are copy-pasted around calls. Use when work must be queued, retried, scheduled, or survive a restart. Use when a workflow spans multiple services and needs rollback. Use when reads and writes need different models. Use when designing module or service boundaries. Use when a dependency is flaky and failures cascade. Use when frontend state or component structure needs organizing. Use when asked how to structure, refactor, or simplify over-engineered code. Use when a named pattern comes up (Strategy, Factory, Builder, Adapter, Repository, CQRS, Event Sourcing, Saga, Outbox, Circuit Breaker, Clean Architecture, Hexagonal). Use when writing new modules, adapters, providers, or agents.
category: engineering
type: reference
risk: medium
source: curated
date_added: "2025-05-01"
---

# Design Patterns

Patterns are compressed answers to recurring pressures in code. This skill maps **concrete code smells** to **the right pattern** and gives you implementation templates that work in TypeScript and Go.

## The core rule: pattern follows pressure

A pattern is justified by a force **already present** in the codebase, not by a force you imagine arriving later. Before recommending any pattern, point to the actual code that hurts.

> "Today, changing X requires editing Y places and this pattern removes that."

If that sentence needs a hypothetical future, say so plainly and recommend the direct implementation instead. Naming the pattern you are deliberately deferring is more useful than installing it.

## Symptom to Pattern (The 20 core patterns)

Read this left-to-right: find the symptom in your code, then read the pattern. Each pattern has its own reference file with full implementation guidance.

| Symptom in the code                                                                                 | Pressure                                                 | Pattern                     | Reference                               |
| --------------------------------------------------------------------------------------------------- | -------------------------------------------------------- | --------------------------- | --------------------------------------- |
| `new ConcreteThing()` scattered across call sites, or test cannot substitute type                   | Object creation is duplicated and coupled                | **Factory**                 | `references/factory.md`                 |
| Constructor with 6+ params, many optional, or steps must happen in order                            | Construction is fragile and unreadable                   | **Builder**                 | `references/builder.md`                 |
| Vendor SDK types sprinkling through domain code, or swapping providers means rewriting everything   | Vendor lock-in leaking across boundaries                 | **Adapter**                 | `references/adapter.md`                 |
| Callers must invoke 5 methods in the right sequence, or a subsystem is too complex to use directly  | Complexity hidden behind a simple operation              | **Facade**                  | `references/facade.md`                  |
| Dependencies are created inside functions/classes, making testing and substitution impossible       | Tight coupling to concrete implementations               | **Dependency Injection**    | `references/dependency-injection.md`    |
| `if (provider === "x") ... else if ...` and the list keeps growing                                  | Algorithm selection duplicated across call sites         | **Strategy**                | `references/strategy.md`                |
| One change must notify several unrelated subsystems, or polling replaces push notification          | Implicit coupling between publishers and subscribers     | **Observer (Pub/Sub)**      | `references/observer.md`                |
| Work must survive restart, be queued, scheduled, or undone                                          | Imperative actions cannot be stored or retried           | **Command**                 | `references/command.md`                 |
| Request must pass through auth, validate, authorize, handle -- and middleware keeps growing         | Fixed pipeline of handlers with early exit               | **Chain of Responsibility** | `references/chain-of-responsibility.md` |
| `if (status === "running")` checks everywhere, illegal states are representable                     | State transitions scattered and ungoverned               | **State**                   | `references/state.md`                   |
| Work arrives faster than it can be processed, or must survive a restart                             | Unbounded concurrency risks resource exhaustion          | **Producer-Consumer**       | `references/producer-consumer.md`       |
| Domain depends on database driver, or you cannot test business logic without infrastructure running | Business rules leak into infrastructure concerns         | **Clean Architecture**      | `references/clean-architecture.md`      |
| Same pressure, different framing. The domain needs multiple entry points (HTTP, CLI, queue)         | Infrastructure coupling with multiple access paths       | **Hexagonal Architecture**  | `references/hexagonal-architecture.md`  |
| Reads vastly outnumber writes, read and write models have different shapes                          | One model optimized for neither reads nor writes         | **CQRS**                    | `references/cqrs.md`                    |
| History matters for audit, or you need to rebuild read models from past events                      | Current state alone cannot answer "how did we get here?" | **Event Sourcing**          | `references/event-sourcing.md`          |
| Domain objects need persistence but should not know SQL                                             | Persistence concern tangled with business logic          | **Repository**              | `references/repository.md`              |
| Multi-service operation needs atomic rollback across services                                       | One service cannot transactionally undo another          | **Saga**                    | `references/saga.md`                    |
| Event published but DB write rolled back (or vice versa)                                            | Dual-write hazard between DB and message broker          | **Outbox**                  | `references/outbox.md`                  |
| One flaky dependency stalls the entire system                                                       | Cascading failure from a slow downstream                 | **Circuit Breaker**         | `references/circuit-breaker.md`         |
| One dependency resource exhaustion starves other dependencies                                       | Shared resource pools across differing reliability       | **Bulkhead**                | `references/bulkhead.md`                |

## Workflow for recommending a pattern

1. **Name the pressure in the user's own terms** -- not "you need a Factory" but "adding a provider means editing four files"
2. **Route to the reference file** and pick the pattern. Consider at least two candidates.
3. **State the cost** -- what gets worse (indirection depth, files, consistency model)
4. **Implement the smallest version** -- one interface, one concrete, no abstract base
5. **Prefer language idiom** -- adapt the textbook form to the user's language

### Response format

```
Pressure:    what specifically hurts today
Pattern:     the name, and the lighter alternative you rejected
Cost:        what gets worse (indirection, files, consistency model)
Minimal cut: the smallest version worth writing now
Deferred:    the pattern you are deliberately not adding yet, and why
```

## Reference files

Each pattern has its own standalone reference file with implementation guidance in TypeScript and Go, common mistakes, and related patterns.

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

## Language idiom

The textbook form of a pattern is Java's form. Reproducing it elsewhere is a code smell.

**TypeScript:**

- Prefer discriminated unions and exhaustive `switch` over class hierarchies for closed variant sets
- Strategy, Command, Factory map to function types, not classes: `type Handler = (input: Input) => Promise<Output>`
- Builder is often `Required<T> & Partial<Options>` with a defaults merge
- Decorator is a higher-order function or Proxy
- Type every seam -- an adapter with loose return types defeats its purpose

**Go:**

- Interfaces at the consumer, one or two methods wide, satisfied implicitly
- Strategy and Command map to func types. Observer maps to channels.
- Decorator maps to `func WithRetry(next Store) Store` (standard http.Handler middleware shape)
- Skip Singleton -- use `sync.Once` or pass explicitly
- Skip Abstract Factory -- no idiomatic Go form. Return a struct of related funcs instead.

**Both:**

- Dependency injection means passing dependencies as parameters. A container is optional and rarely needed below a few dozen services.
- Composition over inheritance is the default. If you need a three-level class hierarchy, look for the composed version first.

## Red flags

Push back on these even if the user asks for the pattern by name:

- **Pattern stacking** -- Repository wrapping a Service wrapping a Manager wrapping the ORM. Collapse them.
- **Interface with exactly one implementation and no test double** -- delete it until the second exists.
- **Singleton as a disguised global variable** -- hides dependencies from signatures and serializes tests.
- **Event Sourcing or CQRS chosen for elegance** -- both impose real cost (replay, schema evolution, eventual consistency). Require a stated audit or read-scaling need.
- **Microservices for a small team's code** -- modular monolith gives the same boundaries without network cost.
- **Abstraction over one vendor with no exit plan** -- "storage abstraction" used only with S3 is S3 with extra steps.
- **Naming classes after their pattern** (`UserFactoryStrategyImpl`) -- name it for what it does in the domain.
