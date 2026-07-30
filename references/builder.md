# Builder

Construct complex objects incrementally, especially when construction has ordering rules or many optional parameters.

## Pressure

Constructor calls with 6 or more parameters where most are optional, or construction has ordering rules (call `.where()` before `.orderBy()`, never after). The constructor signature is fragile -- adding a parameter shifts every call site.

## Solution

Two levels of solution depending on the pressure:

### Level 1: Options object (covers 90% of cases)

Replace positional parameters with a single options object. Stop here unless you have ordering rules.

```ts
// Before
new User("Alice", undefined, 30, true, undefined, "admin");

// Lighter alternative: options object
interface CreateUserOptions {
  name: string;
  age?: number;
  admin?: boolean;
  role?: string;
}

function createUser(opts: CreateUserOptions): User {
  return {
    name: opts.name,
    age: opts.age ?? 18,
    admin: opts.admin ?? false,
    role: opts.role ?? "user",
  };
}
```

### Level 2: Fluent builder (when ordering matters)

```ts
class QueryBuilder<T> {
  private table?: string;
  private where: string[] = [];
  private order?: "asc" | "desc";
  private limit?: number;

  from(table: string): this {
    this.table = table;
    return this;
  }

  where(condition: string): this {
    this.where.push(condition);
    return this;
  }

  orderBy(direction: "asc" | "desc"): this {
    this.order = direction;
    return this;
  }

  take(n: number): this {
    this.limit = n;
    return this;
  }

  build(): string {
    if (!this.table) throw new Error("table is required");
    let sql = `SELECT * FROM ${this.table}`;
    if (this.where.length) sql += ` WHERE ${this.where.join(" AND ")}`;
    if (this.order) sql += ` ORDER BY ${this.order}`;
    if (this.limit) sql += ` LIMIT ${this.limit}`;
    return sql;
  }
}

// Usage
const query = new QueryBuilder()
  .from("users")
  .where("age > 18")
  .orderBy("asc")
  .take(10)
  .build();
```

### Staged builder (type-level ordering enforcement)

```ts
class UserBuilder {
  static name(name: string): UserBuilderAge {
    return new UserBuilderAge(name);
  }
}

class UserBuilderAge {
  constructor(private name: string) {}
  age(a: number): UserBuilderOptional {
    return new UserBuilderOptional(this.name, a);
  }
}

class UserBuilderOptional {
  constructor(
    public name: string,
    public age: number,
  ) {}
  admin(): UserBuilderOptional { /* ... */ return this; }
  build(): User { /* ... */ }
}
// Usage: UserBuilder.name("Alice").age(30).admin().build()
// `.build()` without `.name()` is a compile error.
```

**The tell you need a staged builder:** calling `.build()` without a required field should be impossible at compile time, not just a runtime error.

### Go form

```go
type QueryBuilder struct {
    table  string
    wheres []string
    limit  int
}

func NewQueryBuilder() *QueryBuilder {
    return &QueryBuilder{}
}

func (b *QueryBuilder) From(table string) *QueryBuilder {
    b.table = table
    return b
}

func (b *QueryBuilder) Where(cond string) *QueryBuilder {
    b.wheres = append(b.wheres, cond)
    return b
}

func (b *QueryBuilder) Build() (string, error) {
    if b.table == "" {
        return "", errors.New("table required")
    }
    // build SQL string
}
```

## Minimal cut

Options object first. Fluent builder only when calls must be chained or validation spans multiple steps. Staged builder only when compile-time ordering guarantees are worth the type complexity.

## Cost

A fluent builder separates construction from the constructed object, so readers must look in two places to understand what is being built. The `.build()` method concentrates validation logic away from the call site.

## Common mistakes

- Using a builder when an options object would suffice
- Not validating required fields in `build()`, letting invalid objects exist
- The builder becoming a service locator by accepting too many dependencies

## Related patterns

- **Factory** -- Factory picks which type. Builder assembles one type incrementally.
