# Producer-Consumer

Separate the production of work from its processing using a queue.

## Pressure

Work arrives faster than it can be processed, or work must survive a restart. The rate of production and the rate of consumption are independent and unpredictable -- sometimes bursts arrive, sometimes the processor is slow. Without a queue, the producer must either block or drop work. With multiple consumers, distributing work without a queue requires complex coordination.

## Solution

Producers enqueue work items; consumers dequeue and process them independently. The queue decouples production rate from consumption rate. Multiple consumers can process in parallel.

## Implementation

### In-memory queue (single process)

```ts
class WorkQueue<T> {
  private queue: T[] = [];
  private readonly maxSize: number;

  constructor(maxSize: number = 1000) {
    this.maxSize = maxSize;
  }

  async enqueue(item: T): Promise<void> {
    if (this.queue.length >= this.maxSize) {
      throw new Error("queue full");
    }
    this.queue.push(item);
  }

  async dequeue(): Promise<T | undefined> {
    return this.queue.shift();
  }

  get length() { return this.queue.length; }
}

// Producer
const queue = new WorkQueue<Command>();
queue.enqueue({ kind: "runScan", scanId: "123", target: "example.com" });

// Consumer
while (true) {
  const cmd = await queue.dequeue();
  if (cmd) await handleCommand(cmd);
  else await sleep(100);
}
```

### Production-grade: use a real queue

For durability across restarts, use Postgres (SKIP LOCKED), Redis, RabbitMQ, Kafka, SQS, or NATS JetStream.

```ts
// Postgres as a queue (SKIP LOCKED -- avoids contention between workers)
async function dequeue(): Promise<Command | null> {
  const result = await db.transaction(async (tx) => {
    const [row] = await tx.sql`
      SELECT * FROM commands
      WHERE status = 'pending'
      ORDER BY created_at
      LIMIT 1
      FOR UPDATE SKIP LOCKED
    `;
    if (!row) return null;
    await tx.sql`UPDATE commands SET status = 'processing' WHERE id = ${row.id}`;
    return row;
  });
  return result ? JSON.parse(result.payload) : null;
}
```

### Go form (buffered channel + goroutines)

```go
func RunWorkers[T any](
    ctx context.Context,
    items <-chan T,
    workers int,
    process func(context.Context, T) error,
) {
    var wg sync.WaitGroup
    for i := 0; i < workers; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for item := range items {
                if err := process(ctx, item); err != nil {
                    log.Printf("process error: %v", err)
                }
            }
        }()
    }
    wg.Wait()
}
```

## Three decisions you must make

1. **Queue bounded or unbounded?** Unbounded is a memory leak with extra steps. Always set a max size. When the queue fills, decide: drop newest, drop oldest, or back-pressure the producer.

2. **At-least-once or at-most-once?** At-least-once needs idempotent consumers. At-most-once loses messages on crash.

3. **What happens on repeated failure?** After N retries, send to a Dead Letter Queue, not back to the main queue. A poison message should not block everything else.

## The tell

- You see periodic bursts of work that overwhelm the processing code
- Work needs to survive a process restart
- You want to scale processing independently of production
- You have multiple consumers that should share the workload

## Minimal cut

An in-memory queue (array + shift) is enough for single-process work. Production durability requires a real queue (Postgres, Redis, or message broker). Start with the simplest queue that meets your durability requirements.

## Cost

- Adds infrastructure (the queue) and operational complexity
- At-least-once semantics require idempotent consumers
- Debugging a chain of producers and consumers is harder than a synchronous call
- Monitoring queue depth, consumer lag, and processing latency becomes essential

## Related patterns

- **Command** -- Commands are the work items typically passed through a Producer-Consumer queue
- **Outbox** -- Outbox reliably publishes events from a DB; Producer-Consumer reliably processes them
- **Bulkhead** -- Bulkhead isolates per-dependency resource pools in the consumer
- **Dead Letter Queue** -- Failed items go to a DLQ instead of blocking the main queue
