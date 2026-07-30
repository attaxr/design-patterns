# Command

Represent an action as plain data so it can be stored, queued, retried, scheduled, or undone.

## Pressure

Work must outlive the request that created it -- background jobs, scheduled tasks, retryable operations, undo stacks, audit logs of intent. An imperative function call cannot be stored, and calling it again is not the same as re-executing the intent.

## Solution

Represent the intent to do work as a plain data structure (not executable code). A dispatcher reads the data and invokes the corresponding handler. This separation turns an ephemeral function call into a durable, inspectable record.

## Implementation

### Core idea: commands as data

```ts
// 1. Define commands as a discriminated union of plain data types
type Command =
  | { kind: "runScan"; scanId: string; target: string }
  | { kind: "sendEmail"; emailId: string; to: string }
  | { kind: "deleteUser"; userId: string };

// 2. A handler dispatches by kind -- one function per command
async function handleCommand(cmd: Command): Promise<void> {
  switch (cmd.kind) {
    case "runScan":
      return runScan(cmd.scanId, cmd.target);
    case "sendEmail":
      return sendEmail(cmd.emailId, cmd.to);
    case "deleteUser":
      return deleteUser(cmd.userId);
  }
}
```

### Commands must be serializable

The point of Command is to cross process boundaries. A command that holds a closure, a DB handle, or a class instance cannot be serialized.

```ts
// Good: plain data, serializable
type Command = { kind: "runScan"; scanId: string; target: string };

// Bad: holds runtime state, cannot be queued
class RunScanCommand {
  constructor(
    private db: Database,
    private scanId: string,
  ) {}
}
```

### Command + queue integration

```ts
// Enqueue
async function enqueue(cmd: Command): Promise<void> {
  await db
    .insertInto("commands")
    .values({ payload: JSON.stringify(cmd) })
    .execute();
}

// Worker loop
async function processQueue() {
  while (true) {
    const row = await db
      .selectFrom("commands")
      .where("status", "=", "pending")
      .limit(1)
      .execute();
    if (!row) { await sleep(1000); continue; }
    try {
      await handleCommand(JSON.parse(row.payload) as Command);
      await db.updateTable("commands").set({ status: "done" }).where("id", "=", row.id).execute();
    } catch (err) {
      await db.updateTable("commands").set({ status: "failed", error: String(err) }).execute();
    }
  }
}
```

### Command for undo/redo

```ts
interface Command {
  execute(): Promise<void>;
  undo(): Promise<void>;
}
// Store the execution history so .undo() can reverse it.
```

### Go form

```go
type Command struct {
    Kind    string          `json:"kind"`
    Payload json.RawMessage `json:"payload"`
}

// Handler registry
var handlers = map[string]func(context.Context, json.RawMessage) error{
    "runScan":   handleRunScan,
    "sendEmail": handleSendEmail,
}

func dispatch(ctx context.Context, cmd Command) error {
    h, ok := handlers[cmd.Kind]
    if !ok {
        return fmt.Errorf("unknown command %q", cmd.Kind)
    }
    return h(ctx, cmd.Payload)
}
```

## The tell

You need Command when you find yourself saying "I need to run this later" or "I need to retry this if it fails" or "I need to know what happened." If the work is done synchronously and never retried, a function call is the right tool.

## Minimal cut

Do not create a command dispatcher framework. Start with a discriminated union and a switch in one file. Add queueing, retries, and persistence only when the command actually needs to survive a restart.

## Cost

- Commands and their schemas must be versioned -- a stale command in a queue can break a new handler
- The real execution logic is one more hop away from the call site
- Requires a persistence layer for durable queues

## Related patterns

- **Outbox** -- Outbox reliably publishes events; Command reliably executes work
- **Strategy** -- Command is a request to do something later. Strategy is how to do something now.
- **Saga** -- Saga composes multiple commands with compensating transactions
