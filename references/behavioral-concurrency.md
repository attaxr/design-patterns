# Behavioral & Concurrency Patterns

How objects communicate, and how work gets coordinated.

## Contents
- [Behavioral](#behavioral) — Strategy, Observer, Command, Chain of Responsibility, State, Iterator, Mediator, Memento, Template Method, Visitor, Interpreter
- [Concurrency](#concurrency) — Producer-Consumer, Worker Pool, Actor, Future/Promise, Reactor, Pipeline
- [Choosing between them](#choosing-between-them)

---

## Behavioral

### Strategy
Interchangeable algorithms behind one interface.

**Use when:** a growing conditional selects behavior. `if (provider === "openai")` becomes
`provider.generate()`. Classic fits: AI providers, payment gateways, ranking algorithms,
compression, auth schemes.

**The tell:** the same `switch` on the same value appears in more than one function. That
duplication is what Strategy removes — a single switch in one place is fine and often
clearer.

**TS form:** a function type, usually. `type Generate = (prompt: string) => Promise<Result>`
and a `Record<Kind, Generate>`. Classes only when strategies carry state.

**Go form:** a func type or a one-method interface declared at the consumer.

**Cost:** the behavior is no longer visible at the call site. Naming matters more than
usual.

### Observer
Publish/subscribe. One event, many independent reactions.

**Use when:** a state change must notify subsystems that shouldn't know about each other —
a completed scan updating UI, metrics, logs, and notifications.

**Cost:** this is the pattern most likely to make a system hard to reason about. Control
flow becomes invisible; you can't find "what happens next" by reading. Order of listeners
is usually unspecified. Errors in one listener can strand the rest.

**Mitigations that matter:** a typed event map (not string keys), one place where all
subscriptions are registered, and an explicit decision about whether handlers are sync or
async and what happens when one throws. Prefer explicit calls when there are only two
listeners and they're both yours.

### Command
Represent an action as an object or record, so it can be stored, queued, replayed, undone.

**Use when:** work must outlive the request that created it — background jobs, scheduled
tasks, retryable operations, undo stacks, audit logs of intent.

```ts
type Command =
  | { kind: "runScan"; targetId: string }
  | { kind: "deleteUser"; userId: string };
```

**Serializability is the point.** If a command holds a closure, a DB handle, or a class
instance, it can't cross a queue boundary. Keep commands as plain data and resolve
dependencies in the handler.

**Cost:** two things to maintain per action (the command and its handler) and a dispatch
layer. Worth it only when queueing, retrying, or replaying is real.

### Chain of Responsibility
A request passes through handlers, each of which may handle it or pass it on.

**Use when:** HTTP middleware — auth → validation → authorization → business logic. Also
input sanitization chains, escalation rules, fallback resolution.

**Cost:** implicit ordering. The chain's correctness lives in its assembly order, not in any
one handler, so keep the assembly in one obvious place and comment why each link sits
where it does. Debugging "who swallowed my request" is the recurring pain.

**Note:** you probably already use this via Express/Fastify/`net/http` middleware. Adopting
the framework's form is better than writing your own.

### State
Each state owns its own behavior and legal transitions.

**Use when:** `if (status === "running")` checks appear across the codebase, or when an
illegal transition is a real bug class — orders, scans, payments, auth flows, document
workflows.

```
Pending → Running → Completed → Archived
```

**The value isn't the polymorphism, it's the explicit transition table.** Writing down
which transitions are legal catches bugs that scattered `if`s hide. You can get most of
the benefit from a transition map plus a discriminated union, without a class per state.

**Cost:** more types; transitions become slightly harder to skim as a whole unless you keep
the map centralized.

### Iterator
Sequential access without exposing the underlying structure.

**Use when:** you're building a custom collection, or streaming something too large to
materialize. Generators (`function*`, `AsyncIterator`) are the modern form and handle
backpressure and early termination well.

**Cost:** near zero in TS. In Go, `range`-over-func iterators are now available but a plain
callback or channel is often clearer.

### Mediator
A central object coordinates interactions that would otherwise be many-to-many.

**Use when:** N components each know about the other N−1. The mediator collapses that to
N+1 relationships.

**Cost:** the mediator becomes the place all complexity accumulates. It often grows into a
god object. Consider whether an event bus (Observer) or just fewer components is the better
answer.

### Memento
Capture and restore an object's state without exposing its internals.

**Use when:** undo/redo, checkpointing, optimistic UI rollback.

**Cost:** snapshot size and clone depth. For large state, prefer storing the inverse
operation (Command-based undo) over full snapshots.

### Template Method
A base algorithm with steps subclasses fill in.

**Use when:** several flows share a skeleton and differ in a couple of steps — a scan
pipeline where only parsing differs.

**Cost:** inheritance-based, so it couples subclasses to base-class internals and ordering.
**Prefer the composed form:** pass the varying steps in as functions. That's the same idea
with none of the fragile-base-class problem, and it's the only sensible form in Go.

### Visitor
Add operations to an object tree without modifying the node types.

**Use when:** the node set is stable but operations keep being added — AST compilers,
linters, serializers over a fixed grammar.

**Cost:** the tradeoff is exactly inverted from polymorphism. Adding an operation is easy;
adding a node type means touching every visitor. Choose based on which axis actually
changes.

**TS note:** an exhaustive `switch` over a discriminated union does this with compiler-
checked completeness and far less code. Prefer it unless the node set is open.

### Interpreter
Evaluate a small language or expression tree.

**Use when:** users need to express rules you can't enumerate — query filters, permission
expressions, alert conditions.

**Cost:** you now own a language: parsing, precedence, error messages, sandboxing,
documentation. Consider an existing expression library first, and never `eval` user input.

---

## Concurrency

### Producer-Consumer
Producers enqueue; consumers dequeue and process independently.

**Use when:** work arrives faster than it can be handled, or must survive a restart.
`API → queue → workers → database`.

**Decide explicitly:** queue bounded or unbounded (unbounded is a memory leak with extra
steps), at-least-once or at-most-once delivery, and what happens on repeated failure — see
Dead Letter Queue in `distributed.md`.

**Go form:** buffered channel plus a fixed set of goroutines. Cancel with `context`.

### Worker Pool / Thread Pool
A fixed set of workers draws from shared work.

**Use when:** you need to cap concurrency against a downstream limit — DB connections,
rate-limited APIs, CPU-bound work.

**The cap is the feature.** Unbounded `Promise.all` over a thousand items or a goroutine
per item is how you take down your own database. A pool makes the ceiling explicit.

### Actor
Each actor owns its state and communicates only by messages.

**Use when:** you want to eliminate shared mutable state entirely, and per-entity
serialization is natural — one actor per user, per inventory item, per session.

**Cost:** message-passing everywhere means no atomic operations across actors, and
supervision/restart semantics become your problem. Rarely worth hand-rolling; adopt it only
with a runtime that supports it.

### Future / Promise
A handle on a result that isn't ready yet.

**Use when:** always, in async code — this is the ambient model in TS.

**Watch for:** unhandled rejections, promises created but never awaited, and missing
cancellation. `AbortSignal` in TS and `context.Context` in Go are the cancellation story;
plumb them through from the start rather than retrofitting.

### Reactor / Proactor
Event-loop demultiplexing of I/O events (Reactor) or completion notifications (Proactor).

**Use when:** you're building a runtime. Node *is* a reactor; Go's netpoller sits under the
goroutine scheduler. Understand these to reason about why blocking the loop is fatal in
Node — but you won't implement them.

### Pipeline
Staged transformation, each stage doing one thing.

**Use when:** multi-step processing over a stream — `crawler → parser → analyzer →
classifier → store`.

**Design decisions that matter:** per-stage concurrency (stages rarely want the same
parallelism), backpressure between stages, and where a failed item goes — dropped, retried,
or diverted. Answer all three before building it, because retrofitting backpressure is
painful.

**Go form:** channels between stages; propagate one `context` through all of them so
cancellation unwinds cleanly.

---

## Choosing between them

- **Strategy vs State** — identical structure, different intent. Strategy is chosen by the
  caller and doesn't change itself; State transitions itself based on events. If the object
  decides its own next behavior, it's State.
- **Observer vs Mediator** — Observer is broadcast, publishers unaware of subscribers.
  Mediator is coordination, and it knows all participants.
- **Command vs Strategy** — Command is a request to do something later; Strategy is how to
  do something now. Commands are data and get serialized; strategies are behavior.
- **Chain of Responsibility vs Decorator** — near-identical mechanics. Chain members may
  *stop* the request; decorators always delegate onward.
- **Pipeline vs Chain of Responsibility** — Pipeline transforms data through every stage;
  Chain routes a request until someone handles it.
