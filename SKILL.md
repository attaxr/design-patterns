---
name: design-patterns
description: Use when choosing a design pattern for a coding problem. Use when an if/else or switch on a type, provider, or status keeps growing. Use when a constructor takes many optional parameters. Use when a vendor SDK shape leaks into domain code. Use when logging, retries, or metrics are copy-pasted around calls. Use when work must be queued, retried, scheduled, or survive a restart. Use when a workflow spans multiple services and needs rollback. Use when reads and writes need different models. Use when designing module or service boundaries. Use when a dependency is flaky and failures cascade. Use when frontend state or component structure needs organizing. Use when asked how to structure, refactor, or simplify over-engineered code. Use when a named pattern comes up — Strategy, Factory, Builder, Adapter, Decorator, Repository, CQRS, event sourcing, saga, outbox, circuit breaker, clean architecture, hexagonal. Use when writing new modules, adapters, providers, or agents.
---

# Design Patterns

Patterns are compressed answers to recurring pressures in code. The skill here is not
reciting them — it is matching a pattern to a pressure that actually exists, and refusing
to add one when it doesn't.

## The core rule: pattern follows pressure

A pattern is justified by a force already present in the codebase, not by a force you
imagine arriving later. Applying `Strategy` to a single implementation, or `Repository`
over one query, buys indirection and pays nothing back.

Before recommending any pattern, be able to finish this sentence concretely:

> "Today, changing X requires editing Y places / testing X requires Z, and this pattern
> removes that."

If the sentence needs a hypothetical future to work, say so plainly and recommend the
direct implementation instead. Naming the pattern you're *deliberately deferring* is more
useful than installing it — it tells the next reader where the seam will go.

Two forces genuinely justify a pattern pre-emptively:
- **A second implementation is already funded and specified** (not "we might add Azure").
- **The pattern is the cheap version.** A `Chain of Responsibility` for HTTP middleware
  or a `Repository` interface in Go costs almost nothing and is idiomatic; deferring it
  costs more than adopting it.

## Workflow

1. **Name the pressure in the user's own terms.** Not "you need a Factory" but "adding a
   provider means touching four files". Patterns are the answer; the pressure is the
   question, and the question is what the user can verify.
2. **Route to the relevant reference file** (below) and pick the pattern. Consider at
   least two candidates — most pressures have a lighter and a heavier answer.
3. **State the cost out loud.** Every pattern trades directness for flexibility. Say what
   is lost: indirection depth, harder stack traces, more files, eventual consistency.
4. **Implement the smallest version.** One interface, one concrete implementation, no
   abstract base class, no registry until there are three entries. Patterns grow well;
   they shrink badly.
5. **Prefer the language idiom over the textbook form.** See "Language idiom" below.

## Symptom → pattern

This table is the fastest path. Read it as diagnosis, not prescription — confirm the
symptom is real before applying the fix.

| Symptom in the code | Likely pattern | Reference |
| --- | --- | --- |
| `if (provider === "openai") … else if …` and growing | Strategy | behavioral |
| Constructor with 6+ params, many optional | Builder, or an options object | creational |
| `new ConcreteThing()` scattered across call sites | Factory | creational |
| Vendor SDK shape leaking into domain code | Adapter / Port | structural |
| Logging, retries, metrics copy-pasted around calls | Decorator, Middleware | structural |
| Caller must invoke five methods in the right order | Facade | structural |
| `status` string checked in many branches | State machine | behavioral |
| Request needs auth → validate → authorize → handle | Chain of Responsibility | behavioral |
| One change must notify several unrelated subsystems | Observer / Event bus | behavioral |
| Work must survive restart, be retried or scheduled | Command + queue, Producer-Consumer | concurrency |
| Multi-stage transform over a stream of items | Pipeline | concurrency |
| One flaky dependency stalls everything | Circuit Breaker + Bulkhead | distributed |
| Multi-service operation needs rollback | Saga | distributed |
| Event published but DB write rolled back (or vice versa) | Outbox | distributed |
| Same message processed twice | Inbox / idempotency key | distributed |
| Reads vastly outnumber writes, different shapes | CQRS | architectural |
| Audit trail or "how did it get this way" matters | Event Sourcing | architectural |
| Business rules import the database driver | Clean / Hexagonal Architecture | architectural |
| Repeated hot read against a slow store | Cache Aside | distributed |
| Prop drilling, or global-ish UI state | Provider, Signals, Flux | frontend |
| Component mixes fetching with rendering | Container/Presentational, Hooks | frontend |

## Reference files

Read only what the problem needs — each file is self-contained.

