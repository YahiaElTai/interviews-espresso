# Design Patterns & Principles

> **25 questions** — 15 theory, 10 practical

- Design patterns vs design principles: what each solves, how principles guide pattern selection, why patterns without principles lead to over-engineering
- SOLID principles: SRP, OCP, LSP, ISP, DIP — what each solves, violation examples, conflicts between them, and when rigid adherence hurts readability or adds unnecessary indirection
- DRY, KISS, YAGNI: when they conflict and how to decide which wins
- Singleton pattern: problems with testing, concurrency, hidden dependencies, DI alternatives
- Factory pattern: simple factory, Factory Method, when to use vs direct instantiation or object literals — and when builders are warranted
- Strategy and State patterns: identical structure, different intent, real-world signals
- Observer pattern: EventEmitter, DOM events, pub/sub, pitfalls of overuse
- Decorator and Adapter patterns: adding behavior vs changing interface, TypeScript decorators
- Chain of Responsibility: Express/NestJS middleware, request processing decoupling
- Inversion of Control and Dependency Injection: IoC containers, constructor vs property injection, service locator antipattern — pattern mechanics and testability benefits
- Repository pattern: separating business logic from data access, ORM abstraction tradeoffs
- Composition over inheritance: mixins, interfaces, when inheritance is still right
- Code smells: feature envy, shotgun surgery, primitive obsession, long parameter lists — recognizing smells, choosing the right refactoring, key techniques (extract method/class, move behavior, introduce value objects)
- Cost of abstraction: cognitive overhead, debugging difficulty, when abstractions hurt
---

## Foundational

<details>
<summary>1. What is the difference between a design pattern and a design principle — what problem does each solve, how do principles guide which patterns you select, and why does applying patterns without understanding the underlying principles lead to over-engineering?</summary>

**Design principles** are general guidelines about how to write good software — things like "depend on abstractions, not concretions" (DIP) or "prefer composition over inheritance." They tell you *what properties* good code should have but don't tell you *how* to achieve them in a specific situation.

**Design patterns** are reusable structural solutions to recurring problems — Strategy, Observer, Factory, etc. They tell you *how* to arrange classes and objects to solve a particular design challenge.

**How principles guide pattern selection:** When you face a problem, principles help you evaluate which pattern (if any) fits. For example, if you have a growing switch statement that violates OCP (you modify existing code to add new behavior), the Strategy pattern is a natural fix because it lets you add behavior via new classes without touching existing ones. The principle (OCP) diagnosed the problem; the pattern (Strategy) provided the solution.

**Why patterns without principles lead to over-engineering:** Without understanding *why* a pattern exists, developers apply them prophylactically — "we might need a Factory here someday" — creating indirection that serves no current purpose. A developer who understands DIP knows that injecting a concrete class is fine when there's only one implementation and no testing need for a mock. A developer who only knows patterns might create an interface, a factory, and a DI binding for a class that will never have a second implementation. The result is more code, more cognitive overhead, and no actual design benefit.

</details>

<details>
<summary>2. What are the five SOLID principles (SRP, OCP, LSP, ISP, DIP), what specific design problem does each one prevent, and how do they work together as a system — why does violating one principle often cause violations of the others?</summary>

**S — Single Responsibility Principle (SRP):** A class should have one reason to change. It prevents "god classes" that break when any of several unrelated requirements change. A `UserService` that handles authentication, profile updates, and email notifications has three reasons to change.

**O — Open/Closed Principle (OCP):** Software entities should be open for extension but closed for modification. It prevents the problem of modifying existing, tested code every time you add a feature. Instead of adding another `case` to a switch statement, you add a new class that implements an interface.

**L — Liskov Substitution Principle (LSP):** Subtypes must be substitutable for their base types without altering correctness. It prevents broken polymorphism — like a `Square` extending `Rectangle` where setting width unexpectedly changes height. If calling code can't safely use a subtype wherever the parent is expected, the hierarchy is wrong.

**I — Interface Segregation Principle (ISP):** Clients shouldn't depend on interfaces they don't use. It prevents "fat interfaces" that force implementers to stub out methods they don't need. Instead of one `Repository` interface with 20 methods, split into `ReadRepository` and `WriteRepository`.

**D — Dependency Inversion Principle (DIP):** High-level modules should depend on abstractions, not low-level modules. It prevents tight coupling — a service that directly imports a Postgres client can never use a different data store or be tested with an in-memory fake.

**How they work as a system:**

The principles reinforce each other. DIP makes OCP possible — you can extend behavior by injecting new implementations of an abstraction. ISP supports LSP — smaller, focused interfaces are easier to implement correctly without violating substitutability. SRP feeds into all of them — a class with a single responsibility naturally has a focused interface (ISP) and is easier to extend without modification (OCP).

**Why violations cascade:** A class that violates SRP (does too much) will likely have a fat interface (violates ISP), be hard to extend without modification (violates OCP), and depend directly on concrete implementations of its many concerns (violates DIP). Fix the SRP violation — split the class — and the other violations often resolve naturally.

</details>

<details>
<summary>3. Why does "composition over inheritance" exist as a design guideline — what problems does deep inheritance create that composition solves, how do mixins and interfaces provide composition mechanisms in TypeScript, and when is inheritance still the right choice?</summary>

**Problems with deep inheritance:**

- **Fragile base class problem:** Changing a parent class can break all descendants in unexpected ways. The more layers, the harder it is to reason about what a method call actually does.
- **Tight coupling:** Subclasses are bound to the parent's implementation details, not just its interface. You can't change the parent's internals without auditing every subclass.
- **Rigid hierarchy:** Inheritance models a single "is-a" axis. When a class needs behavior from two different hierarchies (e.g., `Loggable` and `Cacheable`), single inheritance can't express it, leading to awkward workarounds.
- **Shotgun surgery:** Adding a concern (like audit logging) to a class hierarchy often means modifying every level.

**How composition solves these:**

Instead of inheriting behavior, a class *holds references* to objects that provide behavior. Each concern is a separate, swappable component:

```typescript
interface Logger { log(msg: string): void; }
interface Cache { get(key: string): unknown; set(key: string, val: unknown): void; }

class UserService {
  constructor(
    private logger: Logger,
    private cache: Cache,
    private repo: UserRepository
  ) {}
}
```

Each dependency can be replaced, tested independently, and doesn't create a rigid hierarchy.

**TypeScript mixins** provide a way to compose behavior into classes:

```typescript
type Constructor<T = {}> = new (...args: any[]) => T;

function Timestamped<T extends Constructor>(Base: T) {
  return class extends Base {
    createdAt = new Date();
    updatedAt = new Date();
  };
}

// Multiple mixins can be composed: class User extends Timestamped(SoftDeletable(class {}))
class User extends Timestamped(class {}) {
  constructor(public name: string) { super(); }
}
```

**Interfaces** define capability contracts without implementation coupling — a class can implement multiple interfaces, composing its contract from several sources.

**When inheritance is still right:**

- True "is-a" relationships with a shallow hierarchy (1-2 levels) — `HttpError extends Error`.
- Framework requirements — NestJS guards, pipes, and interceptors extend base classes.
- When the base class provides substantial shared implementation that genuinely won't diverge across subclasses (e.g., `TypeORM`'s `BaseEntity`).

The heuristic: **favor composition by default; use inheritance when the relationship is genuinely hierarchical, shallow, and stable.**

</details>

## Conceptual Depth

<details>
<summary>4. When do SOLID principles conflict with each other — for example, when does following SRP push you to violate KISS by creating too many small classes, or when does strict OCP add unnecessary indirection? How do you decide which principle wins in practice, and what are the signs that rigid SOLID adherence is hurting your codebase more than helping it?</summary>

**Common conflicts:**

**SRP vs KISS:** Aggressively splitting classes by responsibility can create a constellation of tiny classes where understanding a single feature requires jumping through 8 files. A `CreateUserHandler` that delegates to `UserValidator`, `UserFactory`, `UserRepository`, `EventPublisher`, `AuditLogger`, and `ResponseMapper` might satisfy SRP, but the reader has to hold all those pieces in their head to understand "create a user."

**OCP vs YAGNI/KISS:** Making code "open for extension" often means adding interfaces, abstract classes, and injection points. If there's only ever one implementation and no realistic prospect of a second, the interface is pure ceremony — you've added indirection for an extension point nobody will use.

**DIP vs readability:** Depending on abstractions means you can't "go to definition" and see the real implementation — you land on an interface. In a small service with one database implementation, the interface between the service and the repository might just be a navigation obstacle.

**ISP vs SRP:** Splitting an interface into many small ones (ISP) can create a proliferation of single-method interfaces that are harder to discover and reason about than one well-organized interface.

**How to decide which principle wins:**

1. **Start with KISS and YAGNI.** Write the simplest thing that works. Only introduce abstractions when you have a concrete reason (testability, multiple implementations, a real extension point).
2. **Apply the "rule of three":** Don't abstract until you've seen the same pattern three times. Two instances might be coincidence.
3. **Measure by change frequency:** SRP matters most in code that changes often. A class that hasn't changed in a year doesn't need to be split, even if it technically has two responsibilities.
4. **Optimize for the reader:** The ultimate test is whether the code is easier to understand and change. If applying a principle makes the code harder to follow, the principle is being misapplied.

**Signs rigid SOLID is hurting you:**

