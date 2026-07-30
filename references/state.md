# State

Make state transitions explicit and illegal states unrepresentable.

## Pressure

`if (status === "running")` checks appear across the codebase, and illegal state transitions are possible at runtime. The codebase has boolean flags (`isLoading`, `isError`, `isSuccess`) where some combinations are nonsensical yet representable. Adding a new state means editing conditionals in N places.

## Solution

Define all valid states as explicit types and all legal transitions in one table. Code either uses the transition table to validate moves or uses class-per-state to encapsulate state-specific behavior. Either way, the set of valid states and transitions is written down exactly once.

## Implementation

### Level 1: Transition table (covers 90% of cases)

```ts
// 1. Define states as a discriminated union
type ScanState =
  | { kind: "pending" }
  | { kind: "running"; startedAt: Date }
  | { kind: "completed"; findings: number }
  | { kind: "failed"; error: string };

// 2. Define legal transitions in one table
const transitions: Record<string, string[]> = {
  pending: ["running"],
  running: ["completed", "failed"],
  completed: ["archived"],
  failed: ["running"], // retry
};

function transition(state: ScanState, next: ScanState): ScanState {
  if (!transitions[state.kind]?.includes(next.kind)) {
    throw new Error(`Illegal transition: ${state.kind} -> ${next.kind}`);
  }
  return next;
}
```

### Level 2: Class-per-state (when each state has complex behavior)

```ts
interface ScanStateHandler {
  start(): Promise<ScanStateHandler>;
  cancel(): Promise<ScanStateHandler>;
  status(): string;
}

class PendingState implements ScanStateHandler {
  async start() { return new RunningState(/* ... */); }
  async cancel() { return new CancelledState(); }
  status() { return "pending"; }
}

class RunningState implements ScanStateHandler {
  async start() { throw new Error("already running"); }
  async cancel() { /* abort the scan */ return new CancelledState(); }
  status() { return "running"; }
}
```

**When to use the class approach:** when each state has complex behavior that differs significantly. For simple cases, the transition table + discriminated union is cleaner.

### Go form

```go
type State string

const (
    StatePending   State = "pending"
    StateRunning   State = "running"
    StateCompleted State = "completed"
    StateFailed    State = "failed"
)

var validTransitions = map[State][]State{
    StatePending:   {StateRunning},
    StateRunning:   {StateCompleted, StateFailed},
    StateCompleted: {},
    StateFailed:    {StateRunning},
}

func Transition(current, next State) (State, error) {
    for _, allowed := range validTransitions[current] {
        if allowed == next { return next, nil }
    }
    return current, fmt.Errorf("transition %s -> %s not allowed", current, next)
}
```

## The tell

You have booleans like `isLoading`, `isError`, `isSuccess` and you can construct invalid combinations (both `isLoading` and `isSuccess` true). Or you have `if (status === "running")` scattered across files. The first form tells you to use a discriminated union. The scattered conditionals tell you the current design has leaked state management across the codebase.

## Minimal cut

Most code does not need the class-per-state form. A transition table plus a `function transition(current, next)` that validates and returns the new state handles 90% of cases. Only escalate to the full State pattern when each state has multiple methods that behave differently.

## Cost

- The transition table must be kept in sync with the actual state machine behavior
- Class-per-state creates N classes where there was one, increasing file count
- Runtime enforcement of transitions (the table/class checks) can hide bugs if not tested

## Strategy vs State

Same structure. Strategy is chosen by the caller and does not change itself. State transitions itself based on events.

## Related patterns

- **Strategy** -- Same structure. Strategy selected externally, State transitions internally.