- `references/creational-structural.md` — object creation and composition: Singleton,
  Factory, Abstract Factory, Builder, Prototype, Adapter, Bridge, Composite, Decorator,
  Facade, Flyweight, Proxy.
- `references/behavioral-concurrency.md` — communication and coordination: Strategy,
  Observer, Command, Chain of Responsibility, State, Iterator, Mediator, Memento,
  Template Method, Visitor, Interpreter, plus Producer-Consumer, Worker Pool, Actor,
  Future, Pipeline.
- `references/architectural.md` — application shape: Layered, Clean, Hexagonal, Onion,
  CQRS, Event Sourcing, Modular Monolith, Microservices, Serverless, MVC/MVVM, and the
  DDD building blocks (Entity, Value Object, Aggregate, Repository, Domain Event,
  Specification).
- `references/distributed.md` — cross-service reliability: Message Bus, Saga, Outbox,
  Inbox, Retry, Dead Letter Queue, Competing Consumers, Circuit Breaker, Bulkhead,
  Sidecar, Ambassador, Strangler Fig, Leader Election, Sharding, cache strategies.
- `references/frontend-functional.md` — UI architecture and functional composition:
  Container/Presentational, Compound Components, Render Props, Hooks, Provider, Flux,
  Signals, plus pure functions, composition, HOFs, currying, monadic error handling.

## Language idiom

The textbook form of a pattern is Java's form. Reproducing it elsewhere is a code smell in
its own right.

**TypeScript**
- Prefer a discriminated union + exhaustive `switch` over a class hierarchy for closed
  sets of variants. The compiler enforces completeness; polymorphism doesn't.
- Strategy, Command, and Factory are usually functions and function types, not classes.
  `type Handler = (input: Input) => Promise<Output>` beats an interface with one method.
- Builder is often just `Required<T> & Partial<Options>` with a defaults merge. Reach for a
  fluent builder only when construction has ordering rules or validation between steps.
- Decorator is a higher-order function wrapping a function, or a Proxy — not necessarily
  the `@decorator` syntax.
- Type every seam. An adapter whose return type is loose defeats its own purpose: the
  vendor's shape leaks through as `unknown` handling at every call site.

**Go**
- Interfaces are discovered, not declared up front. Define the interface at the *consumer*,
  keep it one or two methods wide, and let implementations satisfy it implicitly.
- Strategy and Command are func types. Observer is usually a channel. Producer-Consumer is
  goroutines plus a buffered channel — no framework needed.
- Decorator is a func returning the same interface (`func WithRetry(next Store) Store`).
  This composes cleanly and is the standard `http.Handler` middleware shape.
- Skip Singleton; use package-level state with `sync.Once`, or better, pass dependencies
  explicitly. Skip Abstract Factory entirely — it has no idiomatic Go form.
- Errors are values: wrap with `%w` and inspect with `errors.Is`/`errors.As` rather than
  building an error-type hierarchy.

**Both**
- Dependency injection means passing dependencies as parameters. A container is optional
  and usually unnecessary below a few dozen services.
- Composition over inheritance is not a preference here, it's the default. If a solution
  needs a three-level class hierarchy, look for the composed version first.

## When recommending a pattern

Answer in this shape — it keeps the recommendation falsifiable:

```
Pressure:    what specifically hurts today, in the user's code
Pattern:     the name, and the lighter alternative you rejected
Cost:        what gets worse (indirection, files, consistency model)
Minimal cut: the smallest version worth writing now
Deferred:    the pattern you're deliberately not adding yet, and its trigger
```

The `Deferred` line matters most on greenfield code, where the temptation is to install
the whole catalogue on day one.

## Red flags

Push back when you see these, even if the user asked for the pattern by name:

- **Pattern stacking.** Repository wrapping a Service wrapping a Manager wrapping the ORM,
  each layer passing data through unchanged. Collapse the pass-throughs.
- **Interface with exactly one implementation and no test double.** Delete the interface
  until the second implementation exists.
- **Singleton as a global variable in disguise.** It hides dependencies from the signature
  and serializes tests. Almost always pass the value instead.
- **Event Sourcing or CQRS chosen for elegance.** Both impose real cost: replay logic,
  schema evolution of events, eventual consistency users will notice. Require a stated
  audit or read-scaling need.
- **Microservices to organize a small team's code.** A modular monolith gives the same
  boundaries without the network. Split when deploy cadence or scaling profiles diverge.
- **Abstraction over something with one vendor and no exit plan.** A "storage abstraction"
  used only with S3 is S3 with extra steps.
- **Naming a class after its pattern** (`UserFactoryStrategyImpl`). Name it for what it
  does in the domain; the pattern is an implementation detail.