- New developers can't trace a request through the system without a guide.
- Adding a simple feature requires touching 10+ files.
- Most interfaces have exactly one implementation and no tests use a mock.
- The team spends more time debating architecture than shipping features.
- "Where does this logic live?" is a common question with no obvious answer.

</details>

<details>
<summary>5. When do DRY, KISS, and YAGNI conflict with each other — what does it look like when removing duplication (DRY) creates a complex shared abstraction that violates KISS, or when building for extensibility violates YAGNI? How do you decide which principle takes priority in a given situation?</summary>

**DRY vs KISS — the "wrong abstraction" trap:**

Two pieces of code look similar, so you extract a shared function. Over time, each caller needs slightly different behavior, so you add parameters, flags, and conditional branches. The shared function becomes a complex mess that's harder to understand than the original duplication.

```typescript
// This started as DRY but now violates KISS
function processOrder(order: Order, opts: {
  skipValidation?: boolean;
  applyDiscount?: boolean;
  isRefund?: boolean;
  notifyCustomer?: boolean;
  legacyFormat?: boolean;
}) { /* 200 lines of branching logic */ }
```

The fix is often to *duplicate and diverge* — copy the shared code into each caller and let them evolve independently. Duplication is cheaper than the wrong abstraction.

**DRY vs YAGNI — premature generalization:**

You see duplication between two services, so you build a shared library to eliminate it. But the shared library now needs versioning, its own tests, release management, and both services are coupled to its release cycle. If the duplication was only in two places, the cost of the shared library exceeds the cost of the duplication.

**KISS vs extensibility (YAGNI):**

Building plugin systems, abstract factories, or event-driven architectures "for the future" when current requirements are simple. The extensibility adds cognitive load and debugging difficulty for features nobody has asked for.

**Decision framework:**

1. **Default to KISS.** The simplest solution that meets current requirements wins.
2. **Apply YAGNI aggressively.** Don't build for hypothetical futures. If the requirement doesn't exist today, the code to support it shouldn't either.
3. **Apply DRY selectively.** Eliminate duplication only when the duplicated code represents the *same concept* (same reason to change) — not just code that happens to look similar. Sandi Metz's rule: "duplication is far cheaper than the wrong abstraction."
4. **When DRY and KISS conflict, KISS wins.** Duplicated simple code is better than a shared complex abstraction.
5. **Apply the Rule of Three** (as covered in question 4) before abstracting duplicated code.

</details>

<details>
<summary>6. Why is the Singleton pattern considered problematic despite being one of the most well-known patterns — what specific problems does it cause for testing, concurrency, and hidden dependency management, and what alternatives (dependency injection, module-scoped instances) solve the same "single instance" need without Singleton's downsides?</summary>

Singleton persists because it's the most intuitive answer to "I need exactly one instance" — simple to understand and implement. But that simplicity masks real costs:

**Testing:** Singletons carry state across tests. One test modifies the singleton, and the next test inherits that state — causing flaky, order-dependent test suites. You can't easily replace a singleton with a mock because consumers access it via a global static method (`getInstance()`), not through an injectable reference.

**Hidden dependencies:** When a class calls `DatabasePool.getInstance()` inside its methods, the dependency is invisible from the constructor signature. You can't tell what a class depends on without reading its implementation. This makes refactoring dangerous and understanding call graphs harder.

**Tight coupling:** Every consumer is coupled to the concrete singleton class, not to an abstraction. Swapping implementations (e.g., switching from Redis to Memcached for caching) requires modifying every call site.

**Concurrency (in multi-threaded languages):** Classic double-checked locking is notoriously hard to get right. In Node.js this is less of a concern since it's single-threaded, but in languages like Java or Go, singletons introduce subtle race conditions.

**Alternatives:**

**Module-scoped instances (Node.js natural pattern):**

```typescript
// db-pool.ts
import { Pool } from 'pg';
const pool = new Pool({ connectionString: process.env.DATABASE_URL });
export { pool };
```

Node.js modules are cached after first `require`/`import`, so this naturally gives you a single instance without any Singleton pattern machinery. It's still a global, but it's explicit and importable.

**Dependency injection (the real solution):**

```typescript
class OrderService {
  constructor(private readonly db: Pool) {} // dependency is visible and injectable
}

// In composition root / DI container
const pool = new Pool({ connectionString: process.env.DATABASE_URL });
const orderService = new OrderService(pool);
```

DI solves all the singleton problems: the dependency is visible in the constructor, tests can inject a mock or test database, and the consumer depends on an interface, not a concrete class. The "single instance" guarantee moves to the composition root (or DI container scope), where it belongs.

In NestJS, registering a provider as the default scope (`SINGLETON`) gives you single-instance behavior managed by the DI container, with full testability via `overrideProvider()`.

</details>

<details>
<summary>7. What problem does the Factory pattern solve that direct instantiation and object literals don't — what is the difference between a simple factory and the Factory Method pattern, when does a Factory add real value vs unnecessary indirection, and at what point does object creation complexity warrant a Builder pattern instead?</summary>

**The problem:** When object creation involves logic — deciding which class to instantiate, providing default configuration, assembling dependencies — spreading that logic across every call site creates duplication and couples consumers to concrete classes.

**Simple factory** — a function or static method that encapsulates creation logic:

```typescript
function createNotifier(channel: 'email' | 'sms' | 'slack'): Notifier {
  switch (channel) {
    case 'email': return new EmailNotifier(smtpConfig);
    case 'sms': return new SmsNotifier(twilioClient);
    case 'slack': return new SlackNotifier(webhookUrl);
  }
}
```

The caller doesn't need to know which class to instantiate or what dependencies each needs. This is the most common and practical form in TypeScript.

**Factory Method pattern** — a method defined in a base class that subclasses override to create different products. Less common in TypeScript/Node.js because we favor composition over inheritance, but useful when a framework needs to let you customize object creation:

```typescript
abstract class ReportGenerator {
  generate() {
    const formatter = this.createFormatter(); // subclass decides which
    return formatter.format(this.getData());
  }
  protected abstract createFormatter(): Formatter;
}

class PdfReportGenerator extends ReportGenerator {
  protected createFormatter() { return new PdfFormatter(); }
}
```

**When a factory adds real value:**

- Creation involves choosing between multiple implementations based on runtime data.
- Object construction requires assembling dependencies the caller shouldn't know about.
- You need to enforce invariants during creation (validation, default values).

**When a factory is unnecessary indirection:**

- There's only one implementation and the constructor is straightforward.
- Object literals suffice — `{ name: 'John', role: 'admin' }` doesn't need a factory.
- The factory just calls `new` and passes arguments through without adding logic.

**When to reach for a Builder instead:**

When construction involves many optional parameters, specific ordering requirements, or step-by-step assembly:

```typescript
const query = new QueryBuilder()
  .select('name', 'email')
  .from('users')
  .where('active', true)
  .orderBy('createdAt', 'desc')
  .limit(10)
  .build();
```

The signal: if your factory function takes more than 4-5 parameters and many are optional, a builder provides a clearer, self-documenting creation API. If construction is a single decision ("which type?"), stick with a factory.

</details>

<details>
<summary>8. Why do the Strategy and State patterns have identical class structures but different intents — what is the core difference in what they model, what real-world signals tell you "this is a Strategy problem" vs "this is a State problem," and what goes wrong when you use the wrong one?</summary>

Both patterns have a context object that delegates to an interchangeable object behind an interface. The structure is identical — the difference is entirely in intent and lifecycle.

**Strategy models *algorithm selection*:** The client picks a strategy once (or per-request), and the strategy doesn't change itself. The strategy has no knowledge of other strategies.

- Sorting algorithm selection (quicksort vs mergesort)
- Shipping cost calculation (FedEx vs UPS vs DHL)
- Authentication method (JWT vs API key vs OAuth)

**State models *behavior that changes as internal state transitions*:** The state object itself triggers transitions to other states. Each state knows which states it can transition to.

- Order lifecycle (pending -> confirmed -> shipped -> delivered)
- Connection states (disconnected -> connecting -> connected -> error)
- Document workflow (draft -> review -> approved -> published)

**Real-world signals:**

| Signal | Strategy | State |
|--------|----------|-------|
| Who decides which implementation? | The caller/client | The current state object |
| Does the object change behavior over time? | No — set once per use | Yes — transitions happen |
| Do implementations know about each other? | No | Yes — each state knows valid transitions |
| Is there a lifecycle/workflow? | No | Yes |

**What goes wrong with the wrong choice:**

Using **Strategy when you need State**: You end up with transition logic scattered across the calling code — `if (currentStrategy === 'pending') setStrategy('confirmed')`. The calling code becomes a state machine implemented with if-statements, which is exactly what the State pattern is designed to encapsulate.

Using **State when you need Strategy**: You add unnecessary transition logic and state awareness to what should be simple, independent algorithm implementations. A shipping calculator doesn't need `FedExStrategy` to know about `UpsStrategy` — but if modeled as State, each implementation would need transition methods that make no semantic sense.

</details>

<details>
<summary>9. What problem does the Observer pattern solve, how do Node.js EventEmitter and DOM events implement it, and what are the pitfalls of overusing the Observer pattern — how does it create implicit coupling, make debugging harder, and lead to memory leaks when listeners aren't properly cleaned up?</summary>

**The problem it solves:** When one component changes and multiple other components need to react, hardcoding those reactions creates tight coupling. The Observer pattern lets subjects notify interested parties without knowing who they are or what they do.

**Node.js EventEmitter implementation:**

