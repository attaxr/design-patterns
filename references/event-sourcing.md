# Event Sourcing

Store the sequence of events that led to the current state, rather than the current state itself.

## Pressure

History is part of the domain. You need an audit trail, temporal queries, or the ability to rebuild read models from past data. The question "how did this record get into this state?" is one the system must answer. Current state overwrites the past and cannot answer it.

## Solution

Append events to an immutable store. Current state is derived by replaying events. Every state change is recorded as a new event. The past is never erased, only corrected with compensating events.

```
Events stored:                    Current state (derived):
Deposited($20)  ---+
Withdrew($10)   ---+--> Account Balance: $100
Deposited($90)  ---+
```

## Implementation

```ts
// 1. Define events as a discriminated union
type AccountEvent =
  | { kind: "account_opened"; accountId: string; initialDeposit: number }
  | { kind: "deposited"; amount: number; timestamp: Date }
  | { kind: "withdrew"; amount: number; timestamp: Date };

// 2. Append-only event store
interface EventStore {
  append(
    streamId: string,
    events: AccountEvent[],
    expectedVersion: number,
  ): Promise<void>;
  readStream(streamId: string): Promise<AccountEvent[]>;
}

// 3. Aggregate -- rebuilds state from events
class Account {
  private balance = 0;
  private version = 0;
  private changes: AccountEvent[] = [];

  static loadFromHistory(events: AccountEvent[]): Account {
    const account = new Account();
    for (const event of events) account.apply(event);
    account.version = events.length;
    return account;
  }

  deposit(amount: number) {
    const event: AccountEvent = {
      kind: "deposited",
      amount,
      timestamp: new Date(),
    };
    this.apply(event);
    this.changes.push(event);
  }

  withdraw(amount: number) {
    if (this.balance < amount) throw new Error("insufficient funds");
    const event: AccountEvent = {
      kind: "withdrew",
      amount,
      timestamp: new Date(),
    };
    this.apply(event);
    this.changes.push(event);
  }

  private apply(event: AccountEvent) {
    switch (event.kind) {
      case "account_opened": this.balance = event.initialDeposit; break;
      case "deposited":      this.balance += event.amount; break;
      case "withdrew":       this.balance -= event.amount; break;
    }
  }

  async save(eventStore: EventStore) {
    if (this.changes.length === 0) return;
    await eventStore.append(`account:${this.id}`, this.changes, this.version);
    this.changes = [];
    this.version += this.changes.length;
  }
}
```

### Event versioning

Events are immutable. When the schema changes, you must handle old and new versions.

```ts
type AccountEventV1 = { kind: "deposited"; amount: number; timestamp: Date };

type AccountEventV2 = {
  kind: "deposited";
  amount: number;
  currency: string;
  timestamp: Date;
};

// Upcaster -- converts V1 to V2 on read
function upcast(event: AccountEventV1 | AccountEventV2): AccountEventV2 {
  if ("currency" in event) return event;
  return { ...event, currency: "USD" };
}
```

## The tell

You are asked "how did this record end up like this?" and the current answer is "check the logs." You need to replay historical data to rebuild a read model after a bug fix. Your audit requirements demand knowing what changed, when, and by whom, forever.

## Cost -- substantial

| Challenge | Impact |
|-----------|--------|
| Schema evolution | Events are immutable. Versioning and upcasting are permanent overhead. |
| Replay time | Grows with event volume. Requires snapshotting for large streams. |
| "Just fix the row" is impossible | Corrections are compensating events. |
| Learning curve | Every developer must understand the event model before making changes. |
| Querying current state | Events are not optimized for reads. Needs projections + read models (pair with CQRS). |

## The litmus test

Before adopting Event Sourcing, answer: "What concrete audit, temporal query, or projection requirement makes the event log essential?" If you cannot point to one, the cost probably exceeds the benefit.

## When not to use

- Current state is all you need and "how did we get here" is not a requirement
- The schema changes frequently and event versioning would be a constant tax
- The team has no experience with event-driven systems

## Related patterns

- **CQRS** -- Event Sourcing is almost always paired with CQRS. Events feed the write model; projections build read models.
- **Outbox** -- Outbox reliably publishes events from a relational database. Event Sourcing stores events as the source of truth.
- **Saga** -- Sagas can be implemented with events (choreography style) instead of commands.
