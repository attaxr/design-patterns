# Bulkhead

Isolate resources so that failure or overload in one part of the system does not cascade to others.

## Pressure

One dependency's resource exhaustion starves other dependencies. A shared connection pool or thread pool means a burst of traffic to one service (e.g., payments) uses up all connections, causing unrelated operations (e.g., fetching product catalog) to queue or time out. Different dependencies have different reliability characteristics, but they share the same resource pool.

## Solution

Partition resources by dependency. Each dependency gets its own connection pool, thread pool, or queue. If one partition fills up, only calls to that dependency are affected.

```
Before (shared pool):          After (partitioned):
+-----------------------+      +-- Payments (10 conns) --+
|  Single Pool (50)     |      |  +-------------------+  |
|                       |      |  | Pool (10 max)     |  |
|  - payments           |      |  +-------------------+  |
|  - orders             |      +-------------------------+
|  - email              |      +-- Orders (20 conns) -----+
|                       |      |  +-------------------+  |
|  All compete for      |      |  | Pool (20 max)     |  |
|  the same 50 conns   |      |  +-------------------+  |
+-----------------------+      +-------------------------+
                               +-- Email (5 conns) ------+
                                  +-------------------+  |
                                  | Pool (5 max)      |  |
                                  +-------------------+  +
```

## Implementation

### Sempahore-based bulkhead (in-process)

```ts
interface BulkheadConfig {
  name: string;
  maxConcurrent: number;   // max in-flight calls (e.g., 10)
  maxQueueSize: number;    // max queued calls (e.g., 100)
  timeoutMs: number;       // max time to wait in queue (e.g., 30000)
}

class Bulkhead {
  private active = 0;
  private queue: Array<{
    resolve: (value: unknown) => void;
    reject: (reason: unknown) => void;
    addedAt: number;
  }> = [];
  private config: BulkheadConfig;

  constructor(config: BulkheadConfig) {
    this.config = config;
    this.startDrainTimer();
  }

  async call<T>(fn: () => Promise<T>): Promise<T> {
    if (this.active < this.config.maxConcurrent) {
      return this.execute(fn);
    }

    if (this.queue.length >= this.config.maxQueueSize) {
      throw new BulkheadFullError(`${this.config.name} queue is full`);
    }

    return new Promise((resolve, reject) => {
      this.queue.push({ resolve, reject, addedAt: Date.now() });
    });
  }

  private async execute<T>(fn: () => Promise<T>): Promise<T> {
    this.active++;
    try {
      return await fn();
    } finally {
      this.active--;
      this.drainQueue();
    }
  }

  private drainQueue() {
    while (this.active < this.config.maxConcurrent && this.queue.length > 0) {
      const next = this.queue.shift()!;
      if (Date.now() - next.addedAt > this.config.timeoutMs) {
        next.reject(new BulkheadTimeoutError(`${this.config.name} queue timeout`));
        continue;
      }
      this.execute(() => {
        next.resolve(undefined);
        return Promise.resolve(undefined);
      });
    }
  }

  private startDrainTimer() {
    setInterval(() => this.drainQueue(), 1000);
  }
}

// Usage -- one bulkhead per dependency
const bulkheads = {
  payments: new Bulkhead({ name: "payments", maxConcurrent: 10, maxQueueSize: 100, timeoutMs: 30000 }),
  orders:   new Bulkhead({ name: "orders",   maxConcurrent: 20, maxQueueSize: 200, timeoutMs: 30000 }),
  email:    new Bulkhead({ name: "email",    maxConcurrent: 5,  maxQueueSize: 50,  timeoutMs: 60000 }),
};

async function chargeUser(userId: string, amount: number) {
  return bulkheads.payments.call(() => paymentService.charge(userId, amount));
}
```

### Go form

```go
type Bulkhead struct {
    sem           chan struct{}
    queue         chan func()
    maxConcurrent int
    maxQueueSize  int
    timeout       time.Duration
}

func NewBulkhead(maxConcurrent, maxQueueSize int, timeout time.Duration) *Bulkhead {
    return &Bulkhead{
        sem:           make(chan struct{}, maxConcurrent),
        queue:         make(chan func(), maxQueueSize),
        maxConcurrent: maxConcurrent,
        maxQueueSize:  maxQueueSize,
        timeout:       timeout,
    }
}

func (b *Bulkhead) Execute(fn func()) error {
    select {
    case b.sem <- struct{}{}:
        go func() {
            defer func() { <-b.sem }()
            fn()
        }()
        return nil
    default:
        // Queue or reject
        select {
        case b.queue <- fn:
            return nil
        default:
            return fmt.Errorf("bulkhead queue full")
        }
    }
}
```

## Bulkhead types

| Type | What it isolates | When to use |
|------|-----------------|-------------|
| **Thread/connection pool** | Concurrent connections per dependency | Each dependency has distinct capacity needs |
| **Semaphore** | In-flight calls (no queue) | Must fail fast rather than queue |
| **Queue** | Work items per dependency | Work arrives in bursts, processing is slower |

## Bulkhead + Circuit Breaker: pairing

These two patterns pair naturally. Bulkhead limits concurrent access; Circuit Breaker stops calls when failures accumulate.

```ts
class ResilientDependency {
  private bulkhead: Bulkhead;
  private circuitBreaker: CircuitBreaker;

  constructor(config: BulkheadConfig & CircuitBreakerOptions) {
    this.bulkhead = new Bulkhead(config);
    this.circuitBreaker = new CircuitBreaker(config);
  }

  async call<T>(fn: () => Promise<T>): Promise<T> {
    return this.bulkhead.call(() => this.circuitBreaker.call(fn));
  }
}
```

## How to size bulkheads

- Start with per-dependency capacity: `maxConcurrent = connectionPoolSize / numDependencies`
- Monitor queue depth and timeouts. If queue backs up regularly, increase capacity.
- If queue never has more than 1 item, reduce capacity. The bulkhead is not constraining anything.
- Set `maxQueueSize` to bound memory -- unbounded queues are memory leaks with extra steps.

## Common mistakes

1. **Not sizing per dependency** -- a single bulkhead size for all dependencies ignores their different characteristics.
2. **Bulkhead without monitoring** -- you need to know when a bulkhead is rejecting or queuing. Add metrics.
3. **No timeout on queue** -- a call in the queue forever is indistinguishable from a hung call. Always timeout queued items.
4. **Thread pool bulkhead in single-threaded runtime** -- Node.js single-threaded event loop does not benefit from thread pool isolation. Use semaphore-based bulkhead instead.

## Related patterns

- **Circuit Breaker** -- Bulkhead limits concurrent calls; Circuit Breaker stops calls entirely when failures accumulate. They pair naturally.
- **Producer-Consumer** -- Bulkhead isolates resource pools. Producer-Consumer decouples work arrival from processing.
