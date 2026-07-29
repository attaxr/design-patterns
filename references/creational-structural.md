# Creational & Structural Patterns

How objects get made, and how they fit together.

## Contents
- [Creational](#creational) — Singleton, Factory Method, Abstract Factory, Builder, Prototype
- [Structural](#structural) — Adapter, Bridge, Composite, Decorator, Facade, Flyweight, Proxy
- [Choosing between them](#choosing-between-them)

---

## Creational

### Singleton
One instance, globally reachable.

**Legitimate uses:** process-wide resources with genuinely one instance — a connection
pool, a metrics registry, a loaded config. Note that all three are things you could just
as easily construct once in `main` and pass down.

**Cost:** hides a dependency from the constructor signature, so callers can't see or
substitute it. Tests share mutable state and become order-dependent.

**Prefer instead:** construct once at the entry point, pass explicitly. In Go, package
state with `sync.Once` if you must. In TS, a module-level `const` is already a singleton —
you rarely need the `getInstance()` ceremony.

### Factory Method
A function or method decides which concrete type to build.

**Use when:** the choice of implementation depends on runtime input (a config string, a
URL scheme, a feature flag), and you want that decision in exactly one place.

```ts
// The switch exists once, here, instead of at every call site.
function createProvider(kind: ProviderKind): Provider {
  switch (kind) {
    case "openai": return new OpenAIProvider(cfg);
    case "anthropic": return new AnthropicProvider(cfg);
  }
}
```

**Signal it's needed:** `new ConcreteThing()` appears in more than two or three places, or
a test needs to substitute the concrete type and can't.

**Cost:** minimal. This is one of the cheapest patterns and rarely regretted.

### Abstract Factory
A factory that produces a *family* of related objects that must be used together.

**Use when:** switching one implementation forces switching several — e.g. a Postgres
driver must pair with a Postgres migration runner and a Postgres dialect builder, and
mixing families is a bug.

**Cost:** high ceremony. Two levels of indirection before any real work happens.

**Reality check:** most cases people reach for this are a single Factory returning a
struct of related functions. In Go, return a struct with the family as fields. Only use the
full form when the families are large and mixing them is genuinely a correctness issue.

### Builder
Construct a complex object step by step.

**Use when:** construction has many optional parts, or intermediate validation, or a
required order. Query builders and HTTP request builders are the archetypes.

```ts
new UserBuilder().name("John").age(30).admin().build()
```

**Cost:** an extra type to maintain, plus the risk of a half-built object escaping if
`build()` isn't the only exit.

**Lighter alternative first:** an options object with defaults.
`createUser({ name, age, admin: true })` handles most "too many parameters" cases without a
builder. Escalate to a real builder when steps constrain each other — when calling `.limit()`
before `.from()` should be impossible, types can enforce it via a staged builder.

### Prototype
Create new objects by cloning an existing one.

**Use when:** initialization is expensive and instances differ slightly from a template —
pre-warmed configs, deep-copied default documents.

**Cost:** clone semantics are a trap. Shallow vs deep matters, and JS `structuredClone`
drops functions and class identity. Be explicit about which.

**Modern form:** usually just spreading or a `clone()` method, not a registry of prototypes.

---

## Structural

### Adapter
Convert an interface you don't control into the one your code expects.

**Use when:** a vendor SDK's shape would otherwise spread through your domain. Your code
wants `Storage.save()`; S3 offers `putObject()`. The adapter is where that mismatch lives.

**This is the highest-value structural pattern in most systems.** It's what makes
"swap the provider" a real possibility rather than a slogan — and it's the concrete form of
the "port" in Hexagonal Architecture.

**Do it properly:** the adapter must translate *errors and types too*, not just method
names. An adapter that lets `S3ServiceException` escape hasn't adapted anything. Map vendor
errors to your own error taxonomy at that boundary.

**Cost:** you now own a translation layer, including for features the vendor adds later.

### Bridge
Separate an abstraction from its implementation so both can vary independently.

**Use when:** you have an M×N problem — 3 renderers × 4 platforms — and don't want 12
classes.

**Cost:** rarely the right first answer; often confused with Adapter. Difference: Adapter
retrofits an existing incompatible interface; Bridge is designed upfront to split two axes
of variation.

### Composite
Treat individual objects and groups of them uniformly, via a tree.

**Use when:** the domain is genuinely recursive — file systems, UI trees, nested rule
groups, org charts. `render()` should work on a leaf and a branch identically.

**Cost:** recursive traversal makes performance and cycle-safety your problem.

### Decorator
Wrap an object to add behavior without changing it.

**Use when:** cross-cutting concerns — logging, retry, metrics, caching, auth — need to
apply to many operations. Each concern becomes one wrapper, composable in any order.

```ts
const store = withLogging(withRetry(withMetrics(baseStore)));
```

```go
// The idiomatic Go form: a func taking and returning the same interface.
func WithRetry(next Store, attempts int) Store { /* ... */ }
```

**Order is semantics, not style.** Retry inside metrics counts one logical call; metrics
inside retry counts each attempt. Decide deliberately and comment it.

**Cost:** deep stacks make stack traces and debugging worse. Keep it to a few layers, and
give each wrapper a name that appears in traces.

### Facade
One simple entry point over a complicated subsystem.

**Use when:** callers must currently orchestrate a fixed sequence — `authenticate()`,
`validate()`, `authorize()`, `execute()`, `log()` — and getting the order wrong is a bug.
Collapse it to `request()`.

**Cost:** can become a god object that grows a method per use case. When the facade's
methods stop sharing a subsystem, split it.

### Flyweight
Share immutable intrinsic state across many instances.

**Use when:** genuinely millions of objects with heavy shared data — glyphs, tile maps,
interned strings. Profile before reaching for it.

**Cost:** splits state into intrinsic/extrinsic, which complicates every call. Almost
always premature.

### Proxy
Stand in for another object to control access to it.

**Use when:** you need lazy loading, access control, caching, or remote calls to look like
local ones. RPC client stubs and ORM lazy relations are proxies.

**Cost:** the illusion leaks. A property access that quietly triggers a network call
surprises people and hides latency. Make the expensive path visible where you can.

---

## Choosing between them

Common confusions, resolved:

- **Factory vs Builder** — Factory picks *which* type; Builder assembles *one* type
  incrementally. If both apply, the factory can return a builder.
- **Adapter vs Facade** — Adapter changes an interface to match an expectation; Facade
  simplifies an interface nobody wants to use directly. Adapter is about compatibility,
  Facade about complexity.
- **Decorator vs Proxy** — Both wrap. Decorator adds behavior around the same operation;
  Proxy controls whether and when the operation happens at all.
- **Decorator vs Inheritance** — If you'd need a subclass per combination of features,
  you want decorators. Combinatorial explosion is the tell.
