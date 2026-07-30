# Saga

Manage a multi-service business operation by breaking it into local transactions with compensating actions for rollback.

## Pressure

A business operation spans multiple services that cannot share a database transaction. When one step fails, the previous steps' effects must be undone, but a distributed transaction (2PC) is too slow, unavailable, or unscalable. Without a Saga, a partial failure leaves the system in an inconsistent state with no way to recover.

## Solution

A Saga breaks a distributed operation into a sequence of local transactions, each with a compensating transaction that undoes it. If a step fails, the Saga runs the compensating transactions for every completed step in reverse order.

```
Forward flow:      Create Order -> Reserve Stock -> Charge Card -> Confirm Shipping
Compensation flow:                  Release Stock <- Refund Card <- (rollback from here)
```

## Implementation

### Orchestration Saga (preferred)

A coordinator tells each service what to do and collects results. Easier to understand, debug, and monitor.

```ts
// Each step has both execute and compensate
interface SagaStep<T = unknown> {
  name: string;
  execute(ctx: T): Promise<void>;
  compensate(ctx: T): Promise<void>;
}

class SagaExecutor {
  private completedSteps: SagaStep[] = [];
  private context: Record<string, unknown> = {};

  async run<T>(steps: SagaStep<T>[], initialCtx: T): Promise<void> {
    this.context = { ...initialCtx } as Record<string, unknown>;

    try {
      for (const step of steps) {
        await step.execute(this.context);
        this.completedSteps.push(step);
      }
    } catch (err) {
      console.error(
        `Saga failed at step "${steps[this.completedSteps.length]?.name}":`, err,
      );
      await this.compensate();
      throw new SagaError("Distributed transaction rolled back", err as Error);
    }
  }

  private async compensate(): Promise<void> {
    for (const step of this.completedSteps.reverse()) {
      try {
        await step.compensate(this.context);
      } catch (err) {
        console.error(`Compensation failed for step "${step.name}":`, err);
      }
    }
  }
}

// Usage
const saga = new SagaExecutor();
await saga.run([
  {
    name: "createOrder",
    async execute(ctx) { ctx.order = await orderService.create(ctx.userId, ctx.items); },
    async compensate(ctx) { await orderService.cancel(ctx.order.id); },
  },
  {
    name: "reserveInventory",
    async execute(ctx) { ctx.reservation = await inventoryService.reserve(ctx.order.items); },
    async compensate(ctx) { await inventoryService.release(ctx.reservation.id); },
  },
  {
    name: "chargePayment",
    async execute(ctx) { ctx.payment = await paymentService.charge(ctx.userId, ctx.order.total); },
    async compensate(ctx) { await paymentService.refund(ctx.payment.id); },
  },
], { userId, items });
```

### Go form

```go
type SagaStep struct {
    Name       string
    Execute    func(ctx map[string]interface{}) error
    Compensate func(ctx map[string]interface{}) error
}

type Saga struct {
    completed []SagaStep
}

func (s *Saga) Run(steps []SagaStep, ctx map[string]interface{}) error {
    for _, step := range steps {
        if err := step.Execute(ctx); err != nil {
            s.compensate(steps[:len(s.completed)])
            return fmt.Errorf("saga failed at %s: %w", step.Name, err)
        }
        s.completed = append(s.completed, step)
    }
    return nil
}

func (s *Saga) compensate(steps []SagaStep) {
    for i := len(steps) - 1; i >= 0; i-- {
        if err := steps[i].Compensate(ctx); err != nil {
            log.Printf("compensation failed for %s: %v", steps[i].Name, err)
        }
    }
}
```

### Two coordination styles

| Style | Pros | Cons |
|-------|------|------|
| **Orchestration** | Easy to understand, debug, and monitor. Coordinator tracks progress. | Coordinator is a single point of failure (deploy with redundancy). |
| **Choreography** | More decoupled. No central coordinator. | Harder to trace failures ("where is this order stuck?" requires reading every service's logs). |

**Rule of thumb:** Prefer orchestration unless you have a strong architectural reason for choreography. Being able to answer "where is this order right now?" is worth the coordinator.

## Non-reversible steps

Some actions cannot be compensated -- sending an email, shipping a physical item, recording a regulatory filing. For these:
1. Move them to the end of the Saga so they only run after all reversible steps succeed.
2. Make them the "pivot" step -- the step after which compensation is impossible.
3. Ensure idempotency -- a compensation that retries must not double-refund.

## Common mistakes

1. **Compensation can also fail.** A never-empty network is rare. Always handle compensation failures gracefully: log, alert, and schedule a manual remediation job.
2. **Forgetting idempotency in compensations.** `refund()` called twice may double-refund. Use idempotency keys for every Saga step and compensation.
3. **Saga timeout.** A Saga that waits indefinitely holds resources. Add a saga-level timeout that triggers compensation if the total duration exceeds a limit.
4. **Orchestrator as single point of failure.** The coordinator is a service like any other -- deploy it with redundancy and persistence. Restarting it should resume pending Sagas from a durable store.

## Related patterns

- **Command** -- Saga steps are often implemented as Commands
- **Outbox** -- Outbox ensures the Saga's events are reliably published
- **Circuit Breaker** -- Protect Saga steps from cascading failures
