# Architectural & Domain Patterns

The shape of the application as a whole, and how the domain is modeled inside it.

## Contents
- [Layering](#layering) — Layered, Clean, Hexagonal, Onion
- [Read/write shape](#readwrite-shape) — CQRS, Event Sourcing
- [Deployment shape](#deployment-shape) — Modular Monolith, Microservices, Serverless
- [UI architecture](#ui-architecture) — MVC, MVP, MVVM
- [Domain-Driven Design building blocks](#domain-driven-design-building-blocks)
- [Choosing](#choosing)

---

## Layering

All four of these encode the same rule: **dependencies point inward, toward the domain.**
Business rules must not import the database driver, the HTTP framework, or the vendor SDK.
The differences between them are mostly vocabulary.

### Layered Architecture
`UI → Application → Domain → Infrastructure`, each layer depending only on the one below.

**Use when:** a conventional application where the team benefits from an obvious, boring
structure.

**Cost:** the classic version lets Domain depend on Infrastructure, which is the thing that
eventually hurts — domain code ends up importing the ORM. Clean/Hexagonal exist to fix
exactly that by inverting the bottom dependency.

### Clean Architecture
Same layers, but infrastructure depends on the domain rather than the reverse. The domain
defines interfaces; infrastructure implements them.

**Use when:** the system will outlive its current database, queue, or cloud provider, and
the domain logic is substantial enough to be worth protecting.

**Cost:** more indirection and more files, and every new feature crosses several layers.
This is real friction on small codebases. The payoff arrives when infrastructure changes or
when the domain needs to be tested without any of it running.

### Hexagonal (Ports & Adapters)
The domain defines **ports** (interfaces); **adapters** connect them to the outside world.

```
API → Port → Domain → Port → Repository → Database
```

**Use when:** you want to swap infrastructure or drive the domain from multiple entry points
— HTTP, CLI, queue consumer, test harness — without duplicating logic.

**The practical test:** you should be able to run the entire domain with in-memory adapters
and no network. If you can't, the ports aren't real yet.

**Relationship to Adapter (the GoF pattern):** identical mechanics at a different scale. See
`creational-structural.md`.

### Onion Architecture
Concentric rings, domain model at the center. Effectively Clean/Hexagonal with different
diagrams. Don't spend time distinguishing them; pick one vocabulary and be consistent.

---

## Read/write shape

### CQRS — Command Query Responsibility Segregation
Separate the write model from the read model.

**Use when:** reads and writes have genuinely different shapes or wildly different volumes
— a normalized write model but denormalized read views, or reads outnumbering writes by
orders of magnitude.

**Cost:** two models to keep in sync, and if the read model is updated asynchronously, users
will observe stale data. That's a product decision, not just a technical one — decide what
"I saved it but don't see it" should look like in the UI before building this.

**Lighter first step:** read replicas, or a materialized view, or just different query
methods on the same repository. Full CQRS with separate stores is a large commitment. The
term is often used for the light version; be explicit about which one you mean.

### Event Sourcing
Store the sequence of events; derive current state by replaying them.

```
Deposited $20, Withdrew $10, Deposited $90  →  balance $100
```

**Use when:** history is part of the domain — audit requirements, financial ledgers,
"why is this record in this state", temporal queries, or the ability to build new read
projections retroactively from data you didn't know you'd need.

**Cost, and it's substantial:**
- Events are immutable, so schema evolution means versioning and upcasting them forever.
- Replay time grows; you'll need snapshots.
- "Just fix the row" is no longer possible — corrections are compensating events.
- Every developer must learn the model before they can safely make a change.

**Require a stated reason.** Chosen for elegance rather than an audit or projection need,
this is the pattern most likely to be regretted. It pairs naturally with CQRS (the read
model is a projection) — expect to adopt both.

---

## Deployment shape

### Modular Monolith
One deployable, strong internal module boundaries. Modules talk through explicit interfaces
and own their data.

**Use when:** almost always, as the starting point. You get the boundaries that make
microservices possible later, without distributed transactions, network failure modes, or
per-service infrastructure.

**Cost:** boundaries are enforced by discipline (and lint rules), not by the network. They
erode unless someone defends them.

### Microservices
Independently deployable services owning their own data.

**Use when:** teams need to deploy independently, or components have genuinely different
scaling or availability profiles. Both of those are organizational and load facts you can
verify — not code-structure preferences.

**Cost:** every in-process call that becomes a network call gains latency, partial failure,
retries, idempotency requirements, and distributed tracing. Cross-service consistency needs
Sagas and Outbox (see `distributed.md`). Local development gets much harder.

**Not a reason to split:** wanting cleaner code boundaries. A modular monolith gives you
that for free.

### Serverless
Functions triggered by events, no managed servers.

**Use when:** spiky or infrequent workloads, glue between managed services, event handlers
where cold-start latency is acceptable.

**Cost:** cold starts, execution time limits, connection pooling against a relational DB
(needs a proxy), harder local testing, and vendor coupling in the trigger configuration.

---

## UI architecture

- **MVC** — Model, View, Controller. Server-rendered apps and classic web frameworks.
- **MVP** — Presenter holds view logic, view is passive. Easier to unit test.
- **MVVM** — ViewModel exposes bindable state; the framework syncs it to the view. The
  natural fit for declarative UI frameworks.

In practice modern frontend work is better served by the patterns in
`frontend-functional.md`. These three matter mostly for reading existing code and for mobile
or desktop frameworks that mandate one of them.

---

## Domain-Driven Design building blocks

These are worth adopting piecemeal even without committing to full DDD.

| Building block | What it is | When it earns its place |
| --- | --- | --- |
| **Entity** | Identity persists through change | Two objects with identical fields are still different things (two users named John) |
| **Value Object** | Immutable, equality by value | `Money`, `EmailAddress`, `DateRange` — stops primitive-obsession bugs, and cheap to adopt |
| **Aggregate** | Consistency boundary with one root | You need a rule like "an order's items must always sum to its total" enforced atomically |
| **Repository** | Persistence behind a domain-shaped interface | The domain shouldn't know about SQL; also the seam for in-memory testing |
| **Domain Service** | Logic spanning several aggregates | The behavior doesn't belong to any one entity |
| **Domain Event** | A record that something meaningful happened | Other parts of the system must react without the origin knowing about them |
| **Specification** | A business rule as a composable object | Rules are reused across query and validation, or combined with and/or |
| **Factory** | Complex, invariant-preserving construction | An entity can't be validly built with a plain constructor |

**Highest value for the lowest cost:** Value Objects and Repository. Both pay off
immediately in almost any codebase.

**Requires the most buy-in:** Aggregates. Getting boundaries wrong causes either lock
contention (too big) or broken invariants (too small). Draw them around what must be
consistent *in the same transaction*, and nothing more.

**A note on ubiquitous language:** the cheapest DDD practice is also the most valuable —
use one domain term consistently in code, tests, docs, and conversation. If the domain says
"inventory", the code says `inventory`, never a synonym. Mixed vocabulary for one concept is
a bug factory, and this costs nothing to enforce.

---

## Choosing

- Start with a **modular monolith** using **Hexagonal** boundaries. This combination defers
  the expensive decisions while keeping them available.
- Add **CQRS** only when read and write shapes actually diverge — and prefer the light
  version first.
- Add **Event Sourcing** only with a stated audit or projection requirement you can point at.
- Split to **microservices** when deploy independence or scaling profiles demand it, along
  boundaries the modular monolith has already proven.
- Adopt **DDD building blocks** individually as the domain gets complicated enough to
  reward them. You don't need the whole methodology to benefit from Value Objects.
