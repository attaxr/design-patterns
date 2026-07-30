# Dependency Injection

Pass dependencies in from the outside rather than creating them inside.

## Pressure

Dependencies are created inside functions or classes (`new Database()`, `new HttpClient()`), making testing impossible without real infrastructure and making it impossible to swap implementations without rewriting code. The class is coupled to both the decision of which implementation to use and the lifecycle of that dependency.

## Solution

Accept dependencies as constructor parameters or function arguments. The caller constructs and passes them. This is the single highest-leverage practice in all of software design -- it makes testing trivial, swapping implementations possible, and dependency graphs explicit.

## Implementation

```ts
// Before: tight coupling -- service creates its own dependencies
class OrderService {
  private db = new Database("prod");
  private email = new EmailService();

  async placeOrder(userId: string, items: Item[]) {
    const user = await this.db.findUser(userId);
    const order = await this.db.createOrder(userId, items);
    await this.email.sendConfirmation(user.email, order.id);
    return order;
  }
}
// Cannot test without a real database. Cannot swap email provider.
```

```ts
// After: dependencies are injected (constructor injection)
class OrderService {
  constructor(
    private readonly db: Database,
    private readonly email: EmailService,
  ) {}

  async placeOrder(userId: string, items: Item[]) {
    const user = await this.db.findUser(userId);
    const order = await this.db.createOrder(userId, items);
    await this.email.sendConfirmation(user.email, order.id);
    return order;
  }
}

// Usage
const db = new Database("prod");
const email = new EmailService();
const service = new OrderService(db, email);

// Test -- swap for in-memory implementations
const mockDb = new InMemoryDatabase();
const mockEmail = new MockEmailService();
const testService = new OrderService(mockDb, mockEmail);
```

### The three forms of DI

| Form | Best for |
|------|----------|
| **Constructor injection** | Required, stable dependencies. Dependency is available to all methods. |
| **Parameter injection** | Dependencies that vary per call, or stateless functions. |
| **Property injection** | Optional dependencies with sensible defaults. Rarely needed. |

```ts
// Parameter injection -- good for functions and stateless helpers
async function placeOrder(
  db: Database,
  email: EmailService,
  userId: string,
  items: Item[],
) { /* ... */ }

// Property injection -- for optional dependencies
class HttpClient {
  private retryPolicy: RetryPolicy = new DefaultRetryPolicy();

  setRetryPolicy(policy: RetryPolicy) {
    this.retryPolicy = policy;
  }
}
```

### DI Container -- when do you need one?

```ts
// Manual DI (no container) -- works fine for fewer than ~20 services
const config = Config.load();
const db = new Database(config.db);
const email = new EmailService(config.email);
const orderService = new OrderService(db, email);
const paymentService = new PaymentService(config.stripe, db);
const app = new App(orderService, paymentService);
app.start();
```

**You need a DI container when:**
- Creating the dependency graph manually becomes repetitive error-prone boilerplate
- Dependencies have complex lifecycles (scoped to request, singleton, transient)
- You have hundreds of services

**You do not need a container when:**
- You have fewer than approximately 20 services
- Dependencies have simple lifecycles (create once, use everywhere)
- You are already using a framework that handles wiring (Next.js, NestJS, etc.)

### Go form

```go
type OrderService struct {
    db    Database
    email EmailService
}

func NewOrderService(db Database, email EmailService) *OrderService {
    return &OrderService{db: db, email: email}
}

func main() {
    db := postgres.New(cfg)
    email := sendgrid.New(cfg)
    svc := NewOrderService(db, email)
    // ...
}
```

## Common mistakes

1. **Service locator anti-pattern** -- passing a container or factory around instead of the concrete dependency. A service locator hides dependencies the same way a global does.
2. **Over-injection** -- a class with 7 or more constructor parameters is doing too much. Split it into multiple classes.
3. **Container as a global** -- `container.get(Database)` is Singleton in disguise.
4. **Framework lock-in** -- if your DI container requires special decorators or magic imports across your whole codebase, you have coupled your code to the container.

## The litmus test

Can you construct any class in the system, in a test, with zero infrastructure running, by passing mocks or in-memory implementations through the constructor? If not, your DI is incomplete.

## Related patterns

- **Clean Architecture** -- Clean Architecture relies on DI to invert dependencies from domain to infrastructure
- **Repository** -- Repositories are typically injected into services
