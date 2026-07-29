# Distributed, Integration & Cloud Patterns

Patterns for systems where calls cross a network — where failure is partial, ordering is
weak, and messages arrive twice.

## Contents
- [Messaging](#messaging) — Message Bus, Event Bus, Event-Driven Architecture, Request-Reply, Competing Consumers
- [Consistency](#consistency) — Saga, Outbox, Inbox / idempotency
- [Failure handling](#failure-handling) — Retry, Dead Letter Queue, Circuit Breaker, Bulkhead, Timeout
- [Infrastructure](#infrastructure) — Sidecar, Ambassador, Service Mesh, Strangler Fig, Leader Election, Sharding
- [Caching](#caching) — Cache Aside, Read Through, Write Through
- [The minimum viable set](#the-minimum-viable-set)

---

## Messaging

### Message Bus / Event Bus
Shared transport that decouples producers from consumers (NATS, Kafka, SQS, RabbitMQ).

**Distinguish two things people conflate:**
- **Commands** — directed at one consumer, imperative, expected to be handled ("run this
  scan"). Exactly one handler.
- **Events** — broadcast facts about the past, no expectation of who cares ("scan
  completed"). Zero or more handlers.

Getting this wrong produces events that secretly require a specific consumer, which is a
hidden coupling that breaks the moment that consumer is down.

**Cost:** control flow leaves the codebase. You can no longer read a call stack to
understand what happens; you need tracing and a documented list of topics and their
consumers. Keep that list somewhere findable.

### Event-Driven Architecture
Services react to events rather than calling each other directly.

**Use when:** you want temporal decoupling — a consumer can be down without failing the
producer — and you'll add consumers over time without touching producers.

**Cost:** eventual consistency becomes visible to users, debugging spans services, and
event schema changes affect consumers you may not know about. Version events from the start.

### Request-Reply
RPC semantics over messaging — a correlation ID and a reply subject.

**Use when:** you need a response but want the transport's routing, load-balancing, or
location transparency.

**Cost:** you've rebuilt synchronous coupling on async plumbing, and now own timeouts and
correlation. If every interaction is request-reply, use HTTP or gRPC instead.

### Competing Consumers
Several instances consume from the same queue; each message goes to one of them.

**Use when:** scaling throughput horizontally. This is how worker pools scale across
processes.

**Cost:** per-message ordering is lost across consumers. If order matters, you need
partitioning by key (Kafka partitions, NATS subject sharding) — and then a slow key blocks
its own partition.

---

## Consistency

### Saga
A multi-step business transaction across services, with compensating actions instead of
rollback.

```
Create Order → Reserve Stock → Charge Card → Ship
Shipping fails → Refund → Release Stock
```

**Use when:** an operation spans services that can't share a database transaction.

**The hard part isn't the happy path.** It's that compensations can also fail, and some
actions can't be undone (an email is sent; a physical item shipped). Plan for
compensation retries, and for a manual intervention path when compensation exhausts.

**Two flavors:**
- *Orchestration* — a coordinator drives the steps. Easier to understand and debug; the
  coordinator is a dependency.
- *Choreography* — each service reacts to the previous event. More decoupled; nobody knows
  the whole flow, which makes failures hard to diagnose.

Prefer orchestration unless you have a strong reason. Being able to answer "where is this
order stuck?" is worth a lot.

### Outbox
Write the event to the same database transaction as the state change; a background publisher
ships it to the broker afterwards.

```
BEGIN → write state + write outbox row → COMMIT → publisher → broker
```

**Use when:** you publish events after a DB write — which is nearly always in an
event-driven system.

**Why it's near-mandatory:** without it, a crash between commit and publish loses the event
silently, and a crash between publish and commit emits an event for state that doesn't
exist. Both are the kind of bug that surfaces once a month and takes a week to find. There is
no correct version of "write to DB, then publish" without something like this.

**Cost:** a table, a publisher process, and at-least-once delivery — which is what makes the
Inbox pattern necessary downstream.

### Inbox / Idempotency
Record processed message IDs and skip duplicates.

**Use when:** always, with at-least-once delivery. Assume every message will eventually
arrive twice.

**Implementation:** a unique constraint on `(consumer, message_id)` is usually enough. For
naturally idempotent operations (setting a value, upserting), you may not need the table —
but confirm the operation really is idempotent, including its side effects. Sending an email
twice is not idempotent even if the DB write is.

---

## Failure handling

### Retry
Re-attempt a failed operation.

**Only retry what's safe to retry.** Retrying a non-idempotent write duplicates it. Retrying
a `400` never succeeds. Distinguish transient (timeout, `503`, connection reset) from
permanent (validation, auth, `404`) and only retry the former.

**Use exponential backoff with jitter.** Fixed-interval retries from many clients
synchronize into a thundering herd that keeps a recovering service down. Always cap total
attempts and total elapsed time.

### Dead Letter Queue
Messages that fail repeatedly go to a separate queue.

**Use when:** any durable queue. Without a DLQ, one poison message either blocks the queue
or is silently dropped.

**The part teams forget:** a DLQ nobody monitors is a silent failure with extra steps. It
needs an alert on depth and a documented replay path.

### Circuit Breaker
After a failure threshold, stop calling a dependency; fail fast; probe periodically to
recover.

**Use when:** calling anything that can be slow or down, especially where a queued caller
holds a resource (connection, goroutine, thread) while waiting.

**Why it matters more than it sounds:** without it, a slow dependency consumes every
available worker waiting on timeouts, and one broken downstream service takes down a
healthy upstream one. This is the standard cascading-failure story.

**Requires a fallback decision:** when open, do you return cached data, a degraded
response, or an error? Answer per call site, not globally.

### Bulkhead
Isolate resources per dependency so exhaustion in one doesn't starve the others.

**Use when:** one process talks to several dependencies of differing reliability. Separate
connection pools or concurrency limits per dependency mean the flaky one can't consume all
capacity.

**Pairs with Circuit Breaker:** the bulkhead limits the damage, the breaker stops the
bleeding.

### Timeout
Every network call needs one.

Listed as a pattern because it's the most commonly omitted one. A call with no timeout has
an infinite one, and that's how connection pools exhaust. Set timeouts shorter than the
caller's timeout, and propagate cancellation (`AbortSignal`, `context.Context`) so
abandoned work actually stops.

---

## Infrastructure

### Sidecar
A helper container beside the main one — log shipping, proxying, secret rotation.

**Use when:** cross-cutting infrastructure concerns shouldn't be compiled into the app.

**Cost:** resource overhead per pod and coupled lifecycles.

### Ambassador
A sidecar specifically proxying outbound calls, handling retries, TLS, and service
discovery on the app's behalf.

**Use when:** you want that logic out of application code, or across polyglot services.

### Service Mesh
Ambassadors everywhere, centrally configured — mTLS, retries, traffic shaping,
observability.

**Use when:** many services in several languages, and you need uniform policy.

**Cost:** significant operational complexity and an extra hop. Not worth it for a handful of
services.

### Strangler Fig
Incrementally route traffic from a legacy system to a new one, path by path, until the old
one is unused.

**Use when:** replacing a system that can't be rewritten in one step — which is most of
them.

**Why this is usually right:** it keeps the system working throughout, makes progress
measurable, and lets you stop or reverse at any point. The alternative — a parallel rewrite
with a cutover date — is the well-documented failure mode.

**Cost:** both systems run at once, and you'll need a routing layer plus a plan for shared
data during the transition.

### Leader Election
One instance among many takes a coordinating role.

**Use when:** a scheduled job or a stateful coordinator must run exactly once across a
cluster.

**Cost:** don't hand-roll it. Use the primitives your platform provides (Kubernetes leases,
etcd, Consul, a DB advisory lock). Split-brain from a naive implementation is a data-loss
bug.

### Sharding / Partitioning
Split data across nodes by key.

**Use when:** one node can no longer hold or serve the data.

**Cost:** the shard key choice is close to irreversible and determines whether cross-shard
queries and rebalancing are painful. Cross-shard transactions largely stop being available.
Exhaust vertical scaling, read replicas, and archiving first.

---

## Caching

| Strategy | How it works | Best when | Watch out for |
| --- | --- | --- | --- |
| **Cache Aside** | App checks cache, on miss loads from DB and populates | Read-heavy, tolerant of some staleness | Thundering herd on a hot key expiry — use a lock or stale-while-revalidate |
| **Read Through** | Cache itself loads on miss | You want caching invisible to app code | Less control over the load path and its timeouts |
| **Write Through** | Write to cache and DB together | Consistency matters more than write latency | Every write pays cache latency |
| **Write Behind** | Write to cache, flush to DB async | Write-heavy, some loss tolerable | Data loss on cache failure |

**The recurring bug is invalidation, not population.** Decide up front how a cached entry
becomes wrong and how it gets fixed — TTL, event-driven invalidation, or versioned keys.
"We'll invalidate it when we update" is how stale data ships.

---

## The minimum viable set

For a system with a queue and more than one service, these five aren't optional. Everything
else here is situational.

1. **Timeouts** on every network call, with cancellation propagated.
2. **Retry with backoff and jitter**, only on transient errors, only on idempotent
   operations.
3. **Outbox** for any event published alongside a DB write.
4. **Inbox / idempotency** on every consumer, because delivery is at-least-once.
5. **Dead Letter Queue with an alert**, so poison messages surface instead of vanishing.

Add **Circuit Breaker** and **Bulkhead** as soon as a dependency's reliability differs from
your own. Add **Saga** the first time a workflow spans services.