```typescript
import { EventEmitter } from 'events';

class OrderService extends EventEmitter {
  async placeOrder(order: Order) {
    const saved = await this.repo.save(order);
    this.emit('orderPlaced', saved); // subject doesn't know who's listening
  }
}

// Listeners registered elsewhere
orderService.on('orderPlaced', (order) => inventoryService.reserve(order));
orderService.on('orderPlaced', (order) => emailService.sendConfirmation(order));
orderService.on('orderPlaced', (order) => analyticsService.track('order', order));
```

**DOM events** follow the same pattern — `element.addEventListener('click', handler)` registers an observer; the browser (subject) fires the event without knowing what handlers exist.

**Pitfalls of overuse:**

**Implicit coupling (invisible dependencies):** The code that emits an event has no idea what listeners exist. Grep for `emit('orderPlaced')` and you find one line; grep for `on('orderPlaced')` and you find listeners scattered across the codebase. Understanding what happens when an order is placed requires tracing all listeners — there's no single place that shows the full flow.

**Debugging difficulty:** When something goes wrong after an event fires, the stack trace shows the listener's code, but the causal chain through the event system is invisible. Event ordering is also hard to reason about — if listener A and listener B both handle `orderPlaced`, their execution order depends on registration order, which might not be obvious.

**Memory leaks:** Listeners hold references to their enclosing scope. If you register a listener on a long-lived emitter from a short-lived object without removing it, the short-lived object can never be garbage collected:

```typescript
class WebSocketHandler {
  constructor(private emitter: EventEmitter) {
    // This listener keeps WebSocketHandler alive as long as emitter exists
    emitter.on('message', this.handleMessage.bind(this));
  }

  destroy() {
    // Must explicitly remove — or use AbortController / once()
    this.emitter.removeListener('message', this.handleMessage);
  }
}
```

Node.js warns when more than 10 listeners are added to a single event (likely a leak). Use `once()` for one-shot listeners and always clean up in teardown/dispose methods.

**Error swallowing:** If a listener throws and you don't handle errors on the emitter, the error can crash the process (`'error'` event) or silently break a listener chain.

The guideline: use Observer for genuine pub/sub decoupling (domain events, plugin systems). For direct procedural flows where the sequence matters, a simple function call is clearer.

</details>

<details>
<summary>10. What is the difference between the Decorator pattern and the Adapter pattern — why does one add behavior while the other changes an interface, how do TypeScript decorators relate to the GoF Decorator pattern, and when would you use each in a real codebase?</summary>

**Decorator** wraps an object to *add behavior* while keeping the same interface. The consumer doesn't know it's talking to a decorated version — the wrapper and the original implement the same interface.

```typescript
interface HttpClient {
  get(url: string): Promise<Response>;
}

class BaseHttpClient implements HttpClient {
  async get(url: string) { return fetch(url); }
}

class LoggingHttpClient implements HttpClient {
  constructor(private inner: HttpClient) {}
  async get(url: string) {
    console.log(`GET ${url}`);
    const response = await this.inner.get(url); // delegates to wrapped
    console.log(`Response: ${response.status}`);
    return response;
  }
}

class RetryHttpClient implements HttpClient {
  constructor(private inner: HttpClient, private retries = 3) {}
  async get(url: string) {
    for (let i = 0; i < this.retries; i++) {
      try { return await this.inner.get(url); }
      catch (e) { if (i === this.retries - 1) throw e; }
    }
    throw new Error('unreachable');
  }
}

// Compose decorators — order matters
const client = new RetryHttpClient(new LoggingHttpClient(new BaseHttpClient()));
```

**Adapter** wraps an object to *change its interface* so it matches what a consumer expects. It bridges incompatible interfaces.

```typescript
// Third-party logger with a different API
class WinstonLogger {
  info(meta: { message: string; level: string }) { /* ... */ }
}

// Your app expects this interface
interface AppLogger {
  log(message: string): void;
}

// Adapter bridges the gap
class WinstonAdapter implements AppLogger {
  constructor(private winston: WinstonLogger) {}
  log(message: string) {
    this.winston.info({ message, level: 'info' }); // translates interface
  }
}
```

**Key distinction:** Decorator *preserves* the interface and adds behavior. Adapter *translates* between interfaces without adding behavior.

**TypeScript decorators vs GoF Decorator:**

TypeScript's `@decorator` syntax (experimental decorators or the stage 3 proposal) is related but different. They're metaprogramming annotations that modify class declarations, methods, or properties at definition time — not runtime wrappers:

```typescript
function LogMethod(_target: any, key: string, descriptor: PropertyDescriptor) {
  const original = descriptor.value;
  descriptor.value = function (...args: any[]) {
    console.log(`Calling ${key}`);
    return original.apply(this, args);
  };
}

class UserService {
  @LogMethod
  findUser(id: string) { /* ... */ }
}
```

This is closer to the GoF Decorator in spirit (adding behavior without modifying the original) but operates at the class metadata level rather than through object composition. NestJS uses this extensively — `@Injectable()`, `@Controller()`, `@UseGuards()`.

**When to use each:**

- **Decorator**: Adding cross-cutting concerns (logging, caching, retry, metrics) to existing services without modifying them. Composable and stackable.
- **Adapter**: Integrating third-party libraries, legacy code, or external APIs that don't match your internal interfaces.

</details>

<details>
<summary>11. How does the Chain of Responsibility pattern decouple request processing — why do Express and NestJS use middleware chains instead of a single handler, what makes this pattern powerful for cross-cutting concerns, and when does a long chain become a debugging nightmare?</summary>

**The pattern:** A request passes through a chain of handlers. Each handler either processes the request and passes it along, short-circuits the chain (e.g., returns an error), or does both. No handler needs to know about the others in the chain.

**Why middleware chains instead of a single handler:**

A single handler that does authentication, rate limiting, logging, validation, body parsing, CORS, and business logic would be a massive function violating SRP. The chain lets you compose independent concerns:

```typescript
// Express middleware chain — each concern is isolated
app.use(cors());
app.use(helmet());
app.use(rateLimiter);
app.use(authenticate);
app.use(requestLogger);
app.post('/orders', validateOrderBody, createOrderHandler);
```

Each middleware has a single job. You can reorder them, add new ones, or remove them without touching the others. Different routes can use different combinations.

**Why it's powerful for cross-cutting concerns:**

- **Composability:** Mix and match middleware per route. Public routes skip `authenticate`; admin routes add `requireRole('admin')`.
- **Reusability:** The same `rateLimiter` works on every route without any route knowing about it.
- **Separation of concerns:** The `createOrderHandler` doesn't know or care about authentication — that happened upstream.
- **Short-circuiting:** An auth middleware can reject a request before it ever reaches the business logic, keeping handlers clean.

NestJS takes this further with its execution pipeline: Middleware -> Guards -> Interceptors (pre) -> Pipes -> Handler -> Interceptors (post) -> Exception Filters. Each layer handles a specific type of cross-cutting concern.

**When a long chain becomes a debugging nightmare:**

- **Invisible flow:** When a request passes through 15 middleware functions, tracing why a header is missing or a body is transformed requires stepping through each one. The chain is implicit — there's no single place that shows the full pipeline for a given route.
- **Order dependency:** Middleware order matters silently. Putting `bodyParser` after `validateBody` breaks validation. Putting `authenticate` after the route handler makes auth useless. These bugs are subtle because each middleware works correctly in isolation.
- **Error propagation:** An error thrown in middleware 7 might be caught by an error handler registered in middleware 2, or it might bubble up unhandled. The error path through a long chain is hard to reason about.
- **Performance opacity:** Each middleware adds latency. In a chain of 20, it's not obvious which ones are slow without explicit tracing.

**Mitigation:** Keep chains short (5-8 middleware max per route), use clear naming, add request tracing/correlation IDs, and document the expected middleware order. Group related middleware into named compositions rather than listing them individually.

</details>

<details>
<summary>12. What is Inversion of Control and why does it matter -- how does IoC differ from dependency injection (DI is one form of IoC), what role do IoC containers play (NestJS's DI container, for example), why does inverting control make systems more modular and extensible, and why is the Service Locator considered an antipattern despite also implementing IoC -- what problems does it cause compared to constructor injection?</summary>

**Inversion of Control (IoC)** is the principle that a component should not control how its dependencies are created or located — that control is *inverted* and given to something external. Instead of "I create what I need," it's "someone gives me what I need."

**IoC is the principle; DI is the most common implementation:**

- **Dependency Injection:** Dependencies are passed in (usually via constructor). The component declares what it needs; the caller provides it.
- **Template Method:** The framework calls your code (Hollywood Principle: "don't call us, we'll call you"). Express calling your route handler is IoC.
- **Service Locator:** The component asks a registry for its dependencies at runtime.

All three invert control. DI is preferred because it makes dependencies explicit.

**IoC containers** automate dependency wiring. In NestJS:

```typescript
@Injectable()
class OrderRepository {
  constructor(private readonly db: DatabaseClient) {}
}

@Injectable()
class OrderService {
  constructor(private readonly repo: OrderRepository) {} // NestJS resolves this
}

@Module({
  providers: [OrderService, OrderRepository, DatabaseClient],
})
class OrderModule {}
```

The container reads constructor signatures, resolves the dependency graph, and instantiates everything in the right order. You declare what each class needs; the container handles creation, lifecycle (singleton vs request-scoped), and teardown.

**Why IoC makes systems modular:**

- Components are decoupled from their dependencies' concrete types — swap implementations by changing the wiring, not the consumer.
- Testing becomes trivial — inject mocks or fakes through the same constructor.
- New features can be added by registering new implementations without modifying existing code (OCP).

**Service Locator — why it's an antipattern:**

```typescript
// Service Locator — component pulls its own dependencies
class OrderService {
  private repo: OrderRepository;

  constructor() {
    this.repo = ServiceLocator.get<OrderRepository>('OrderRepository'); // hidden dependency
  }
}
```

Problems compared to constructor injection:

- **Hidden dependencies:** The constructor signature doesn't reveal what the class needs. You must read the implementation to discover dependencies — same problem as the Singleton pattern (covered in question 6).
- **Runtime failures instead of compile-time errors:** If the locator doesn't have a registered `OrderRepository`, you get a runtime error. With constructor injection, a missing dependency is caught at startup (or at compile time with TypeScript's strict mode).
- **Testing difficulty:** Tests must configure a global service locator before constructing the class. With DI, you just pass mocks to the constructor.
- **Couples to the locator:** Every class depends on the `ServiceLocator` itself — a global dependency that's hard to eliminate.

