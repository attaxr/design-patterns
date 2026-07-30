# Chain of Responsibility

Pass a request through a chain of handlers where each handler decides to process it or pass it to the next.

## Pressure

A request must pass through multiple processing stages (authentication, validation, authorization, handling), and you need to add or remove stages without changing the core logic. A fixed pipeline of operations where any handler can stop the chain.

## Solution

Define a handler type that accepts a request and optionally delegates to the next handler. Compose handlers into a pipeline. Each handler runs only if the previous handler passed control.

## Implementation

### Middleware pattern (most common form)

```ts
// 1. Define the handler type
type Handler = (req: Request) => Promise<Response>;

// 2. Define middleware as a function wrapping a handler
type Middleware = (next: Handler) => Handler;

// 3. Build the chain
function compose(middleware: Middleware[], handler: Handler): Handler {
  return middleware.reduceRight((next, mw) => mw(next), handler);
}

// Usage
const pipeline = compose(
  [withAuth, withValidation, withLogging],
  handleRequest,
);
```

```ts
// Concrete middleware example
function withAuth(next: Handler): Handler {
  return async (req) => {
    const token = req.headers.authorization;
    if (!token) return { status: 401, body: "unauthorized" };
    const user = await verifyToken(token);
    return next({ ...req, user }); // pass enriched request
  };
}
```

### Go form (standard http.Handler)

```go
type Middleware func(http.Handler) http.Handler

func WithAuth(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        token := r.Header.Get("Authorization")
        if token == "" {
            http.Error(w, "unauthorized", http.StatusUnauthorized)
            return
        }
        ctx := context.WithValue(r.Context(), "user", token)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}

// Compose
handler := WithAuth(WithValidation(WithLogging(handleRequest)))
```

### Stop-the-chain vs pass-through

```ts
// Stop -- handler returns without calling next
function withCache(next: Handler): Handler {
  return async (req) => {
    const cached = await cache.get(req.url);
    if (cached) return { status: 200, body: cached }; // short-circuit
    return next(req);
  };
}

// Pass-through -- handler calls next unconditionally
function withLogging(next: Handler): Handler {
  return async (req) => {
    console.time(req.url);
    const res = await next(req); // always delegates
    console.timeEnd(req.url);
    return res;
  };
}
```

## The tell

You have a sequence of operations where (a) any step can reject the request, (b) steps are added and removed independently, and (c) the sequence is the same for many request types.

## When to use

- HTTP middleware pipelines (auth, rate limiting, logging, CORS)
- Request validation or enrichment chains
- Permission/authorization gates

## When not to use

- The chain has 2 fixed steps that never change -- a simple function is clearer
- You need to fan-out to multiple handlers (use Observer for that)

## Cost

- The chain's correctness lives in its assembly order, not in any one handler
- Debugging "who swallowed my request" is the recurring pain
- Keep assembly in one obvious place and comment why each link sits where it does

## Note

If you use Express, Fastify, Chi, or Mux, you already have this. Use the framework's middleware system rather than writing your own.

## Related patterns

- **Decorator** -- Chain members may stop the request. Decorators always delegate.
- **Pipeline** -- Pipeline transforms data through every stage. Chain routes a request until someone handles it.
- **Observer** -- Observer fans-out to all subscribers. Chain picks one handler per link.
