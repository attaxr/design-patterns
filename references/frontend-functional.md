# Frontend & Functional Patterns

Composition patterns for UI code, and the functional techniques that underpin them.

## Contents
- [Component patterns](#component-patterns) — Container/Presentational, Compound Components, Render Props, Hooks
- [State patterns](#state-patterns) — Provider, Flux/Redux, Signals, State Machine
- [Functional patterns](#functional-patterns) — pure functions, composition, HOFs, currying, monadic error handling
- [Choosing a state approach](#choosing-a-state-approach)

---

## Component patterns

### Container / Presentational
Split data fetching and state from rendering.

**Use when:** a component you want to test, reuse, or restyle is entangled with a data
source. The presentational half takes props and returns markup; the container half supplies
them.

**Modern framing:** the boundary is now usually a custom hook (state) plus a pure component
(rendering) rather than two components. In React Server Components the split maps onto
server (fetch) and client (interact), which makes the separation structural.

**Cost:** two units where there was one. Not worth it for a component used once with no
meaningful logic.

### Compound Components
Related components share implicit state through context, so consumers compose the structure.

```tsx
<Tabs defaultValue="a">
  <Tabs.List><Tabs.Trigger value="a" /></Tabs.List>
  <Tabs.Panel value="a" />
</Tabs>
```

**Use when:** building reusable UI primitives whose layout consumers need to control. Trying
to express every arrangement through props on a monolithic component produces a 20-prop API;
compound components let structure be expressed as structure.

**Cost:** the implicit context coupling means a misplaced child fails at runtime rather than
compile time. Guard with a context check that throws a clear error, and type the shared
context strictly.

### Render Props
Pass a function that renders, letting a component share behavior without owning presentation.

**Use when:** behavior must be shared and the consumer must control rendering — and hooks
can't do it, which is now rare. Still useful for components that need to control *where*
their children render, or for list virtualization APIs.

**Cost:** nesting gets deep quickly. Prefer hooks for logic reuse.

### Hooks
Extract stateful logic into reusable functions.

**Use when:** logic is reused, or a component's body has grown past comfortable reading.
`useScanStatus(id)` composes better than any HOC or render-prop equivalent and keeps types
intact.

**What to watch:**
- Dependency arrays are the main correctness risk — stale closures over changing values.
- A hook that returns ten things is doing too much; split by concern.
- Hooks compose but don't dedupe: two components calling the same fetching hook fetch twice
  unless a caching layer sits underneath.

**Cost:** near zero. This is the default form of logic reuse in React.

---

## State patterns

### Provider
Context supplies values to a subtree without prop drilling.

**Use when:** genuinely tree-wide concerns — theme, auth session, locale, a client instance.

**Cost:** any context value change re-renders all consumers. Split contexts by update
frequency: a rarely-changing session and a frequently-changing form state must not share a
provider. Also note that context is not a state manager — it's a distribution mechanism.

### Flux / Redux
Unidirectional data flow: dispatch action → reducer → new state → view.

**Use when:** many components mutate shared state and you need the mutations traceable —
complex editors, dashboards with cross-filtering, anything wanting time-travel debugging or
an audit of user intent.

**Cost:** substantial boilerplate relative to the alternatives, and a strong tendency to
absorb server state that belongs in a query cache.

**Note:** much of what Redux was used for is now better served by a server-state library
(TanStack Query, SWR) for remote data plus a small store (Zustand, Jotai) for genuine client
state. Reach for full Redux when the traceability itself is the requirement.

### Signals
Fine-grained reactivity — reading a signal subscribes precisely that computation.

**Use when:** the framework offers them (Solid, Vue, Svelte, Angular, Preact). Updates skip
the diffing step and touch only what depends on the changed value.

**Cost:** a second mental model alongside component rendering, and reactivity that can be
lost by destructuring or reading outside a tracked scope.

### State Machine
Explicit states and legal transitions for complex UI flows.

**Use when:** multi-step flows where invalid combinations are a real bug source — checkout,
onboarding wizards, upload with retry, forms with async validation. The tell is a component
holding several booleans (`isLoading`, `isError`, `isSuccess`) where some combinations are
nonsensical yet representable.

**The value:** replacing `isLoading && !isError` reasoning with one `status` that can only
hold valid values. A discriminated union plus a transition map gets you most of it without a
library.

See also State in `behavioral-concurrency.md`.

---

## Functional patterns

### Pure functions
Same input, same output, no side effects.

**Why it's foundational:** pure functions are trivially testable, safely memoizable, and
safe to move. Push side effects to the edges and keep the middle pure — this single habit
does more for testability than any structural pattern.

### Immutable objects
Never mutate; produce new values.

**Use when:** always, for shared state. Required for change detection in React and
equivalents, and eliminates a whole class of aliasing bugs.

**Cost:** allocation on every update. Rarely matters; when it does, use a structural-sharing
library rather than reverting to mutation.

### Composition
Build behavior by combining small functions.

```ts
const process = (input: Raw) => store(classify(analyze(parse(input))));
```

**Use when:** a transformation has distinct stages. Each stage is independently testable.

**Cost:** over-composition is real. A chain of eight one-line functions can be harder to read
than one clear function. Compose at meaningful boundaries.

### Higher-order functions
Functions taking or returning functions.

**Use when:** parameterizing behavior — this is how Strategy, Decorator, and Template Method
all look in idiomatic TS. `withRetry(fn)` returning a wrapped `fn` is the whole Decorator
pattern.

### Currying / partial application
Fix some arguments now, supply the rest later.

**Use when:** creating specialized versions of a general function —
`const logScan = log("scan")`. Also makes pipelines read cleanly.

**Cost:** heavily curried code is unfamiliar to most TS readers and gives worse error
messages. Use it where it clarifies, not as a house style.

### Monadic error handling (Result / Either)
Return errors as values instead of throwing.

**Use when:** errors are expected outcomes rather than exceptions — validation, parsing,
external calls whose failure is routine. `Result<T, E>` makes failure visible in the type,
so it can't be forgotten.

```ts
type Result<T, E = Error> =
  | { ok: true; value: T }
  | { ok: false; error: E };
```

**Cost:** it doesn't compose with `async/await` or `throw` without a helper layer, and mixing
both styles in one codebase is worse than either alone. Pick one convention per boundary and
document it — a common split is `Result` for domain and validation errors, exceptions for
programmer errors and unrecoverable faults.

**Functors, applicatives, monads** are the general vocabulary here (`map`, `flatMap`), but you
don't need the theory to use `Result`. Introduce the terminology only if the team already
speaks it.

---

## Choosing a state approach

Work down this list and stop at the first that fits — each step costs more than the last.

1. **Local state** (`useState`) — until it needs sharing. Most state stays here.
2. **Lift to the nearest common parent** — until prop drilling becomes noise.
3. **Server-state library** for anything fetched. This eliminates most "global state" needs,
   since much of it was cached server data all along.
4. **Context/Provider** for tree-wide, low-frequency values.
5. **A small store** (Zustand, Jotai, signals) for genuine cross-tree client state.
6. **Flux/Redux** when traceable mutation history is itself the requirement.
7. **State machine**, orthogonally, whenever a flow has invalid states that are currently
   representable.

The most common frontend architecture mistake is starting at 5 or 6 and putting server data
there.