**Constructor injection wins** because dependencies are visible, type-checked, and testable without any global state.

</details>

<details>
<summary>13. Why does the Repository pattern exist — what problems does it solve by putting an abstraction between business logic and data access, what are the tradeoffs of using a repository over calling an ORM directly, and when does the repository abstraction become a leaky abstraction that adds cost without value?</summary>

**The problem it solves:** When business logic directly calls an ORM or database client, the business layer is coupled to a specific data access technology. Changing the ORM, switching to a different database, or writing tests that don't hit a real database all require modifying business logic.

The Repository pattern provides a collection-like interface for accessing domain objects, hiding the data access implementation behind it:

```typescript
interface OrderRepository {
  findById(id: string): Promise<Order | null>;
  findByCustomer(customerId: string): Promise<Order[]>;
  save(order: Order): Promise<Order>;
  delete(id: string): Promise<void>;
}
```

The service layer depends on this interface. Whether the implementation uses Prisma, raw SQL, or an in-memory Map is irrelevant to the business logic.

**Tradeoffs — Repository vs calling ORM directly:**

| | Repository | Direct ORM calls |
|---|---|---|
| **Testability** | Inject a mock/fake implementation | Need to mock the ORM itself or use a test DB |
| **Swappability** | Change data source by swapping implementation | ORM calls scattered throughout business logic |
| **Complexity** | Extra layer of abstraction | Less code, fewer files |
| **Query flexibility** | Must expose every query the business layer needs | Full ORM API available everywhere |
| **Discoverability** | All data access for an entity in one place | Queries scattered across services |

**When the repository becomes a leaky abstraction:**

- **Exposing ORM types:** If repository methods return Prisma models or accept Prisma query options, the abstraction is leaking — consumers are still coupled to Prisma.
- **Mirroring the ORM API:** A repository with `findMany({ where: { status: 'active' }, include: { items: true } })` is just a pass-through to Prisma. It adds a layer without adding value.
- **Complex queries that don't fit the pattern:** When you need database-specific features (window functions, CTEs, full-text search), the repository either becomes a thin wrapper that obscures what's happening, or it exposes query-builder-like methods that defeat the abstraction.
- **Single implementation, no tests using mocks:** If there's only one implementation and tests use a real database anyway, the interface adds navigation overhead without payoff.

**When to use it:**

