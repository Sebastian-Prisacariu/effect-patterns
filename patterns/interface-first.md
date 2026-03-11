# Interface-First Development with Effect

Build complete applications using only service contracts, then implement later.

## Overview

Interface-First Development (also known as Contract-First or Ports & Adapters) is a pattern where you:

1. **Define service contracts** — What operations exist and their types
2. **Build your entire application** — Wire everything together using only contracts
3. **Implement last** — Create "Live" layers with actual implementations

Effect's type system makes this incredibly powerful: TypeScript tracks all dependencies and errors at compile time, ensuring your application is correctly wired before you write a single line of implementation.

## Why Use This Pattern?

### The Problem with Implementation-First

Traditional development often looks like this:

```typescript
// ❌ Implementation-first: details leak everywhere
class UserService {
  constructor(private db: PostgresClient, private redis: RedisClient) {}
  
  async createUser(data: CreateUserInput) {
    // Implementation details baked in from day one
    const user = await this.db.query('INSERT INTO users...')
    await this.redis.set(`user:${user.id}`, JSON.stringify(user))
    await sendgrid.send({ to: user.email, template: 'welcome' })
    return user
  }
}
```

Problems:
- **Coupled to implementations** — Postgres, Redis, SendGrid are hardcoded
- **Hard to test** — Need real databases or complex mocking
- **Hard to refactor** — Changing email provider touches business logic
- **Design distracted by details** — You're thinking about SQL while designing flows

### The Interface-First Solution

```typescript
// ✅ Interface-first: pure contracts
class UserRepository extends Context.Tag("UserRepository")<
  UserRepository,
  {
    readonly create: (user: User) => Effect.Effect<User, DatabaseError>
    readonly findById: (id: string) => Effect.Effect<User, UserNotFound>
  }
>() {}

class UserCache extends Context.Tag("UserCache")<
  UserCache,
  {
    readonly set: (user: User) => Effect.Effect<void>
    readonly get: (id: string) => Effect.Effect<User, CacheMiss>
  }
>() {}

class EmailService extends Context.Tag("EmailService")<
  EmailService,
  {
    readonly sendWelcome: (user: User) => Effect.Effect<void, EmailError>
  }
>() {}
```

Now your business logic is pure:

```typescript
const createUser = (data: CreateUserInput) =>
  Effect.gen(function* () {
    const repo = yield* UserRepository
    const cache = yield* UserCache
    const email = yield* EmailService
    
    const user = User.fromInput(data)
    yield* repo.create(user)
    yield* cache.set(user)
    yield* email.sendWelcome(user)
    
    return user
  })

// Type: Effect<User, DatabaseError | EmailError, UserRepository | UserCache | EmailService>
```

**Zero implementation exists, but:**
- ✅ Complete flow is defined
- ✅ All errors are typed  
- ✅ All dependencies are tracked
- ✅ TypeScript verifies composition

## How It Works

### Step 1: Define Service Contracts

A contract is a `Context.Tag` with a service interface:

```typescript
import { Context, Effect } from "effect"

// Domain errors
class UserNotFound extends Data.TaggedError("UserNotFound")<{
  readonly id: string
}> {}

class DatabaseError extends Data.TaggedError("DatabaseError")<{
  readonly cause: unknown
}> {}

// Service contract
interface UserRepositoryService {
  readonly findById: (id: string) => Effect.Effect<User, UserNotFound>
  readonly findByEmail: (email: string) => Effect.Effect<User, UserNotFound>
  readonly create: (user: User) => Effect.Effect<User, DatabaseError>
  readonly update: (user: User) => Effect.Effect<User, DatabaseError>
  readonly delete: (id: string) => Effect.Effect<void, DatabaseError>
}

// The Tag — a type-level token
class UserRepository extends Context.Tag("UserRepository")<
  UserRepository,
  UserRepositoryService
>() {}
```

**Key points:**
- The interface defines **what**, not **how**
- Errors are explicit in the return type
- No implementation details (no mention of Postgres, SQL, etc.)

### Step 2: Build Application Logic

Use services by yielding their tags:

```typescript
// A use case that composes multiple services
const registerUser = (input: RegisterInput) =>
  Effect.gen(function* () {
    const users = yield* UserRepository
    const email = yield* EmailService
    const analytics = yield* AnalyticsService
    
    // Check if user exists
    const existing = yield* users.findByEmail(input.email).pipe(
      Effect.option  // Convert UserNotFound to None
    )
    
    if (Option.isSome(existing)) {
      return yield* Effect.fail(new EmailAlreadyRegistered({ email: input.email }))
    }
    
    // Create user
    const user = yield* users.create(User.fromRegisterInput(input))
    
    // Side effects (can run in parallel)
    yield* Effect.all([
      email.sendWelcome(user),
      analytics.track("user_registered", { userId: user.id })
    ], { concurrency: "unbounded" })
    
    return user
  })
```

The type system tracks everything:

