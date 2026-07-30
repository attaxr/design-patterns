# Circuit Breaker

Detect failures and prevent calls to a failing dependency, allowing it time to recover.

## Pressure

One flaky dependency stalls the entire system. A downstream service is slow or failing, and calling it repeatedly wastes resources, queuing up requests until the whole application runs out of threads or connections. Cascading failure from a slow downstream service propagates to your callers.

## Solution

Wrap calls to the dependency in a circuit breaker that tracks failures. When failures exceed a threshold, the circuit opens (stops calls) immediately. After a timeout, it allows limited test calls (half-open). If they succeed, the circuit closes. If they fail, it opens again.

```
Closed (normal) -> (failures > threshold) -> Open (fast-fail)
Open (fast-fail) -> (timeout elapsed) -> Half-Open (test)
Half-Open (success) -> Closed (normal)
Half-Open (failure) -> Open (fast-fail)
```

## Implementation

```ts
type CircuitState = "closed" | "open" | "half-open";

interface CircuitBreakerOptions {
  failureThreshold: number;   // failures before opening (e.g., 5)
  successThreshold: number;   // successes before closing from half-open (e.g., 3)
  openTimeoutMs: number;      // time before transitioning to half-open (e.g., 30000)
  halfOpenMaxCalls: number;   // max test calls in half-open state (e.g., 3)
}

class CircuitBreaker {
  private state: CircuitState = "closed";
  private failureCount = 0;
  private successCount = 0;
  private lastFailureTime = 0;
  private halfOpenCalls = 0;

  constructor(private options: CircuitBreakerOptions) {}

  async call<T>(fn: () => Promise<T>): Promise<T> {
    if (this.state === "open") {
      if (Date.now() - this.lastFailureTime >= this.options.openTimeoutMs) {
        this.state = "half-open";
        this.halfOpenCalls = 0;
      } else {
        throw new CircuitBreakerOpenError("Circuit breaker is open");
      }
    }

    if (this.state === "half-open" && this.halfOpenCalls >= this.options.halfOpenMaxCalls) {
      throw new CircuitBreakerOpenError("Circuit breaker is half-open and at capacity");
    }

    try {
      this.halfOpenCalls++;
      const result = await fn();
      this.onSuccess();
      return result;
    } catch (err) {
      this.onFailure();
      throw err;
    }
  }

  private onSuccess() {
    if (this.state === "half-open") {
      this.successCount++;
      if (this.successCount >= this.options.successThreshold) {
        this.reset();
      }
    }
    this.failureCount = 0; // consecutive failures only
  }

  private onFailure() {
    this.failureCount++;
    this.lastFailureTime = Date.now();
    if (this.failureCount >= this.options.failureThreshold) {
      this.transitionTo("open");
    }
  }

  private reset() {
    this.state = "closed";
    this.failureCount = 0;
    this.successCount = 0;
  }

  private transitionTo(state: CircuitState) {
    this.state = state;
    this.failureCount = 0;
    this.successCount = 0;
  }
}

// Usage
const breaker = new CircuitBreaker({
  failureThreshold: 5,
  successThreshold: 3,
  openTimeoutMs: 30000,
  halfOpenMaxCalls: 3,
});

async function fetchRecommendations(userId: string): Promise<Recommendation[]> {
  return breaker.call(async () => {
    const response = await fetch(`/api/recommendations/${userId}`);
    return response.json();
  });
}
```

### Go form

```go
type State int
const (
    Closed   State = 0
    Open     State = 1
    HalfOpen State = 2
)

type CircuitBreaker struct {
    mu               sync.Mutex
    state            State
    failureCount     int
    successCount     int
    lastFailureTime  time.Time
    failureThreshold int
    successThreshold int
    openTimeout      time.Duration
}

func (cb *CircuitBreaker) Call(fn func() (interface{}, error)) (interface{}, error) {
    cb.mu.Lock()
    if cb.state == Open && time.Since(cb.lastFailureTime) > cb.openTimeout {
        cb.state = HalfOpen
        cb.failureCount = 0
        cb.successCount = 0
    }
    if cb.state == Open {
        cb.mu.Unlock()
        return nil, fmt.Errorf("circuit breaker open")
    }
    cb.mu.Unlock()

    result, err := fn()
    cb.mu.Lock()
    defer cb.mu.Unlock()

    if err != nil {
        cb.failureCount++
        cb.lastFailureTime = time.Now()
        if cb.failureCount >= cb.failureThreshold {
            cb.state = Open
        }
        return nil, err
    }

    if cb.state == HalfOpen {
        cb.successCount++
        if cb.successCount >= cb.successThreshold {
            cb.state = Closed
            cb.failureCount = 0
        }
    } else {
        cb.failureCount = 0
    }
    return result, nil
}
```

### Fallback strategies

```ts
// When the circuit is open, provide a degraded response
try {
  return await breaker.call(() => fetchRecommendations(userId));
} catch (err) {
  if (err instanceof CircuitBreakerOpenError) {
    return getCachedRecommendations(userId); // degraded fallback
  }
  throw err;
}
```

### Integration with Retry

**Retry then circuit break.** The circuit breaker protects the retry wrapper, not the other way around:

```ts
const result = await breaker.call(async () => {
  return retry(() => fetchRecommendations(userId), { maxRetries: 3 });
});
```

If retry exhausts all attempts, the failure counts toward the circuit breaker's threshold. This is the correct order -- retry handles transient failures; circuit breaker handles persistent ones.

## Common mistakes

1. **No fallback** -- an open circuit that throws an error on every call is only half useful. Always provide a degraded fallback (cache, default, queue).
2. **Timeouts not configured** -- a slow call is as bad as a failing one. The circuit breaker needs a timeout to detect slow calls as failures.
3. **Permanent open** -- without the half-open recovery mechanism, once open, the circuit stays open forever.
4. **Single process state** -- circuit breaker state is in-process by default. For distributed systems, use a shared store (Redis) or accept that each instance tracks independently.

## Related patterns

- **Bulkhead** -- Bulkhead limits concurrent calls; Circuit Breaker stops calls entirely when failures accumulate. They pair naturally.
- **Retry** -- Retry handles transient failures; Circuit Breaker handles persistent ones. Use Retry inside Circuit Breaker.
- **Timeout** -- Timeout is the prerequisite. Circuit Breaker needs timeouts to classify slow calls as failures.