- Services with complex business logic that should be testable without a database.
- Multi-tenancy systems where the repository handles tenant scoping (so business logic can't forget it).
- When you genuinely anticipate changing the data store or need multiple implementations (e.g., cache-backed reads, write-through).

**When to skip it:** Simple CRUD services, prototypes, or services where the ORM *is* the repository (Prisma's generated client already provides a clean, typed data access API).

</details>

<details>
<summary>14. How do code smells connect to specific pattern solutions and refactoring techniques — for example, why does a growing switch statement signal the need for Strategy, why does feature envy suggest extract method/move behavior to the class it envies, and why does primitive obsession point toward extract class/value objects? What is the process for mapping a smell to the right refactoring, and when should you NOT refactor (the code works, is rarely changed, or the abstraction cost exceeds the benefit)?</summary>

**Smell-to-refactoring mapping:**

| Code Smell | What it signals | Refactoring / Pattern |
|---|---|---|
| **Growing switch/if-else on type** | New behavior requires modifying existing code (OCP violation) | **Strategy pattern** — each case becomes a class implementing a shared interface |
| **Feature envy** | A method uses more data from another class than its own | **Move Method** — relocate the method to the class whose data it uses |
| **Primitive obsession** | Raw strings/numbers used for domain concepts (email, money, ID) | **Extract Value Object** — `Email`, `Money`, `OrderId` classes with validation and behavior |
| **Long parameter list** | Too many arguments, often with booleans controlling behavior | **Introduce Parameter Object** or **Builder** — group related params into a typed object |
| **Shotgun surgery** | One change requires edits across many files | **Move Method / Extract Class** — consolidate the scattered logic into one place |
| **Divergent change** | One class changes for multiple unrelated reasons | **Extract Class** — split into classes with single responsibilities (SRP) |
| **Data clumps** | Same group of fields appears together repeatedly | **Extract Class** — the clump is a missing concept (e.g., `Address`, `DateRange`) |
| **Message chains** | `a.getB().getC().getD()` — deep navigation | **Hide Delegate** — expose a method on `a` that encapsulates the chain |

**The process for mapping a smell to the right refactoring:**

1. **Name the smell.** Identifying the specific smell narrows the solution space.
2. **Ask: what's the structural problem?** Feature envy = behavior in the wrong place. Switch on type = missing polymorphism. Primitive obsession = missing domain concept.
3. **Choose the simplest refactoring that addresses the structural problem.** Don't jump to a design pattern when Extract Method suffices.
4. **Verify with tests.** Refactoring should not change behavior — run tests before and after each step.

**When NOT to refactor:**

- **The code works and rarely changes.** A 3-year-old utility function with a switch statement that hasn't been modified in 18 months doesn't need Strategy. The cost of the refactoring exceeds the benefit.
- **You're about to delete or replace the code.** Don't polish code that's being thrown away.
- **The abstraction cost exceeds the smell cost.** Replacing a 5-case switch with 5 strategy classes, an interface, a factory, and a registry might be technically "cleaner" but practically worse — more files, more indirection, harder to understand for a simple problem.
- **No tests exist.** Refactoring without tests is risky. If the code lacks test coverage, write tests first — or accept the smell until you can.
- **It's exploratory/prototype code.** Premature structure in code that's still finding its shape slows you down. Let the design emerge, then refactor.

The rule: **refactor code that changes frequently and is hard to change. Leave stable code alone.**

</details>

<details>
<summary>15. What is the cost of abstraction — why do abstractions add cognitive overhead and debugging difficulty, how do you recognize when an abstraction is making code harder to understand rather than easier, and what principles help you decide when to abstract vs when to keep things concrete?</summary>

**Why abstractions have costs:**

- **Cognitive overhead:** Every abstraction layer is a level of indirection the reader must navigate. Clicking "Go to Definition" on an interface method takes you to the contract, not the implementation. You then need to figure out which concrete class is wired in at runtime. Multiply this across 5 layers and understanding a simple request flow requires holding a mental map of the entire DI graph.
- **Debugging difficulty:** Stack traces through abstract layers show interface calls and base class methods rather than the actual logic. Stepping through a debugger means stepping through wrapper after wrapper before reaching the real code. Event-driven abstractions (Observer, middleware chains) break linear stack traces entirely.
- **Indirection tax:** Every new interface, abstract class, or wrapper adds files, imports, and type definitions that must be maintained. Renaming a method means updating the interface, every implementation, and every mock.
- **Wrong abstraction lock-in:** Once code depends on an abstraction, changing the abstraction's shape is expensive — it's a shotgun surgery across all implementations and consumers. A premature abstraction can be harder to remove than the duplication it was meant to prevent.

**Signs an abstraction is hurting more than helping:**

- **Pass-through layers:** A method that just calls the same method on an inner object with the same arguments. The layer exists but adds nothing.
- **One implementation:** An interface with exactly one implementing class and no mocks in tests. The interface is ceremony, not abstraction.
- **Frequent "where does this actually happen?" questions:** If developers routinely struggle to find where the real work is done, the abstractions are obscuring rather than clarifying.
- **Abstraction naming is vague:** `AbstractBaseProcessorFactory` tells you nothing about what it does. Good abstractions have names that reveal intent (`OrderRepository`, `PaymentGateway`).
- **Configuration explosion:** When the abstraction requires extensive setup, configuration objects, or factory registration to use, the overhead may exceed the value.

**Principles for deciding when to abstract:**

1. **The Rule of Three** (as covered in question 4): wait for three concrete instances before abstracting.
2. **Abstract at the point of pain.** If you're struggling to test something, that's a signal for an interface. If you're duplicating code that represents the same concept across three services, that's a signal for extraction.
3. **Prefer concrete code until forced otherwise.** Direct function calls are easier to read, debug, and refactor than indirect ones. Start concrete; abstract when the concrete code creates a real problem.
4. **"Could I delete this layer and the code would still be clear?"** If yes, the layer is adding cost without value.
5. **Abstractions should reduce the total information a developer needs to hold in their head,** not increase it. If understanding the abstraction requires understanding all its implementations anyway, it's not abstracting anything.

</details>

## Practical — Implementation & Refactoring

<details>
<summary>16. Given a function with a growing switch statement or if-else chain that selects behavior based on a type (e.g., calculating shipping cost by carrier), refactor it to use the Strategy pattern in TypeScript — show the interface, concrete strategies, and the context class, and explain why this refactoring makes the code easier to extend without modifying existing logic (OCP)</summary>

**Before — the growing switch statement:**

```typescript
function calculateShippingCost(carrier: string, weight: number, distance: number): number {
  switch (carrier) {
    case 'fedex':
      return weight * 2.5 + distance * 0.01;
    case 'ups':
      return weight * 2.0 + distance * 0.015 + 3.0; // flat surcharge
    case 'dhl':
      const baseCost = weight * 1.8 + distance * 0.02;
      return distance > 1000 ? baseCost * 0.9 : baseCost; // long-distance discount
    // Every new carrier = modify this function
    default:
      throw new Error(`Unknown carrier: ${carrier}`);
  }
}
```

Every new carrier means editing this function — violating OCP. The function also becomes harder to test as each branch grows in complexity.

**After — Strategy pattern:**

```typescript
// 1. Strategy interface
interface ShippingStrategy {
  calculate(weight: number, distance: number): number;
}

// 2. Concrete strategies — each in its own file if complex
class FedExShipping implements ShippingStrategy {
  calculate(weight: number, distance: number): number {
    return weight * 2.5 + distance * 0.01;
  }
}

class UpsShipping implements ShippingStrategy {
  calculate(weight: number, distance: number): number {
    return weight * 2.0 + distance * 0.015 + 3.0;
  }
}

class DhlShipping implements ShippingStrategy {
  calculate(weight: number, distance: number): number {
    const baseCost = weight * 1.8 + distance * 0.02;
    return distance > 1000 ? baseCost * 0.9 : baseCost;
  }
}

// 3. Strategy registry — maps carrier names to strategies
const shippingStrategies = new Map<string, ShippingStrategy>([
  ['fedex', new FedExShipping()],
  ['ups', new UpsShipping()],
  ['dhl', new DhlShipping()],
]);

// 4. Context class that uses the strategy
class ShippingCalculator {
  constructor(private strategies: Map<string, ShippingStrategy>) {}

  calculate(carrier: string, weight: number, distance: number): number {
    const strategy = this.strategies.get(carrier);
    if (!strategy) throw new Error(`Unknown carrier: ${carrier}`);
    return strategy.calculate(weight, distance);
  }
}
```

**Why this satisfies OCP:**

Adding a new carrier (e.g., USPS) requires:
1. Creating a new class: `class UspsShipping implements ShippingStrategy { ... }`
2. Registering it: `shippingStrategies.set('usps', new UspsShipping())`

No existing code is modified. Each strategy is independently testable — you can unit test `DhlShipping.calculate()` without touching FedEx or UPS logic.

**When to keep the switch:** If there are only 2-3 simple cases and they're unlikely to grow, the switch is simpler and more readable. The Strategy refactoring pays off when you have 4+ cases, complex per-case logic, or a pattern of frequently adding new cases. As discussed in question 14, refactor at the point of pain, not in anticipation.

</details>

<details>
<summary>17. Implement the Decorator pattern in TypeScript to add logging, caching, and retry behavior to a service class without modifying the original — show both the classical wrapper approach and TypeScript's decorator syntax (using experimental decorators or the TC39 proposal), and explain when each approach is more appropriate</summary>

**Approach 1: Classical wrapper (GoF Decorator)**

The core idea (covered in question 10) is that the decorator and the original share an interface. Here we compose three decorators:

```typescript
interface UserService {
  findById(id: string): Promise<User | null>;
}

// Original implementation
class UserServiceImpl implements UserService {
  async findById(id: string): Promise<User | null> {
    return db.query('SELECT * FROM users WHERE id = $1', [id]);
  }
}

// Logging decorator
class LoggingUserService implements UserService {
  constructor(private inner: UserService, private logger: Logger) {}

  async findById(id: string): Promise<User | null> {
    this.logger.info(`findById called with id=${id}`);
    const result = await this.inner.findById(id);
    this.logger.info(`findById returned ${result ? 'user' : 'null'}`);
    return result;
  }
}

// Caching decorator
class CachingUserService implements UserService {
  private cache = new Map<string, { user: User | null; expiry: number }>();

  constructor(private inner: UserService, private ttlMs = 60_000) {}

  async findById(id: string): Promise<User | null> {
    const cached = this.cache.get(id);
    if (cached && cached.expiry > Date.now()) return cached.user;

    const user = await this.inner.findById(id);
    this.cache.set(id, { user, expiry: Date.now() + this.ttlMs });
    return user;
  }
}

// Retry decorator
class RetryingUserService implements UserService {
  constructor(private inner: UserService, private maxRetries = 3) {}

  async findById(id: string): Promise<User | null> {
    for (let attempt = 1; attempt <= this.maxRetries; attempt++) {
      try {
        return await this.inner.findById(id);
      } catch (err) {
        if (attempt === this.maxRetries) throw err;
        await new Promise((r) => setTimeout(r, 100 * attempt)); // backoff
      }
    }
    throw new Error('unreachable');
  }
}

// Compose — order matters: retry wraps cache wraps logging wraps real service
const userService: UserService = new RetryingUserService(
  new CachingUserService(
    new LoggingUserService(new UserServiceImpl(), logger)
  )
);
```

**Approach 2: TypeScript method decorators**

```typescript
// Retry decorator factory
function Retry(maxRetries = 3) {
  return function (_target: any, _key: string, descriptor: PropertyDescriptor) {
    const original = descriptor.value;
    descriptor.value = async function (...args: any[]) {
      for (let attempt = 1; attempt <= maxRetries; attempt++) {
        try {
          return await original.apply(this, args);
        } catch (err) {
          if (attempt === maxRetries) throw err;
          await new Promise((r) => setTimeout(r, 100 * attempt));
        }
      }
      throw new Error('unreachable');
    };
  };
}

// Cache decorator factory
function Cache(ttlMs = 60_000) {
  const cache = new Map<string, { value: any; expiry: number }>();
  return function (_target: any, _key: string, descriptor: PropertyDescriptor) {
    const original = descriptor.value;
    descriptor.value = async function (...args: any[]) {
      const key = JSON.stringify(args);
      const cached = cache.get(key);
      if (cached && cached.expiry > Date.now()) return cached.value;
      const value = await original.apply(this, args);
      cache.set(key, { value, expiry: Date.now() + ttlMs });
      return value;
    };
  };
}

class UserServiceImpl {
  @Retry(3)
  @Cache(60_000)
  async findById(id: string): Promise<User | null> {
    return db.query('SELECT * FROM users WHERE id = $1', [id]);
  }
}
```

**When to use each:**

| | Classical Wrapper | TS Decorator Syntax |
|---|---|---|
| **Best for** | Cross-cutting concerns that need to be composed at runtime, swapped in tests, or applied selectively per instance | Method-level concerns that apply uniformly to a class, especially in framework code |
| **Testability** | Excellent — each decorator is independently testable; compose with mocks | Harder — decorators are baked into the class at definition time |
| **Runtime flexibility** | Can compose different decorator stacks for different contexts | Fixed at class definition |
| **Framework integration** | Standard OOP, works everywhere | NestJS, TypeORM, and other decorator-heavy frameworks |
| **Verbosity** | More code (one class per decorator) | Less code, but magic is less visible |

Use classical wrappers when you need composability and testability. Use TS decorators for framework conventions and simple, universal cross-cutting concerns where runtime flexibility isn't needed.

</details>

<details>
<summary>18. Implement an Observer pattern using Node.js EventEmitter for a domain event system (e.g., order placed triggers inventory update, email notification, and analytics) — show the emitter setup, listener registration, and proper cleanup to avoid memory leaks, and explain what goes wrong when listeners accumulate without being removed</summary>

Building on the Observer concepts covered in question 9, here's a complete domain event system with proper lifecycle management:

```typescript
import { EventEmitter } from 'events';

// Type-safe event definitions
interface DomainEvents {
  orderPlaced: [order: Order];
  orderCancelled: [orderId: string, reason: string];
  paymentReceived: [payment: Payment];
}

// Typed event bus — wraps EventEmitter for type safety
class DomainEventBus {
  private emitter = new EventEmitter();

  constructor() {
    // Increase limit if you expect many listeners — but track them
    this.emitter.setMaxListeners(20);
  }

  emit<K extends keyof DomainEvents>(event: K, ...args: DomainEvents[K]) {
    this.emitter.emit(event, ...args);
  }

  on<K extends keyof DomainEvents>(event: K, listener: (...args: DomainEvents[K]) => void) {
    this.emitter.on(event, listener as any);
    return () => this.emitter.removeListener(event, listener as any); // return cleanup fn
  }

  once<K extends keyof DomainEvents>(event: K, listener: (...args: DomainEvents[K]) => void) {
    this.emitter.once(event, listener as any);
  }

  removeAllListeners(event?: keyof DomainEvents) {
    this.emitter.removeAllListeners(event);
  }
}

// Domain event bus — single instance managed by DI
const eventBus = new DomainEventBus();

// --- Listener registration (e.g., in module initialization) ---

class OrderEventListeners {
  private cleanupFns: Array<() => void> = [];

  constructor(
    private inventory: InventoryService,
    private email: EmailService,
    private analytics: AnalyticsService,
  ) {}

  register(eventBus: DomainEventBus) {
    // Store cleanup functions for each listener
    this.cleanupFns.push(
      eventBus.on('orderPlaced', async (order) => {
        await this.inventory.reserve(order.items);
      }),
      eventBus.on('orderPlaced', async (order) => {
        await this.email.sendOrderConfirmation(order.customerEmail, order);
      }),
      eventBus.on('orderPlaced', async (order) => {
        this.analytics.track('order_placed', {
          orderId: order.id,
          total: order.total,
        });
      }),
    );
  }

  // Critical: clean up on shutdown or when this module is destroyed
  destroy() {
    this.cleanupFns.forEach((cleanup) => cleanup());
    this.cleanupFns = [];
  }
}

// --- Emitting events ---

class OrderService {
  constructor(private repo: OrderRepository, private eventBus: DomainEventBus) {}

  async placeOrder(dto: CreateOrderDto): Promise<Order> {
    const order = await this.repo.save(dto);
    this.eventBus.emit('orderPlaced', order); // fire-and-forget
    return order;
  }
}
```

**What goes wrong when listeners accumulate:**

1. **Memory leaks:** Each `on()` call adds a reference. If `register()` is called on every incoming request (e.g., inside a request-scoped handler) without cleanup, listeners pile up. After 1000 requests, 3000 listener functions exist — each holding references to their closure scope (services, request data, etc.).

2. **Duplicate execution:** Those 3000 listeners all fire on the next `orderPlaced` event. One order triggers 1000 inventory reservations, 1000 emails, and 1000 analytics events.

3. **Node.js warning:** `MaxListenersExceededWarning: Possible EventEmitter memory leak detected. 11 orderPlaced listeners added.` This warning is your first signal — never suppress it with `setMaxListeners(Infinity)` without understanding why the count is growing.

**Prevention patterns:**

- Return cleanup functions from `on()` (as shown above) and call them in `destroy()` / `onModuleDestroy()`.
- Use `once()` for listeners that should fire only once.
- In NestJS, register listeners in module `onModuleInit()` and clean up in `onModuleDestroy()`.
- Use `AbortController` with `events.on(emitter, event, { signal })` for automatic cleanup when the signal aborts.

</details>

<details>
<summary>19. Implement a Repository pattern in TypeScript that wraps a database client (e.g., Prisma or a raw SQL client) — show the repository interface, a concrete implementation, and how a service class depends on the interface rather than the implementation. Demonstrate how this enables swapping the data source in tests without touching business logic</summary>

Building on the concepts from question 13:

```typescript
// --- Domain model (not a Prisma model — pure TypeScript) ---
interface Order {
  id: string;
  customerId: string;
  items: OrderItem[];
  total: number;
  status: 'pending' | 'confirmed' | 'shipped';
  createdAt: Date;
}

// --- Repository interface (the abstraction business logic depends on) ---
interface OrderRepository {
  findById(id: string): Promise<Order | null>;
  findByCustomer(customerId: string): Promise<Order[]>;
  save(order: Omit<Order, 'id' | 'createdAt'>): Promise<Order>;
  updateStatus(id: string, status: Order['status']): Promise<Order>;
}

// --- Concrete implementation using Prisma ---
class PrismaOrderRepository implements OrderRepository {
  constructor(private prisma: PrismaClient) {}

  async findById(id: string): Promise<Order | null> {
    const record = await this.prisma.order.findUnique({
      where: { id },
      include: { items: true },
    });
    return record ? this.toDomain(record) : null;
  }

  async findByCustomer(customerId: string): Promise<Order[]> {
    const records = await this.prisma.order.findMany({
      where: { customerId },
      include: { items: true },
      orderBy: { createdAt: 'desc' },
    });
    return records.map(this.toDomain);
  }

  async save(order: Omit<Order, 'id' | 'createdAt'>): Promise<Order> {
    const record = await this.prisma.order.create({
      data: {
        customerId: order.customerId,
        total: order.total,
        status: order.status,
        items: { create: order.items },
      },
      include: { items: true },
    });
    return this.toDomain(record);
  }

  async updateStatus(id: string, status: Order['status']): Promise<Order> {
    const record = await this.prisma.order.update({
      where: { id },
      data: { status },
      include: { items: true },
    });
    return this.toDomain(record);
  }

  // Map Prisma model to domain model — keeps Prisma types out of business logic
  private toDomain(record: any): Order {
    return {
      id: record.id,
      customerId: record.customerId,
      items: record.items,
      total: record.total.toNumber(), // Prisma Decimal -> number
      status: record.status,
      createdAt: record.createdAt,
    };
  }
}

// --- Service depends on the interface, not Prisma ---
class OrderService {
  constructor(private readonly orderRepo: OrderRepository) {}

  async confirmOrder(id: string): Promise<Order> {
    const order = await this.orderRepo.findById(id);
    if (!order) throw new Error(`Order ${id} not found`);
    if (order.status !== 'pending') throw new Error('Only pending orders can be confirmed');
    return this.orderRepo.updateStatus(id, 'confirmed');
  }
}
```

**Test with an in-memory fake — no database needed:**

```typescript
class InMemoryOrderRepository implements OrderRepository {
  private orders = new Map<string, Order>();

  async findById(id: string): Promise<Order | null> {
    return this.orders.get(id) ?? null;
  }

  async findByCustomer(customerId: string): Promise<Order[]> {
    return [...this.orders.values()].filter((o) => o.customerId === customerId);
  }

  async save(order: Omit<Order, 'id' | 'createdAt'>): Promise<Order> {
    const saved: Order = {
      ...order,
      id: crypto.randomUUID(),
      createdAt: new Date(),
    };
    this.orders.set(saved.id, saved);
    return saved;
  }

  async updateStatus(id: string, status: Order['status']): Promise<Order> {
    const order = this.orders.get(id);
    if (!order) throw new Error('Not found');
    const updated = { ...order, status };
    this.orders.set(id, updated);
    return updated;
  }

  // Test helper — seed data
  seed(order: Order) {
    this.orders.set(order.id, order);
  }
}

// Test — no database, no mocking library, fast and deterministic
describe('OrderService', () => {
  it('confirms a pending order', async () => {
    const repo = new InMemoryOrderRepository();
    repo.seed({ id: '1', customerId: 'c1', items: [], total: 100, status: 'pending', createdAt: new Date() });

    const service = new OrderService(repo);
    const result = await service.confirmOrder('1');

    expect(result.status).toBe('confirmed');
  });

  it('rejects confirming a shipped order', async () => {
    const repo = new InMemoryOrderRepository();
    repo.seed({ id: '1', customerId: 'c1', items: [], total: 100, status: 'shipped', createdAt: new Date() });

    const service = new OrderService(repo);
    await expect(service.confirmOrder('1')).rejects.toThrow('Only pending orders');
  });
});
```

The business logic in `OrderService` is tested without Prisma, without a database, and without `jest.mock()`. The `toDomain` mapping in `PrismaOrderRepository` keeps Prisma types from leaking into the domain — the service never sees a Prisma model.

</details>

<details>
<summary>20. Given a tightly coupled TypeScript class that directly instantiates its dependencies (e.g., a UserService that creates its own database connection and email client inside the constructor), refactor it to use Dependency Inversion — show the before and after code, extract interfaces, inject dependencies through the constructor, and demonstrate how the refactored version is testable with mock implementations</summary>

**Before — tightly coupled:**

```typescript
import { Pool } from 'pg';
import { SESClient, SendEmailCommand } from '@aws-sdk/client-ses';

class UserService {
  private db: Pool;
  private emailClient: SESClient;

  constructor() {
    // Creates its own dependencies — hidden, untestable, inflexible
    this.db = new Pool({ connectionString: process.env.DATABASE_URL });
    this.emailClient = new SESClient({ region: 'eu-west-1' });
  }

  async createUser(email: string, name: string): Promise<User> {
    // Direct SQL — coupled to Postgres
    const result = await this.db.query(
      'INSERT INTO users (email, name) VALUES ($1, $2) RETURNING *',
      [email, name]
    );
    const user = result.rows[0];

    // Direct AWS SDK call — coupled to SES
    await this.emailClient.send(new SendEmailCommand({
      Source: 'noreply@app.com',
      Destination: { ToAddresses: [email] },
      Message: {
        Subject: { Data: 'Welcome!' },
        Body: { Text: { Data: `Hi ${name}, welcome!` } },
      },
    }));

    return user;
  }
}
```

**Problems:** Can't test without a real Postgres database and AWS credentials. Can't swap email providers. Can't reuse the class with a different database. Dependencies are invisible from the outside.

**After — Dependency Inversion applied:**

```typescript
// Step 1: Extract interfaces for each dependency
interface UserRepository {
  create(email: string, name: string): Promise<User>;
}

interface EmailSender {
  sendWelcome(to: string, name: string): Promise<void>;
}

// Step 2: Concrete implementations (separate files)
class PgUserRepository implements UserRepository {
  constructor(private db: Pool) {}

  async create(email: string, name: string): Promise<User> {
    const result = await this.db.query(
      'INSERT INTO users (email, name) VALUES ($1, $2) RETURNING *',
      [email, name]
    );
    return result.rows[0];
  }
}

class SesEmailSender implements EmailSender {
  constructor(private client: SESClient) {}

  async sendWelcome(to: string, name: string): Promise<void> {
    await this.client.send(new SendEmailCommand({
      Source: 'noreply@app.com',
      Destination: { ToAddresses: [to] },
      Message: {
        Subject: { Data: 'Welcome!' },
        Body: { Text: { Data: `Hi ${name}, welcome!` } },
      },
    }));
  }
}

// Step 3: Service depends on abstractions, injected via constructor
class UserService {
  constructor(
    private readonly userRepo: UserRepository,
    private readonly emailSender: EmailSender,
  ) {} // dependencies are visible and injectable

  async createUser(email: string, name: string): Promise<User> {
    const user = await this.userRepo.create(email, name);
    await this.emailSender.sendWelcome(email, name);
    return user;
  }
}

// Step 4: Wiring in composition root (or DI container)
const db = new Pool({ connectionString: process.env.DATABASE_URL });
const sesClient = new SESClient({ region: 'eu-west-1' });

const userService = new UserService(
  new PgUserRepository(db),
  new SesEmailSender(sesClient),
);
```

**Testing — inject fakes, no real infrastructure:**

```typescript
describe('UserService', () => {
  it('creates a user and sends welcome email', async () => {
    const fakeRepo: UserRepository = {
      create: async (email, name) => ({ id: '1', email, name, createdAt: new Date() }),
    };
    const sentEmails: Array<{ to: string; name: string }> = [];
    const fakeEmail: EmailSender = {
      sendWelcome: async (to, name) => { sentEmails.push({ to, name }); },
    };

    const service = new UserService(fakeRepo, fakeEmail);
    const user = await service.createUser('john@example.com', 'John');

    expect(user.email).toBe('john@example.com');
    expect(sentEmails).toEqual([{ to: 'john@example.com', name: 'John' }]);
  });

  it('does not send email if user creation fails', async () => {
    const failingRepo: UserRepository = {
      create: async () => { throw new Error('duplicate email'); },
    };
    const fakeEmail: EmailSender = {
      sendWelcome: async () => { throw new Error('should not be called'); },
    };

    const service = new UserService(failingRepo, fakeEmail);
    await expect(service.createUser('dup@example.com', 'Dup')).rejects.toThrow('duplicate email');
  });
});
```

**What changed:** Dependencies moved from hidden internal creation to explicit constructor parameters. The service depends on interfaces, not concrete classes. Tests inject simple objects that satisfy the interface — no mocking libraries, no database, no AWS credentials. Swapping from SES to SendGrid means creating a new `SendGridEmailSender` class and changing the wiring — `UserService` is untouched.

</details>

## Practical — Smell Detection & Refactoring

<details>
<summary>21. Given a TypeScript class that exhibits multiple code smells (feature envy — methods that mostly use data from another class; primitive obsession — using raw strings for emails, currencies, or IDs; and a long parameter list in a constructor), identify each smell, explain what structural problem it indicates, and show the step-by-step refactoring for each — creating value objects, moving behavior to the right class, and introducing a parameter object or builder</summary>

**The smelly class:**

```typescript
class OrderProcessor {
  constructor(
    private customerEmail: string,        // primitive obsession
    private customerName: string,         // primitive obsession
    private customerTier: string,         // primitive obsession
    private itemPrices: number[],         // long parameter list...
    private currency: string,             // primitive obsession
    private taxRate: number,
    private discountCode: string | null,
    private shippingAddress: string,      // primitive obsession
  ) {}

  // Feature envy — this method mostly uses Customer data
  getCustomerDisplayName(): string {
    if (this.customerTier === 'premium') {
      return `★ ${this.customerName}`;
    }
    return this.customerName;
  }

  // Feature envy — pricing logic that belongs elsewhere
  calculateTotal(): number {
    const subtotal = this.itemPrices.reduce((sum, p) => sum + p, 0);
    const discount = this.discountCode === 'SAVE10' ? 0.1 :
                     this.discountCode === 'SAVE20' ? 0.2 : 0;
    const afterDiscount = subtotal * (1 - discount);
    return afterDiscount * (1 + this.taxRate);
  }

  // Primitive obsession — manual email validation
  sendConfirmation(): void {
    if (!this.customerEmail.includes('@')) throw new Error('Invalid email');
    // ... send email
  }
}
```

**Smell 1: Primitive obsession** -- raw strings for domain concepts.

`customerEmail`, `currency`, `customerTier`, and `shippingAddress` are all raw strings with implicit rules. Validation is scattered, and nothing prevents passing a currency where an email is expected.

**Refactoring -- extract value objects:**

```typescript
class Email {
  readonly value: string;
  constructor(value: string) {
    if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(value)) {
      throw new Error(`Invalid email: ${value}`);
    }
    this.value = value.toLowerCase();
  }
}

// Note: production Money types should use integer cents or a library like dinero.js
// to avoid floating point issues (e.g., 0.1 + 0.2 !== 0.3)
class Money {
  constructor(
    readonly amount: number,
    readonly currency: 'USD' | 'EUR' | 'GBP',
  ) {
    if (amount < 0) throw new Error('Amount cannot be negative');
  }

  add(other: Money): Money {
    if (this.currency !== other.currency) throw new Error('Currency mismatch');
    return new Money(this.amount + other.amount, this.currency);
  }

  multiply(factor: number): Money {
    return new Money(this.amount * factor, this.currency);
  }
}

type CustomerTier = 'standard' | 'premium' | 'enterprise';
```

Now the type system prevents mixing up strings, and validation happens at construction -- not scattered across methods.

**Smell 2: Feature envy** -- methods using another class's data more than their own.

`getCustomerDisplayName()` only uses customer fields. `calculateTotal()` is really pricing logic. These methods belong on the objects they're envious of.

**Refactoring -- move behavior to the right class:**

```typescript
class Customer {
  constructor(
    readonly email: Email,
    readonly name: string,
    readonly tier: CustomerTier,
  ) {}

  // Display logic belongs here — it uses Customer data
  get displayName(): string {
    return this.tier === 'premium' ? `★ ${this.name}` : this.name;
  }
}

class PricingCalculator {
  constructor(private readonly taxRate: number) {}

  calculate(items: Money[], discountCode: string | null): Money {
    const subtotal = items.reduce(
      (sum, item) => sum.add(item),
      new Money(0, items[0].currency),
    );
    const discountRate = this.resolveDiscount(discountCode);
    return subtotal.multiply(1 - discountRate).multiply(1 + this.taxRate);
  }

  private resolveDiscount(code: string | null): number {
    const discounts: Record<string, number> = { SAVE10: 0.1, SAVE20: 0.2 };
    return code ? (discounts[code] ?? 0) : 0;
  }
}
```

**Smell 3: Long parameter list** -- too many constructor args.

Eight constructor parameters is a sign that the class holds too many unrelated concepts. After extracting value objects and moving behavior, we can group remaining params into a parameter object.

**Refactoring -- introduce parameter object + simplified class:**

```typescript
interface OrderInput {
  customer: Customer;
  items: Money[];
  discountCode: string | null;
  shippingAddress: Address; // another value object
}

class OrderProcessor {
  constructor(
    private readonly pricing: PricingCalculator,
  ) {}

  process(input: OrderInput): ProcessedOrder {
    const total = this.pricing.calculate(input.items, input.discountCode);
    return {
      customerDisplay: input.customer.displayName,
      total,
      shippingAddress: input.shippingAddress,
    };
  }
}
```

**Summary of what changed:**

| Smell | Problem | Refactoring |
|---|---|---|
| Primitive obsession | No type safety, scattered validation | Extract value objects (`Email`, `Money`, `CustomerTier`) |
| Feature envy | Behavior in wrong class | Move methods to `Customer` and `PricingCalculator` |
| Long parameter list | Too many unrelated params in constructor | Group into `OrderInput` parameter object; inject services via DI |

The refactored `OrderProcessor` has one injected dependency and one method parameter -- each a well-typed, cohesive concept. The class is focused, testable, and each piece can evolve independently.

</details>

---

## Experience-Based Questions

These questions test real-world experience. Prepare by mapping them to your own projects and situations.

<details>
<summary>22. Tell me about a time you introduced a design pattern to solve a real problem in a codebase — what was the problem, which pattern did you choose and why, how did you convince the team it was the right approach, and what was the measurable impact on code quality or developer productivity?</summary>

**What the interviewer is looking for:**

- You can identify when a pattern is genuinely needed vs when it's premature abstraction.
- You understand the *problem* the pattern solves, not just the pattern itself.
- You can communicate technical decisions to a team and build consensus.
- You measure impact, not just "it felt cleaner."

**Suggested structure (STAR+):**

1. **Situation:** Describe the codebase state -- what was painful, what was breaking, what was slowing the team down.
2. **Task:** What specific problem needed solving (growing switch statements, tight coupling, testing difficulty, etc.).
3. **Action:** Which pattern you chose, *why that pattern and not alternatives*, how you introduced it (prototype, RFC, pairing session), and how you convinced skeptics.
4. **Result:** Measurable outcome -- fewer bugs, faster feature delivery, easier onboarding, reduced test flakiness.
5. **Reflection:** What you'd do differently, or what you learned about when patterns help vs hurt.

**Example outline to personalize:**

- *Situation:* Payment processing service with a growing if-else chain for different payment providers (Stripe, PayPal, bank transfer). Every new provider meant modifying the core processing function, and each branch had provider-specific error handling that was getting tangled.
- *Task:* Adding Adyen support required touching 6 places in the same function. Two recent bugs were caused by modifications to one provider accidentally affecting another.
- *Action:* Proposed the Strategy pattern -- extracted each provider into a class implementing a `PaymentProvider` interface. Created a small RFC doc showing the before/after, estimated effort (2 days), and demonstrated how adding Adyen would be a single new file + registry entry. Paired with a skeptical teammate to implement the first extraction.
- *Result:* Adding Adyen took 1 day instead of the estimated 3. Test coverage for each provider went from ~40% to ~90% because they were now independently testable. Zero cross-provider bugs in the following 6 months.
- *Reflection:* Should have done it after the third provider, not the fifth. The pattern was clearly needed by provider 3 -- waiting cost us those 2 bugs.

**Key points to hit:**

- Lead with the *problem*, not the pattern. "We had a maintainability problem" not "I wanted to use Strategy."
- Show that you evaluated alternatives (could we just refactor the function? use a config map?).
- Demonstrate team communication -- you didn't just go off and refactor unilaterally.
- Quantify impact where possible (bug count, development time, test coverage).

</details>

<details>
<summary>23. Describe a time you over-engineered a solution or applied the wrong design pattern — what led to that decision, how did you realize the abstraction was hurting more than helping, and what did you do to simplify or reverse course?</summary>

**What the interviewer is looking for:**

- Self-awareness and intellectual honesty -- you can admit mistakes.
- You understand the *cost* of abstraction (as covered in question 15), not just the benefits.
- You know how to recognize over-engineering and have the courage to reverse course.
- Growth mindset -- you learned from the experience and changed your approach.

**Suggested structure (STAR+):**

1. **Situation:** What the codebase/feature looked like and what you were building.
2. **Decision:** What you over-engineered and *why you thought it was the right call at the time* (this is critical -- show the reasoning, not just the mistake).
3. **Realization:** The concrete signals that told you the abstraction was hurting -- teammate confusion, debugging difficulty, feature velocity slowdown, etc.
4. **Correction:** What you did to simplify -- removed the abstraction, inlined code, reverted to a simpler approach.
5. **Lesson:** The principle or heuristic you now use to avoid the same mistake.

**Example outline to personalize:**

- *Situation:* Building a notification system that initially only sent emails. Anticipated needing SMS, push notifications, and Slack.
- *Decision:* Built a full abstract notification framework: `NotificationChannel` interface, `NotificationFactory`, `NotificationRouter`, `NotificationTemplate` system, `NotificationQueue` with retry logic. All for a feature that currently only sent one type of email.
- *Realization:* Three months later, we still only sent emails. A junior developer needed 2 days to figure out where to change the email template because of 6 layers of abstraction. When we finally added SMS, the abstraction didn't fit SMS's requirements (character limits, different template system) and had to be reworked anyway.
- *Correction:* Removed the framework. Replaced it with a simple `EmailService` class -- 50 lines instead of 400. When SMS was actually needed, we built `SmsService` separately, then extracted common patterns *after* seeing what they actually shared.
- *Lesson:* "Build for what you need today, not what you might need tomorrow" (YAGNI). Premature abstraction is more expensive than duplication because the wrong abstraction actively fights against the requirements that actually emerge.

**Key points to hit:**

- Don't be vague -- name the specific pattern or abstraction that was wrong.
- Show the reasoning that led to the mistake (it should sound reasonable, just premature).
- Quantify the cost if possible (developer time wasted, bugs introduced, onboarding friction).
- End with a concrete heuristic you now apply (Rule of Three, YAGNI, etc.).

</details>

<details>
<summary>24. Tell me about a time you refactored a significant portion of a codebase to improve its architecture — what was the original state (code smells, anti-patterns), how did you plan and execute the refactoring without breaking production, and how did you measure success?</summary>

**What the interviewer is looking for:**

- You can identify architectural problems and articulate why they matter (not just "the code was messy").
- You can plan and execute a large refactoring safely -- incrementally, with tests, without stopping feature development.
- You understand risk management -- how to avoid breaking production during structural changes.
- You define and measure success concretely.

**Suggested structure:**

1. **Original state:** Name specific smells/anti-patterns (god classes, tight coupling, no separation of concerns, duplicated logic across services). Explain the *business impact* -- slow feature delivery, frequent bugs, painful onboarding.
2. **Planning:** How you scoped the work, got buy-in, and designed the target architecture. Did you write an RFC? Create a migration plan? Identify the riskiest parts?
3. **Execution strategy:** How you did it incrementally without a "big bang" rewrite. Key techniques: strangler fig pattern, feature flags, parallel implementations, comprehensive test coverage before refactoring.
4. **Risk mitigation:** How you ensured production wasn't broken -- test coverage, canary deployments, monitoring, rollback plan.
5. **Success measurement:** Concrete metrics -- deployment frequency, bug rate, time-to-implement-feature, test execution time, developer satisfaction.

**Example outline to personalize:**

- *Original state:* A monolithic order processing module (~3000 lines) that handled validation, pricing, inventory, payment, and notifications in a single class. Shotgun surgery on every change -- adding a new discount type required modifying 8 methods. Test suite took 15 minutes because everything required full database setup. New team members took 3+ weeks to feel comfortable making changes.
- *Planning:* Drew the target architecture on a whiteboard with the team -- separate services for pricing, inventory, and notifications behind interfaces. Created a Jira epic with ~15 stories, each refactoring one concern. Got buy-in by showing the last 3 sprint's bug count tied to this module (7 bugs in 6 weeks).
- *Execution:* Used the strangler fig approach -- extracted one concern at a time behind an interface, wired it through DI, and verified behavior with existing integration tests. Each extraction was a single PR that could be reviewed and merged independently. Feature development continued in parallel on the un-refactored parts.
- *Risk mitigation:* Wrote characterization tests before each extraction to capture existing behavior. Used feature flags to route traffic through new code paths gradually. Monitored error rates after each deployment.
- *Success measurement:* Test suite dropped from 15 to 4 minutes. Bug count in the module dropped from ~7 per 6 weeks to 1. Adding a new payment method went from 5 days to 1 day. Onboarding time for the module decreased (measured by time-to-first-PR).

**Key points to hit:**

- Emphasize *incremental* over "big bang" -- interviewers want to see production safety awareness.
- Show that you got team buy-in, not just solo cowboy refactoring.
- Connect the refactoring to business outcomes, not just code aesthetics.
- Mention what you'd do differently (scope too large? should have started with tests? communicated better?).

</details>

<details>
<summary>25. Describe a situation where you had to balance code quality and design patterns against delivery speed — how did you decide what to implement properly vs what to ship as technical debt, and how did you ensure the debt was eventually paid down?</summary>

**What the interviewer is looking for:**

- Pragmatism -- you understand that shipping matters, and perfect code that never ships is worthless.
- Judgment -- you can distinguish between acceptable shortcuts and dangerous ones.
- Discipline -- you don't just accumulate debt indefinitely; you have a system for paying it down.
- Communication -- you make tradeoffs transparently, not silently.

**Suggested structure:**

1. **Context:** What was the business pressure (deadline, customer commitment, competitive pressure) and what was the technical ideal?
2. **Decision framework:** How you decided *what* to cut corners on and *what* to do properly. What was your criteria for "acceptable debt" vs "unacceptable shortcut"?
3. **What you shipped:** The specific shortcuts you took and why each was acceptable given the constraints.
4. **Debt tracking:** How you ensured the debt was visible -- tickets, ADRs, TODO comments with ticket references, tech debt backlog.
5. **Paydown:** How and when the debt was actually addressed. What made it possible to prioritize?

**Example outline to personalize:**

- *Context:* Customer-facing feature needed for a major client demo in 3 weeks. Ideal implementation would use the Strategy pattern for multi-provider support, proper repository pattern, and comprehensive error handling. Timeline only allowed for one provider.
- *Decision framework:* Applied two rules: (1) Never take shortcuts on data integrity or security -- those are bugs, not debt. (2) Acceptable to skip extensibility patterns when there's genuinely only one implementation today. Specifically:
  - **Done properly:** Input validation, error handling, database schema (hard to change later), API contract (public interface).
  - **Shipped as debt:** Hardcoded to single provider (no Strategy yet), service called ORM directly (no Repository), minimal test coverage on happy path only.
- *What made this acceptable:* The shortcuts were all *additive* -- adding the Strategy pattern later wouldn't require changing existing code, just wrapping it. The database schema supported multiple providers from day one, so the hard-to-change part was right.
- *Debt tracking:* Created 3 follow-up tickets immediately, linked to the original PR. Added `// TODO(JIRA-1234): Extract to Strategy when second provider is added` comments at each shortcut. Documented the decision in the PR description.
- *Paydown:* When the second provider was actually requested (6 weeks later), the tickets were ready. Refactoring took 2 days because the schema was already right. One of the three debt tickets (comprehensive error handling) was addressed in a subsequent sprint's "tech health" allocation.

**Key points to hit:**

- Show that the decision was *deliberate*, not accidental. You chose to take on debt with eyes open.
- Distinguish between "strategic debt" (conscious, tracked, bounded) and "reckless debt" (cutting corners on things that will cause outages or data corruption).
- Demonstrate a system for tracking and paying down debt -- not just good intentions.
- Show awareness that some debt is fine to live with indefinitely if the code rarely changes (as covered in question 14 -- don't refactor stable code).

</details>