```typescript
// TypeScript infers:
// Effect<
//   User,
//   DatabaseError | EmailError | AnalyticsError | EmailAlreadyRegistered,
//   UserRepository | EmailService | AnalyticsService
// >
```

### Step 3: Compose Use Cases

Build larger flows from smaller ones:

```typescript
const onboardUser = (input: OnboardInput) =>
  Effect.gen(function* () {
    const user = yield* registerUser(input)
    yield* createDefaultWorkspace(user.id)
    yield* assignFreePlan(user.id)
    yield* scheduleOnboardingEmails(user.id)
    return user
  })
```

Still no implementations — just composing contracts.

### Step 4: Create Implementations (Live Layers)

Only now do you write actual implementations:

```typescript
import { Layer } from "effect"

// PostgreSQL implementation
const UserRepositoryLive = Layer.succeed(UserRepository, {
  findById: (id) =>
    Effect.tryPromise({
      try: () => db.query("SELECT * FROM users WHERE id = $1", [id]),
      catch: (e) => new DatabaseError({ cause: e })
    }).pipe(
      Effect.flatMap((rows) =>
        rows[0]
          ? Effect.succeed(User.fromRow(rows[0]))
          : Effect.fail(new UserNotFound({ id }))
      )
    ),
    
  findByEmail: (email) =>
    Effect.tryPromise({
      try: () => db.query("SELECT * FROM users WHERE email = $1", [email]),
      catch: (e) => new DatabaseError({ cause: e })
    }).pipe(
      Effect.flatMap((rows) =>
        rows[0]
          ? Effect.succeed(User.fromRow(rows[0]))
          : Effect.fail(new UserNotFound({ id: email }))
      )
    ),
    
  create: (user) =>
    Effect.tryPromise({
      try: () => db.query(
        "INSERT INTO users (id, email, name) VALUES ($1, $2, $3) RETURNING *",
        [user.id, user.email, user.name]
      ),
      catch: (e) => new DatabaseError({ cause: e })
    }).pipe(Effect.map((rows) => User.fromRow(rows[0]))),
    
  update: (user) =>
    Effect.tryPromise({
      try: () => db.query(
        "UPDATE users SET email = $2, name = $3 WHERE id = $1 RETURNING *",
        [user.id, user.email, user.name]
      ),
      catch: (e) => new DatabaseError({ cause: e })
    }).pipe(Effect.map((rows) => User.fromRow(rows[0]))),
    
  delete: (id) =>
    Effect.tryPromise({
      try: () => db.query("DELETE FROM users WHERE id = $1", [id]),
      catch: (e) => new DatabaseError({ cause: e })
    }).pipe(Effect.asVoid)
})
```

### Step 5: Create Test Implementations

In-memory implementations for testing:

```typescript
const makeUserRepositoryTest = () => {
  const users = new Map<string, User>()
  
  return Layer.succeed(UserRepository, {
    findById: (id) =>
      Effect.sync(() => users.get(id)).pipe(
        Effect.flatMap((user) =>
          user
            ? Effect.succeed(user)
            : Effect.fail(new UserNotFound({ id }))
        )
      ),
      
    findByEmail: (email) =>
      Effect.sync(() => [...users.values()].find((u) => u.email === email)).pipe(
        Effect.flatMap((user) =>
          user
            ? Effect.succeed(user)
            : Effect.fail(new UserNotFound({ id: email }))
        )
      ),
      
    create: (user) =>
      Effect.sync(() => {
        users.set(user.id, user)
        return user
      }),
      
    update: (user) =>
      Effect.sync(() => {
        users.set(user.id, user)
        return user
      }),
      
    delete: (id) =>
      Effect.sync(() => {
        users.delete(id)
      })
  })
}
```

### Step 6: Wire It All Together

```typescript
// Production
const LiveLayer = Layer.mergeAll(
  UserRepositoryLive,
  EmailServiceLive,
  AnalyticsServiceLive
)

const program = registerUser(input).pipe(
  Effect.provide(LiveLayer)
)

await Effect.runPromise(program)

// Testing
const TestLayer = Layer.mergeAll(
  makeUserRepositoryTest(),
  EmailServiceTest,
  AnalyticsServiceTest
)

const testProgram = registerUser(input).pipe(
  Effect.provide(TestLayer)
)

await Effect.runPromise(testProgram)
```

## Project Structure

A typical interface-first project structure:

```
src/
├── domain/                     # Pure domain logic
│   ├── user/
│   │   ├── model.ts           # User type, validation
│   │   ├── errors.ts          # UserNotFound, etc.
│   │   └── service.ts         # Context.Tag only (no impl)
│   ├── email/
│   │   ├── errors.ts
│   │   └── service.ts         # Context.Tag only
│   └── analytics/
│       └── service.ts         # Context.Tag only
│
├── application/                # Use cases (compose services)
│   ├── register-user.ts
│   ├── onboard-user.ts
│   └── delete-account.ts
│
├── infrastructure/             # Implementations (Live layers)
│   ├── persistence/
│   │   ├── user-repository-postgres.ts
│   │   └── user-repository-memory.ts
│   ├── email/
│   │   ├── email-ses.ts
│   │   └── email-console.ts   # Dev: just logs
│   └── analytics/
│       ├── analytics-posthog.ts
│       └── analytics-noop.ts  # Dev: does nothing
│
├── layers/                     # Layer compositions
│   ├── live.ts                # Production layers
│   ├── dev.ts                 # Development layers
│   └── test.ts                # Test layers
│
└── main.ts                     # Entry point
```

