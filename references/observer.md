# Observer (Pub/Sub)

Allow one-to-many notification where publishers and subscribers are unaware of each other.

## Pressure

A state change must notify subsystems that should not know about each other. Without this, you either poll (busy-loop checking for changes) or add direct coupling between unrelated modules. The set of subscribers changes independently of the publisher.

## Solution

Define a typed event system where publishers emit events and subscribers register handlers. The event bus mediates all communication -- publishers never know who is listening.

## Implementation

### Typed event bus -- the key is typed events

String-based event names are a runtime error hiding as a feature. Define all event types in one place.

```ts
// 1. Define all event types in one place
type Events = {
  "scan.completed": { scanId: string; findings: number };
  "scan.failed": { scanId: string; error: string };
  "user.created": { userId: string; email: string };
};

// 2. A typed pub/sub utility
class TypedEventBus {
  private listeners = new Map<string, Set<Function>>();

  on<K extends keyof Events>(
    event: K,
    handler: (payload: Events[K]) => void,
  ): void {
    if (!this.listeners.has(event as string)) {
      this.listeners.set(event as string, new Set());
    }
    this.listeners.get(event as string)!.add(handler);
  }

  emit<K extends keyof Events>(event: K, payload: Events[K]): void {
    for (const handler of this.listeners.get(event as string) ?? []) {
      handler(payload);
    }
  }

  off<K extends keyof Events>(
    event: K,
    handler: (payload: Events[K]) => void,
  ): void {
    this.listeners.get(event as string)?.delete(handler);
  }
}
```

### Synchronous vs Async -- decide explicitly

```ts
// Option A: sync (blocking, in-order, errors propagate)
emit(event, payload);
// handler runs immediately, any throw stops subsequent handlers

// Option B: async (fire-and-forget, concurrent, errors isolated)
for (const handler of listeners) {
  Promise.resolve(handler(payload)).catch((err) =>
    console.error("handler failed", err),
  );
}

// Option C: queued (async, ordered, survives restart)
// Emit writes to an outbox table; a background worker dispatches to handlers.
```

### Go form: channels

```go
type ScanEvent struct {
    ScanID   string
    Findings []Finding
}

type ScanSubscriber func(ScanEvent)

type EventBus struct {
    subscribers map[string][]ScanSubscriber
    mu          sync.RWMutex
}

func (b *EventBus) Subscribe(event string, sub ScanSubscriber) {
    b.mu.Lock()
    defer b.mu.Unlock()
    b.subscribers[event] = append(b.subscribers[event], sub)
}

func (b *EventBus) Publish(event string, payload ScanEvent) {
    b.mu.RLock()
    defer b.mu.RUnlock()
    for _, sub := range b.subscribers[event] {
        go sub(payload) // async dispatch; handle panics in subscriber
    }
}
```

## Common mistakes

1. **Memory leaks from unregistered listeners** -- always provide an `off()` or `unsubscribe()` method
2. **Stringly-typed events** -- a typo in an event name is a runtime error. Use typed event maps.
3. **Synchronous dispatch with slow handlers** -- one slow handler blocks all others. Default to async unless ordering guarantees are required.
4. **Error in one handler breaks others** -- isolate handler errors with try/catch per handler.

## Cost

- Asynchronous dispatch makes the execution order of handlers non-deterministic from the publisher's perspective
- Memory management of subscriptions is now the caller's responsibility
- Stack traces no longer show the full call chain (the publisher and handler are decoupled)

## Related patterns

- **Outbox** -- For reliable, durable event publishing that survives crashes
- **Mediator** -- Mediator knows about all participants. Observer broadcasts without knowing who is listening.
- **Producer-Consumer** -- Observer for notifications, Producer-Consumer for work distribution.
