# Strategy

Extract varying behavior into interchangeable implementations selected by the caller.

## Pressure

A `switch` or `if/else` on a type discriminator keeps growing, and the same switch appears in multiple functions. Adding a new variant means duplicating the branching logic across every function that handles it.

## Solution

Define each branch as a separate implementation of a common interface (or a function type for stateless strategies). Build a strategy map in one place. Call sites select the strategy by key and delegate.

## Implementation

### Function type strategy (TypeScript) -- preferred for stateless strategies

```ts
// Before: every function repeats the switch
function generate(prompt: string, kind: ProviderKind) {
  switch (kind) {
    case "openai" /* call OpenAI */:
    case "anthropic" /* call Anthropic */:
  }
}

function embed(text: string, kind: ProviderKind) {
  switch (kind) { /* same switch */ }
}

// After: extract the varying behavior into a strategy type
type GenerateFn = (prompt: string) => Promise<string>;
type EmbedFn = (text: string) => Promise<number[]>;

interface ProviderStrategies {
  generate: GenerateFn;
  embed: EmbedFn;
}

// Build the strategy map in one place
const strategies: Record<ProviderKind, ProviderStrategies> = {
  openai: { generate: openaiGenerate, embed: openaiEmbed },
  anthropic: { generate: anthropicGenerate, embed: anthropicEmbed },
};

// Usage -- no switch
const result = await strategies[kind].generate(prompt);
```

### Class strategy -- when strategies carry state

```ts
interface PaymentStrategy {
  pay(amount: number): Promise<PaymentResult>;
}

class StripeStrategy implements PaymentStrategy {
  constructor(private apiKey: string) {} // carries state
  async pay(amount: number) { /* ... */ }
}

class PayPalStrategy implements PaymentStrategy {
  constructor(private email: string) {}
  async pay(amount: number) { /* ... */ }
}

// Context holds the current strategy
class PaymentProcessor {
  constructor(private strategy: PaymentStrategy) {}
  async process(amount: number) {
    return this.strategy.pay(amount);
  }
}
```

### Go form

```go
// Function type strategy (stateless)
type GenerateFn func(ctx context.Context, prompt string) (string, error)

var strategies = map[ProviderKind]GenerateFn{
    ProviderOpenAI:    openaiGenerate,
    ProviderAnthropic: anthropicGenerate,
}

// Interface strategy (stateful)
type PaymentStrategy interface {
    Pay(ctx context.Context, amount float64) error
}

type StripeStrategy struct {
    apiKey string
}
func (s *StripeStrategy) Pay(ctx context.Context, amount float64) error { /* ... */ }
```

## How the LLM should add a new strategy

1. Create the new implementation (function or class)
2. Add one entry to the strategy map (or register it)
3. No call sites change

That is the value proposition: Open/Closed principle applied to one switch statement.

## When to use

- The same switch on a type discriminator appears in multiple functions
- Adding a new variant should not require editing every call site
- The set of variants is open (third parties may add new strategies)

## When not to use

- A single switch on a type discriminator in one function. One switch is fine.
- Two branches with no prospect of a third. Two branches can stay as an if/else.
- The variants are algorithmically simple and unlikely to diverge.

## Cost

The behavior is no longer visible at the call site. `strategies[kind].generate(prompt)` tells you what but not how. Naming matters more than usual -- use strategy names that describe behavior (`stripeStrategy`), not structure (`strategyA`).

## Related patterns

- **State** -- Same structure as Strategy. Strategy is chosen by the caller and does not change itself. State transitions itself based on events.
- **Command** -- Command is a request to do something later. Strategy is how to do something now.