## When to Use This Pattern

### ✅ Good Fit

- **Complex domains** — Many services with intricate interactions
- **Multiple environments** — Need different implementations for prod/dev/test
- **Team collaboration** — Multiple people can work on different layers
- **Long-lived projects** — Refactoring implementations won't break business logic
- **High test coverage** — Easy to test with in-memory implementations

### ❌ Not Ideal

- **Simple scripts** — Overhead not worth it for one-off scripts
- **Prototyping** — When you're still figuring out what to build
- **Single implementation** — If you'll only ever have one implementation, consider `Effect.Service` instead

## Context.Tag vs Effect.Service

| Aspect | Context.Tag | Effect.Service |
|--------|-------------|----------------|
| **Definition** | Interface + Tag separate from implementation | Interface + Tag + Default implementation together |
| **Flexibility** | Multiple implementations easy | One "blessed" implementation |
| **Use case** | Interface-first, ports & adapters | Quick services with obvious single implementation |
| **Testing** | Provide test layer | Override with `Layer.provide` |

**Use `Context.Tag` when:** You want interface-first design, multiple implementations, or maximum flexibility.

**Use `Effect.Service` when:** There's one obvious implementation and you want convenience.

## Common Patterns

### Accessor Functions

Create convenient accessor functions for cleaner code:

```typescript
class UserRepository extends Context.Tag("UserRepository")<
  UserRepository,
  UserRepositoryService
>() {
  // Static accessor methods
  static findById = (id: string) =>
    Effect.flatMap(UserRepository, (repo) => repo.findById(id))
    
  static create = (user: User) =>
    Effect.flatMap(UserRepository, (repo) => repo.create(user))
}

// Usage becomes cleaner:
const program = Effect.gen(function* () {
  const user = yield* UserRepository.findById("123")
  // vs: const repo = yield* UserRepository; const user = yield* repo.findById("123")
})
```

### Service with Dependencies

When a service implementation needs other services:

```typescript
// The contract (no dependencies visible)
class OrderService extends Context.Tag("OrderService")<
  OrderService,
  {
    readonly create: (input: CreateOrderInput) => Effect.Effect<Order, OrderError>
  }
>() {}

// Implementation that depends on other services
const OrderServiceLive = Layer.effect(
  OrderService,
  Effect.gen(function* () {
    const users = yield* UserRepository
    const inventory = yield* InventoryService
    const payments = yield* PaymentService
    
    return {
      create: (input) =>
        Effect.gen(function* () {
          const user = yield* users.findById(input.userId)
          yield* inventory.reserve(input.items)
          yield* payments.charge(user, input.total)
          // ... create order
        })
    }
  })
)

// The layer declares its dependencies
// OrderServiceLive: Layer<OrderService, never, UserRepository | InventoryService | PaymentService>
```

### Layered Testing

Test different layers independently:

```typescript
// Unit test: test use case with all mocks
const unitTestLayer = Layer.mergeAll(
  UserRepositoryTest,
  EmailServiceTest,
  AnalyticsServiceTest
)

// Integration test: real DB, mock external services
const integrationTestLayer = Layer.mergeAll(
  UserRepositoryLive,  // Real Postgres
  EmailServiceTest,    // Mock email
  AnalyticsServiceTest // Mock analytics
)

// E2E test: everything real
const e2eTestLayer = Layer.mergeAll(
  UserRepositoryLive,
  EmailServiceLive,
  AnalyticsServiceLive
)
```

## Learning Curve

This pattern has a steeper initial learning curve because:

1. **Thinking in contracts** — You must design interfaces before implementations
2. **Effect's type system** — Understanding `Effect<A, E, R>` and `Layer<A, E, R>`
3. **Deferred gratification** — You can't "just run it" until layers are wired

But the payoff:

1. **Faster iteration later** — Swap implementations without touching business logic
2. **Confident refactoring** — Types catch wiring errors
3. **Natural testability** — Test layers are trivial to create
4. **Clean architecture** — Boundaries are enforced by the type system

## Further Reading

- [Effect Documentation: Services](https://effect.website/docs/requirements-management/services/)
- [Effect Documentation: Layers](https://effect.website/docs/requirements-management/layers/)
- [hex-effect](https://github.com/jkonowitch/hex-effect) — Reference implementation of Hexagonal Architecture with Effect
- [Hexagonal Architecture](https://alistair.cockburn.us/hexagonal-architecture/) — Original pattern by Alistair Cockburn
