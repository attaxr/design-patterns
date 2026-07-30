# Facade

Provide a simple, unified interface to a complex subsystem.

## Pressure

Callers must invoke a fixed sequence of operations across multiple subsystems, and getting the order wrong is a bug. The same 3-5 line sequence appears at multiple call sites. A subsystem has too many moving parts to use directly.

## Solution

Create a single class or function that wraps the subsystem interactions behind a simple operation. Callers interact with the facade instead of orchestrating the subsystems themselves.

## Implementation

```ts
// Before: every caller must orchestrate the full sequence
await authenticate(req);
const validated = await validate(req);
const authorized = await authorize(validated);
const result = await execute(authorized);
await log(result);

// After: one method hides the complexity
class ScanFacade {
  async handleRequest(req: Request): Promise<Result> {
    await this.auth.authenticate(req);
    const validated = await this.validator.validate(req);
    const authorized = await this.authorizer.authorize(validated);
    const result = await this.scanner.execute(authorized);
    await this.logger.log(result);
    return result;
  }
}

// Callers now have one line:
const result = await facade.handleRequest(req);
```

```go
// Go form
type ScanFacade struct {
    auth      Authenticator
    validator Validator
    authorizer Authorizer
    scanner   Scanner
    logger    Logger
}

func NewScanFacade(auth Authenticator, v Validator, authz Authorizer, s Scanner, l Logger) *ScanFacade {
    return &ScanFacade{auth: auth, validator: v, authorizer: authz, scanner: s, logger: l}
}

func (f *ScanFacade) HandleRequest(ctx context.Context, req Request) (Result, error) {
    if err := f.auth.Authenticate(ctx, req); err != nil {
        return Result{}, fmt.Errorf("auth: %w", err)
    }
    // ...
}
```

## The tell

The same multi-step sequence appears at multiple call sites. If it appears once, it is just a function. A facade is warranted when the sequence involves multiple distinct subsystems and you want to decouple callers from all of them.

## Cost

The facade can grow into a god object. If its methods stop sharing a common subsystem, split it into multiple facades. A facade that just delegates to a single subsystem is not providing value.

## Facade vs Adapter

| Adapter | Facade |
|---------|--------|
| Changes an interface to match an expectation | Simplifies an interface nobody wants to use directly |
| About compatibility | About complexity |
| Usually one-to-one (interface mapping) | Many-to-one (orchestration) |

## Minimal cut

Start with a function that calls the sequence. Promote it to a class when it needs its own state or when dependency injection makes testing easier.

## Related patterns

- **Adapter** -- Adapter converts, Facade simplifies
- **Mediator** -- Mediator coordinates communication between colleagues. Facade provides a simpler view of a subsystem.
