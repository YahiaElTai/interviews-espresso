# Software Architecture

> **20 questions** — 13 theory, 7 practical

- Problems without architecture: coupling explosion, circular dependencies, untestable code, what breaks as codebases grow
- Dependency Rule: why dependencies must point inward toward higher-level policy, how dependency inversion (not injection) enforces this at the architectural level, relationship to all layered architecture styles
- Testability as an architectural driver: how dependency inversion enables test doubles, pure domain logic vs infrastructure-dependent code, choosing test boundaries based on architecture layers
- Layered (n-tier) architecture: presentation/business/data layers, strengths (simplicity, familiarity), weaknesses (cross-cutting concerns, rigid boundaries), signals a team has outgrown it
- Clean architecture: concentric layers (entities, use cases, interface adapters, frameworks)
- Hexagonal architecture: ports and adapters, how it differs from clean architecture in practice, repository pattern as a canonical adapter example
- DDD core: entities, value objects, aggregates, aggregate boundary rules, domain events
- Bounded contexts: identification, mapping to codebases and teams, context mapping, anti-corruption layers, Conway's Law and how org structure shapes boundaries
- Monolith first: why starting with microservices is premature, well-structured monolith design
- Modular monolith: explicit module boundaries, internal APIs, path to microservice extraction
- Pragmatic architecture: right-sizing complexity to team size and stage, signs of over-engineering (unused abstractions, layers with no logic), architecture as a tool not a goal

---

## Foundational

<details>
<summary>1. What happens to a codebase that grows without intentional architecture — why do coupling explosion, circular dependencies, and untestable code emerge, what specific symptoms signal each problem, and why does the cost of change grow non-linearly as these problems compound?</summary>

Without intentional architecture, every module tends to import whatever it needs directly, creating a web of dependencies that compounds over time.

**Coupling explosion** happens because there are no rules about who can depend on whom. A "quick" import from module A into module B creates an implicit contract. Multiply this across dozens of files and you get a dependency graph where changing one module forces changes in 5 others. **Symptoms**: changing a database schema requires touching HTTP handlers; renaming a type breaks unrelated tests; PRs keep growing in scope because "one more file" needs updating.

**Circular dependencies** emerge when two modules reference each other — often indirectly through a chain (A -> B -> C -> A). This happens because there is no layering to enforce direction. **Symptoms**: mysterious import errors or undefined values at runtime (especially in Node.js where circular `require()` returns a partial module), inability to extract a module into its own package because it drags half the codebase with it.

**Untestable code** results from business logic being tangled with infrastructure (database calls, HTTP clients, file system). When your order validation function directly calls `prisma.order.findMany()`, the only way to test it is to spin up a real database. **Symptoms**: test suite requires full infrastructure to run, tests are slow and flaky, teams skip testing because the setup cost is too high.

**Why cost grows non-linearly**: each new dependency doesn't just add one link — it multiplies the paths a change can propagate through. The growth is quadratic, not linear, which is why a codebase that was "fine" at 10 files becomes painful at 50 and unmaintainable at 200.

</details>

<details>
<summary>2. What is the Dependency Rule — why must dependencies always point inward toward higher-level policy, how does dependency inversion (as distinct from dependency injection) enforce this rule at the architectural level, and why is this rule the common thread across layered, clean, and hexagonal architectures?</summary>

The Dependency Rule states that source code dependencies must always point inward — from low-level details (frameworks, databases, HTTP) toward high-level policy (business rules, domain logic). The inner layers define *what* the system does; outer layers define *how*. Inner layers must never know about outer layers.

**Why inward?** Business rules are the reason the software exists. They change for business reasons. Infrastructure changes for technical reasons (swap Postgres for DynamoDB, Express for Fastify). If your domain logic imports from your HTTP framework, a framework upgrade forces domain changes — the most stable code becomes hostage to the most volatile code.

**Dependency inversion vs. dependency injection** — these are different concepts:

- **Dependency inversion** (the D in SOLID) is an *architectural principle*: high-level modules should not depend on low-level modules; both should depend on abstractions. Concretely, the domain layer defines an interface (e.g., `OrderRepository`), and the infrastructure layer implements it. The dependency arrow *inverts* — instead of domain -> infrastructure, both point toward the abstraction which lives in the domain.
- **Dependency injection** is a *wiring mechanism*: passing a concrete implementation into a constructor or function at runtime. It's one way to *achieve* dependency inversion, but you can also use factory functions, module-level composition, or service locators.

**Why it's the common thread**: Layered architecture says "depend only on the layer below." Clean architecture says "dependencies point toward the center (entities)." Hexagonal architecture says "the application core defines ports; adapters implement them." Different vocabulary, same rule — outer details depend on inner policy, never the reverse. Understanding this shared principle means you can work in any of these styles without confusion.

</details>

<details>
<summary>3. How do layered, clean, hexagonal, and vertical slice architectures relate to each other — what core problem do they all attempt to solve, how do they differ in the way they organize code and enforce boundaries, and why is it important to understand the family rather than just pick one?</summary>

They all solve the same core problem: **separating business logic from infrastructure so the system is testable, maintainable, and adaptable to change.** They differ in how they draw and enforce those boundaries.

**Layered (n-tier)**: Horizontal layers — presentation, business logic, data access. Each layer depends only on the layer below. Simple and intuitive, but dependencies flow *downward toward the database*, making the data layer hardest to change.

**Clean architecture**: Concentric circles — entities at the center, then use cases, interface adapters, frameworks at the outer ring. Key shift: dependencies point *inward toward the domain*, not downward toward the database. The domain defines interfaces that outer layers implement.

**Hexagonal (ports and adapters)**: Same dependency direction as clean architecture, different vocabulary. The core defines "ports" (interfaces), and "adapters" implement them. Emphasizes symmetry between inbound adapters (HTTP, CLI) and outbound adapters (databases, APIs). Less prescriptive about internal layering.

**Vertical slice**: Organizes by feature instead of technical layer. Each slice contains its own handler, validation, data access. Reduces coupling between features at the cost of some duplication.

| Style | Organization axis | Dependency direction | Best for |
|-------|------------------|---------------------|----------|
| Layered | Horizontal (technical layers) | Downward (toward DB) | Simple CRUD apps, small teams |
| Clean | Concentric (domain at center) | Inward (toward domain) | Complex domain logic |
| Hexagonal | Core + adapters | Inward (toward core) | Integration-heavy apps |
| Vertical slice | Per-feature | Within each slice | Feature-independent apps |

**Why understand the family**: In practice, most real codebases blend elements from multiple styles. You might use hexagonal ports/adapters for external integrations, clean architecture layering within your domain, and vertical slices for independent features. If you only know one style as a rigid template, you cannot adapt. Understanding the shared principle (dependency rule, as covered in question 2) lets you make pragmatic choices rather than religious ones.

</details>

## Conceptual Depth

<details>
<summary>4. What is layered (n-tier) architecture, what are its real tradeoffs (both strengths and weaknesses), and what specific signals tell you a team has outgrown it — why do cross-cutting concerns and rigid layer boundaries become painful as complexity grows?</summary>

Layered architecture divides the application into horizontal layers, typically three: **presentation** (HTTP handlers, controllers), **business logic** (services), and **data access** (repositories, ORM). Each layer can only call the layer directly below it.

**Strengths:**
- **Low cognitive overhead** — most developers understand it immediately. Onboarding is fast.
- **Clear separation of concerns** — HTTP handling is separate from business logic, which is separate from database queries.
- **Familiar tooling support** — frameworks like NestJS, Express patterns, and most tutorials naturally guide toward this structure.
- **Good enough for many apps** — a standard CRUD API with moderate complexity fits this model well.

**Weaknesses:**
- **Dependencies point toward the database**, not the domain. The data access layer is the foundation everything depends on. Changing your database or ORM ripples upward through the entire codebase.
- **Cross-cutting concerns break the model** — logging, authentication, validation, error handling, and caching don't belong to any single layer. Teams end up either duplicating logic across layers or creating awkward "utility" modules that everything imports, defeating the purpose of layering.
- **Business logic leaks** — without strict enforcement, business rules creep into controllers ("if user is admin, skip validation") and into repositories ("apply discount in the query"). The service layer becomes a thin pass-through.
- **Rigid horizontal slicing** makes it hard to reason about features. To understand "how does order creation work?" you have to trace through 3+ files across 3 directories.

**Signals you've outgrown it:**
- The service layer is a hollow pass-through while business logic scatters across controllers, services, and repository queries with no clear owner.
- Adding a new feature requires touching every layer even when the change is logically simple.
- You need to share logic between two services but the layering makes it awkward (no clear "where does shared domain logic go?").
- Cross-cutting concerns (auth, caching, audit logging) are duplicated or inconsistently applied because they don't fit the layer model.

When these signals appear, it is time to introduce domain-centric architecture (clean/hexagonal) where the dependency direction inverts toward the domain rather than the database.

</details>

<details>
<summary>5. Explain clean architecture's concentric layers (entities, use cases, interface adapters, frameworks) — what belongs in each layer, why does the dependency direction always point inward, and what goes wrong when teams put business logic in the wrong layer?</summary>

Clean architecture organizes code in concentric rings. The innermost ring is the most stable (changes rarely); the outermost is the most volatile (changes frequently). Dependencies always point inward.

**Layer 1 — Entities (innermost):** Enterprise-wide business rules and domain objects. These are pure — no framework imports, no database types, no I/O. An `Order` entity with methods like `addLineItem()` and `calculateTotal()` belongs here. Entities encapsulate the most general, reusable business rules that would exist even if the application didn't.

**Layer 2 — Use Cases (application logic):** Application-specific business rules that orchestrate entities to fulfill a specific user goal. A `CreateOrderUseCase` that validates input, calls entity methods, and invokes repository/notification interfaces lives here. Use cases define *what the application does* — they know about entities but not about HTTP or databases. They declare the interfaces (ports) they need from the outer world.

**Layer 3 — Interface Adapters:** Translates between the format the use cases expect and the format the external world uses. Controllers convert HTTP requests into use case input DTOs. Repository implementations convert domain objects to/from database rows. Presenters format use case output into API responses. This layer is the boundary — it imports from both directions but the *source code dependency* still points inward (adapters depend on use case interfaces, not the other way around).

**Layer 4 — Frameworks & Drivers (outermost):** Express/Fastify setup, Prisma client configuration, message queue connections, third-party SDK initialization. Pure infrastructure glue. This layer is where you wire everything together — create the concrete implementations and inject them into the inner layers.

**Why inward?** As covered in question 2, the dependency rule ensures the most stable code (business rules) never depends on the most volatile code (frameworks). You can swap Express for Fastify without touching a single use case or entity.

**What goes wrong with misplaced business logic:**

- **Business logic in controllers**: Validation rules, authorization checks, and transformation logic in HTTP handlers become untestable without HTTP mocking. The same logic can't be reused from a CLI or message queue consumer — you end up duplicating it.
- **Business logic in repositories**: Discount calculations in SQL queries, filtering rules in ORM queries. Now your business rules are expressed in SQL, untestable without a database, and invisible to anyone reading the domain layer.
- **Business logic in the framework layer**: Express middleware that enforces domain rules. Tied to a specific framework, scattered across configuration files, and discovered only by tracing the middleware chain.

The pattern: misplaced business logic is always **harder to test, harder to find, and harder to reuse**. Clean architecture's layering exists to make the "right" placement obvious.

</details>

<details>
<summary>6. What is hexagonal architecture (ports and adapters) — how do ports and adapters work, why does it make infrastructure swappable, and how does it differ from clean architecture in practice despite sharing the same dependency direction principle?</summary>

Hexagonal architecture (Alistair Cockburn, 2005) structures the application as a core surrounded by ports and adapters. The "hexagon" is just a visual metaphor for the application core — the shape has no special meaning beyond emphasizing that the core has multiple faces (ports) connecting to the outside world.

**Ports** are interfaces defined by the application core that describe what it needs or what it offers. There are two types:
- **Driving ports** (primary/inbound): How the outside world triggers the application. Example: `CreateOrderUseCase` interface — the core says "I can create orders if you give me this input."
- **Driven ports** (secondary/outbound): What the application needs from the outside world. Example: `OrderRepository` interface, `PaymentGateway` interface — the core says "I need something that can store orders and process payments."

**Adapters** are concrete implementations that connect ports to real infrastructure:
- **Driving adapters**: HTTP controllers, CLI handlers, message queue consumers — they call the driving ports.
- **Driven adapters**: Postgres repository, Stripe payment client, SES email sender — they implement the driven ports.

**Why infrastructure is swappable:** The core only knows about the port interfaces it defined. The Postgres adapter implements `OrderRepository`; so could a DynamoDB adapter or an in-memory fake. Swapping infrastructure means writing a new adapter — zero changes to the core. This is dependency inversion (as covered in question 2) applied systematically.

**How it differs from clean architecture in practice:**

| Aspect | Clean Architecture | Hexagonal |
|--------|-------------------|-----------|
| Internal structure | Prescribes 4 concentric layers with specific roles | Just says "core + ports + adapters" — less prescriptive about internal layers |
| Vocabulary | Entities, use cases, interface adapters, frameworks | Ports (driving/driven), adapters (primary/secondary) |
| Emphasis | Layering and separation within the domain | Symmetry between inbound and outbound — treats "things that call us" and "things we call" uniformly |
| Practical difference | Tends to produce more files/layers (entity, use case, adapter per concern) | Often simpler structure — core with interfaces, adapters that implement them |

In practice, most teams blend the two: they use hexagonal's port/adapter vocabulary for external integrations and clean architecture's internal layering (entities vs. use cases) within the core. The styles are complementary, not competing.

</details>

<details>
<summary>7. Explain the core DDD building blocks — what are entities, value objects, and aggregates, what rules govern aggregate boundaries (consistency boundary, single transaction, reference by ID), and how do domain events allow aggregates to communicate without coupling?</summary>

**Entities** are objects with a persistent identity. Two `Order` objects with the same `id` are the same order even if their data differs (the order was updated). Identity is what matters, not attribute equality. Entities have lifecycle — they are created, change state, and may be deleted.

**Value objects** are objects defined entirely by their attributes, with no identity. Two `Money` objects with `{ amount: 100, currency: 'EUR' }` are interchangeable. Value objects should be immutable — instead of mutating, you create a new instance. Good candidates: `EmailAddress`, `DateRange`, `Address`, `Money`. They encapsulate validation (an `EmailAddress` is always valid by construction) and domain-specific operations (`money.add(otherMoney)` handles currency matching).

**Aggregates** are clusters of entities and value objects treated as a single unit for data changes. The **aggregate root** is the entry point — all modifications go through it.

**Aggregate boundary rules:**

1. **Consistency boundary**: Everything inside one aggregate must be consistent after every operation. An `Order` aggregate ensures its line items always add up to the correct total — this invariant is enforced within the aggregate, in a single transaction.
2. **Single transaction**: One transaction = one aggregate. Never modify multiple aggregates in a single transaction. This is how you maintain clear consistency boundaries and avoid locking issues.
3. **Reference by ID**: Aggregates reference other aggregates by their ID, not by holding direct object references. An `OrderLineItem` stores `productId: string`, not `product: Product`. This prevents one aggregate from reaching into another's internals and keeps them independently loadable.

**Domain events** are the mechanism for cross-aggregate communication without coupling. When an aggregate completes an action, it emits an event describing what happened: `OrderPlaced`, `PaymentReceived`, `InventoryReserved`. Other aggregates (or services) subscribe to these events and react independently.

Why events, not direct calls? If `Order` directly calls `Inventory.reserve()`, the Order aggregate is coupled to the Inventory aggregate. With events, `Order` simply publishes `OrderPlaced` — it does not know or care who listens. The inventory service handles `OrderPlaced` in its own transaction, maintaining the one-aggregate-per-transaction rule. This also means eventual consistency between aggregates, which is the expected tradeoff — strong consistency within an aggregate, eventual consistency between them.

</details>

<details>
<summary>8. What are bounded contexts in DDD — how do you identify them in a complex domain, how do they map to codebases and teams, and why does getting boundaries wrong lead to distributed monolith problems?</summary>

A **bounded context** is a boundary within which a particular domain model is defined and consistent. The same real-world concept can mean different things in different contexts. "Customer" in the billing context has payment methods and invoices. "Customer" in the shipping context has addresses and delivery preferences. Trying to create one unified `Customer` model that serves both leads to a bloated, confused object that changes for conflicting reasons.

**How to identify bounded contexts:**

- **Language boundaries**: When the same word means different things to different teams or in different conversations, that is a context boundary. If the sales team and the warehouse team use "order" to mean different things, those are likely different contexts.
- **Team boundaries**: Conway's Law — system structure mirrors org structure. If separate teams own separate parts of the domain, those often map to separate bounded contexts. Fighting this mapping usually fails.
- **Consistency requirements**: Data that must be strongly consistent belongs in the same context. Data that can tolerate eventual consistency can live in separate contexts.
- **Rate of change**: Parts of the domain that change for different reasons or at different speeds are candidates for separate contexts.

**Mapping to codebases and teams:**

Ideally, one bounded context = one team = one deployable (or one module in a modular monolith). Each context has its own models, its own database schema (or at least its own set of tables), and its own internal logic. Contexts communicate through well-defined interfaces — APIs, events, or anti-corruption layers.

An **anti-corruption layer (ACL)** is a translation layer between two bounded contexts. When context A consumes data from context B, the ACL translates B's model into A's model. This prevents one context's design choices from leaking into and corrupting another's model.

**Why wrong boundaries create distributed monoliths:**

If you split into microservices along the wrong boundaries, closely related data ends up in different services. The symptoms are unmistakable:
- Services need synchronous calls to each other for every operation (chatty communication)
- Changes to one service require coordinated deployments of 2-3 other services
- Distributed transactions or sagas span multiple services for what should be a simple operation
- Data is duplicated across services with no clear owner, leading to inconsistency

This is a distributed monolith — you have all the complexity of microservices (network failures, eventual consistency, deployment coordination) with none of the benefits (independent deployment, independent scaling, team autonomy). Getting the bounded context boundaries right is the single most important architectural decision when decomposing a system.

</details>

<details>
<summary>9. What is the repository pattern — why does it define an interface in the domain layer but implement it in the infrastructure layer, how does this apply dependency inversion, and what problems emerge when repositories leak infrastructure concerns (query builders, ORM types) into the domain?</summary>

The repository pattern provides a collection-like interface for accessing domain objects, abstracting away the persistence mechanism. To the domain layer, a repository looks like an in-memory collection: `findById()`, `save()`, `delete()`. How data is actually stored (Postgres, DynamoDB, a file) is hidden behind the interface.

**Why the interface lives in the domain and the implementation in infrastructure:**

This is dependency inversion in action (as covered in question 2). The domain layer defines what it *needs* — "I need something that can find and save orders." That is the interface:

```typescript
// domain/ports/order-repository.ts
interface OrderRepository {
  findById(id: string): Promise<Order | null>;
  save(order: Order): Promise<void>;
}
```

The infrastructure layer provides what it *can do* — "I can store orders in Postgres":

```typescript
// infrastructure/adapters/postgres-order-repository.ts
class PostgresOrderRepository implements OrderRepository {
  async findById(id: string): Promise<Order | null> {
    const row = await this.pool.query('SELECT * FROM orders WHERE id = $1', [id]);
    return row ? this.toDomain(row) : null;
  }
}
```

The dependency arrow points from infrastructure toward domain (the implementation imports the interface), not the other way around. The domain has zero knowledge of Postgres.

**What happens when repositories leak infrastructure concerns:**

- **ORM types in domain signatures**: If `findById` returns a Prisma-generated `PrismaOrder` type instead of a domain `Order`, the domain layer now transitively depends on Prisma. Changing ORMs means changing the domain — exactly what the pattern was designed to prevent.
- **Query builder exposure**: If the repository interface accepts `where` clauses or filter objects in ORM-specific syntax, callers must understand the ORM to use the repository. The abstraction is broken.
- **Lazy loading leaks**: If the repository returns entities with ORM-managed lazy-loaded relations, calling code might trigger unexpected database queries deep in the domain layer. Business logic now has hidden I/O.
- **Transaction management leaking up**: If the repository forces callers to pass transaction objects or database connections, the domain layer is coupled to infrastructure's transaction model.

The fix is always the same: the repository interface should speak in domain terms only — domain entities as return types, domain-meaningful parameters as inputs. All translation between domain and infrastructure happens inside the adapter implementation.

</details>

<details>
<summary>10. Why is "monolith first" considered the pragmatic default — what goes wrong when teams start with microservices before understanding their domain boundaries, what does a well-structured monolith look like, and how does it differ from a "big ball of mud"?</summary>

**Why monolith first:** At the start of a project, you don't understand your domain well enough to draw correct service boundaries. Microservice boundaries are expensive to change — they involve API contracts, separate deployments, network communication, and data ownership. Drawing them wrong early means building a distributed monolith (as described in question 8). A monolith lets you refactor freely — moving code between modules is a file move, not a cross-service migration.

Martin Fowler's advice: "You shouldn't start a new project with microservices, even if you're sure your application will be big enough to make it worthwhile." The reason is simple: the cost of getting boundaries wrong in a monolith is low (refactor), while the cost in microservices is high (rewrite integrations, migrate data, update contracts).

**What goes wrong with premature microservices:**
- Teams guess at boundaries and get them wrong. Features that should be one service span three, requiring constant cross-service coordination.
- Operational complexity arrives before the team has the maturity to handle it — distributed tracing, service mesh, contract testing, deployment orchestration.
- Simple operations (join two tables) become distributed queries. What was a SQL join is now an API call with latency, failure modes, and consistency challenges.
- Development velocity drops because every feature requires touching multiple repos, coordinating deployments, and debugging network issues.

**Well-structured monolith vs. big ball of mud:**

| Aspect | Big ball of mud | Well-structured monolith |
|--------|----------------|------------------------|
| Module boundaries | None — any code imports anything | Explicit modules with defined public APIs |
| Data access | Any code queries any table | Each module owns its tables; others go through the module's API |
| Dependencies | Circular, implicit, everywhere | Directed — modules declare dependencies, no cycles |
| Domain logic | Scattered across controllers, middleware, queries | Concentrated in domain/service layer within each module |
| Extractability | Requires full rewrite | Individual modules can be extracted to services with clear cut points |

A well-structured monolith is essentially a modular monolith (covered in question 11) — it has the internal boundaries of microservices without the operational overhead of distributed deployment. When you eventually *do* understand your domain well enough to split, the boundaries are already drawn.

</details>

<details>
<summary>11. What is a modular monolith — how do you enforce explicit module boundaries and internal APIs within a single deployable, what rules prevent modules from becoming coupled, and how does this architecture create a clean extraction path to microservices when the time comes?</summary>

A modular monolith is a single deployable application where the code is organized into well-defined modules with explicit boundaries, public APIs, and private internals. It gives you the development simplicity of a monolith with the structural discipline of microservices.

**Enforcing module boundaries and internal APIs:**

Each module exposes a public API (a set of exported functions, types, or a service interface) and hides everything else. In TypeScript, the primary enforcement mechanism is the module's `index.ts` — only what's re-exported from there is public. Everything else is internal:

```
modules/
  orders/
    index.ts          # Public API — only these exports are importable
    internal/
      order.entity.ts
      order.service.ts
      order.repository.ts
  inventory/
    index.ts
    internal/
      ...
```

Other modules import only from `modules/orders`, never from `modules/orders/internal/*`. This can be enforced with ESLint rules (e.g., `eslint-plugin-import` with restricted path patterns) or TypeScript path aliases that make internal paths unresolvable from outside the module.

**Rules that prevent coupling:**

1. **No cross-module database access**: Each module owns its tables. The orders module never queries the inventory module's tables directly — it calls the inventory module's public API instead. This is the most important rule because shared database access is the fastest path to invisible coupling.
2. **No importing internal types**: Modules share data through DTOs defined in their public API, not through internal domain entities. If `orders` needs inventory data, it calls `inventory.getStock(productId)` and receives a simple DTO, not an `InventoryItem` entity.
3. **No circular dependencies between modules**: Module A can depend on module B, but then B cannot depend on A. If two modules need to communicate bidirectionally, use events (as covered in question 7).
4. **Communication through events for side effects**: When an order is placed and inventory needs to be reserved, the orders module emits an `OrderPlaced` event. The inventory module subscribes. No direct call, no coupling.

**Clean extraction path to microservices:**

When a module needs to become its own service (for independent scaling, separate team ownership, different deployment cadence), the extraction is straightforward because the boundaries already exist:

1. The module's public API becomes the service's HTTP/gRPC API — the interface signatures are already defined.
2. The module's owned tables become the service's database — no shared tables to untangle.
3. In-process event publishing becomes message queue publishing — the event contracts are already defined.
4. Other modules that called the public API now call an HTTP client adapter instead — same interface, different transport.

The key insight: the hardest part of extracting a microservice is finding the boundary. In a modular monolith, you already drew it.

</details>

<details>
<summary>12. What does pragmatic architecture mean in practice — how do you right-size architecture decisions for your team's size and codebase complexity, what are the concrete signs of over-engineering (premature abstraction, unnecessary indirection, pattern stuffing), and when is "good enough" the right call?</summary>

Pragmatic architecture means choosing the simplest architecture that handles your *current* complexity while leaving room for the *likely next* evolution — not building for every hypothetical future. Architecture is a tool to manage complexity, not a goal in itself.

**Right-sizing for team and complexity:**

- **2-3 person team, early product**: Simple layered architecture. Maybe just `routes/`, `services/`, `repositories/`. Don't introduce clean architecture for a CRUD API with 10 endpoints. The overhead of ports, adapters, and use case classes slows a small team down when the domain is still being discovered.
- **5-10 person team, growing product**: Modular monolith with clear module boundaries. Possibly hexagonal patterns for modules with complex integrations. Enforce rules with linting, not just convention.
- **10+ person team, established product**: Full domain-centric architecture where it matters (complex modules), simpler patterns where it doesn't (CRUD modules). Different modules can have different levels of architectural investment.

The heuristic: **add architectural structure when the cost of not having it becomes visible** — when refactoring is hard, when tests are slow, when two teams step on each other. Don't add it preemptively.

**Concrete signs of over-engineering:**

- **Premature abstraction**: An `OrderRepository` interface with one implementation *and no test fakes* — if you're testing against the real database anyway, the interface adds indirection with no benefit. But if you use fakes for testing (as shown in Q15), the interface earns its keep even with one production implementation.
- **Unnecessary indirection**: A request passes through Controller -> Validator -> Mapper -> UseCase -> DomainService -> Repository -> Mapper -> Response, where the UseCase and DomainService are one-line pass-throughs. Each layer should add value — if it doesn't, it's noise.
- **Pattern stuffing**: Using the Strategy pattern, Factory pattern, and Observer pattern in a module with one code path. Patterns are solutions to specific problems — applying them without the problem is cargo culting.
- **Layers with no logic**: An "application layer" where every method is `return this.repository.findById(id)`. The layer exists because the architecture diagram says it should, not because there's logic to put there.
- **Premature generalization**: Building a "plugin system" for something that currently has one plugin and no plans for more.

**When "good enough" is the right call:**

- When the team understands the code and can change it quickly — even if it's not "architecturally pure"
- When the feature is experimental and might be deleted in 3 months
- When the added complexity of "doing it right" exceeds the complexity of the problem itself
- When you have a clear path to refactor later and the cost of that refactor is low (internal code, no public API contracts)

The test: *Would a new team member understand this code in 30 minutes?* If yes, your architecture is working. If it takes 2 hours just to trace a request through the layers, you may have over-invested.

</details>

<details>
<summary>13. Why should testability be treated as an architectural driver rather than an afterthought — how does dependency inversion enable test doubles at layer boundaries, what is the practical difference between testing pure domain logic and infrastructure-dependent code, and how do you choose which architectural boundaries become test boundaries?</summary>

**Why testability is an architectural driver:** If you design your architecture without thinking about testing, you end up with code that's hard to test — not because the tests are wrong, but because the code's structure prevents isolation. Testability is a forcing function for good design: code that's easy to test is naturally decoupled, has clear interfaces, and separates concerns. If you can't test a component without spinning up a database, a message queue, and three other services, that's an architecture problem, not a testing problem.

**How dependency inversion enables test doubles:**

When the domain layer defines interfaces (ports) and the infrastructure layer implements them (as covered in questions 2 and 9), you can substitute any implementation at test time:

```typescript
// The use case depends on the interface, not the implementation
class CreateOrderUseCase {
  constructor(private orderRepo: OrderRepository) {}

  async execute(input: CreateOrderInput): Promise<Order> {
    const order = Order.create(input);
    await this.orderRepo.save(order);
    return order;
  }
}

// In tests — inject a fake, no database needed
const fakeRepo: OrderRepository = {
  save: async (order) => { saved.push(order); },
  findById: async (id) => saved.find(o => o.id === id) ?? null,
};
const useCase = new CreateOrderUseCase(fakeRepo);
```

Without dependency inversion, `CreateOrderUseCase` would directly import and call `PostgresOrderRepository`, and you would need a real Postgres instance to test it.

**Pure domain logic vs. infrastructure-dependent code:**

- **Pure domain logic** (entities, value objects, domain services): No I/O, no side effects. Test with simple unit tests — call a method, assert the result. These tests are fast (sub-millisecond), deterministic, and never flaky. Example: testing that `Order.addLineItem()` correctly recalculates the total.
- **Infrastructure-dependent code** (repositories, HTTP clients, queue publishers): Involves I/O, external systems, network calls. Requires either test doubles (mocks/fakes) or integration tests against real infrastructure (test databases, test containers). These tests are slower and more complex to set up.

The architectural payoff: the more business logic you push into pure domain objects (inner layers), the more of your system you can test with fast, simple unit tests. Infrastructure-dependent code in the outer layers is tested with integration tests, but there's less of it and it changes less often.

**Choosing which boundaries become test boundaries:**

1. **Always test across the domain boundary**: The interface between use cases and infrastructure is your primary test seam. Mock/fake the infrastructure ports when testing use cases. This gives you the highest value — testing business logic in isolation.
2. **Test infrastructure adapters with integration tests**: Verify that your Postgres repository actually reads/writes correctly, that your HTTP client handles errors. These tests use real infrastructure (test containers) but don't test business logic.
3. **Skip testing pure pass-through layers**: If a controller just validates input and calls a use case, a few integration tests through the HTTP layer are sufficient — no need for isolated controller unit tests.
4. **Test at module boundaries in a modular monolith**: Each module's public API is a natural test boundary. Test the module as a unit through its public interface, mocking other modules' APIs.

The rule of thumb: **test boundaries should align with architectural boundaries.** If your architecture has a clear seam between domain and infrastructure, that's where your tests should split between unit and integration.

</details>

## Practical — Implementation & Refactoring

<details>
<summary>14. Show how you would structure a TypeScript/Node.js project using clean or hexagonal architecture — lay out the directory structure, show a concrete example of an entity, a use case, a port (interface), and an adapter (implementation), and explain how the dependency rule is enforced at the code level so that inner layers never import from outer layers.</summary>

**Directory structure** (blending clean architecture's internal layering with hexagonal vocabulary):

```
src/
  domain/                  # Inner layer — pure business logic, zero dependencies
    entities/
      order.ts
      order-line-item.ts
    value-objects/
      money.ts
      email-address.ts
    ports/                 # Interfaces the domain needs from the outside
      order-repository.ts
      payment-gateway.ts
      event-publisher.ts

  application/             # Use cases — orchestrate domain objects
    use-cases/
      create-order.ts
      cancel-order.ts
    dtos/
      create-order-input.ts
      order-output.ts

  infrastructure/          # Outer layer — implements ports with real technology
    adapters/
      postgres-order-repository.ts
      stripe-payment-gateway.ts
      sqs-event-publisher.ts
    http/
      controllers/
        order-controller.ts
      middleware/
        auth.ts
    config/
      database.ts

  composition-root.ts      # Wires everything together
```

**Entity** — pure domain logic, no imports from outer layers:

```typescript
// domain/entities/order.ts
import { Money } from '../value-objects/money';

export class Order {
  private lineItems: OrderLineItem[] = [];
  private _status: 'draft' | 'placed' | 'cancelled' = 'draft';

  constructor(
    public readonly id: string,
    public readonly customerId: string,
  ) {}

  addLineItem(productId: string, quantity: number, unitPrice: Money): void {
    if (this._status !== 'draft') throw new Error('Cannot modify a placed order');
    if (quantity < 1 || quantity > 100) throw new Error('Quantity must be 1-100');
    this.lineItems.push(new OrderLineItem(productId, quantity, unitPrice));
  }

  get total(): Money {
    return this.lineItems.reduce(
      (sum, item) => sum.add(item.subtotal),
      Money.zero('EUR'),
    );
  }

  place(): void {
    if (this.lineItems.length === 0) throw new Error('Cannot place empty order');
    this._status = 'placed';
  }
}
```

**Port** — interface defined in the domain:

```typescript
// domain/ports/order-repository.ts
import { Order } from '../entities/order';

export interface OrderRepository {
  findById(id: string): Promise<Order | null>;
  save(order: Order): Promise<void>;
}
```

**Use case** — imports from domain only, never from infrastructure:

```typescript
// application/use-cases/create-order.ts
import { Order } from '../../domain/entities/order';
import { OrderRepository } from '../../domain/ports/order-repository';
import { Money } from '../../domain/value-objects/money';
import { CreateOrderInput } from '../dtos/create-order-input';

export class CreateOrderUseCase {
  constructor(private orderRepo: OrderRepository) {}

  async execute(input: CreateOrderInput): Promise<string> {
    const order = new Order(crypto.randomUUID(), input.customerId);
    for (const item of input.lineItems) {
      order.addLineItem(item.productId, item.quantity, new Money(item.price, 'EUR'));
    }
    order.place();
    await this.orderRepo.save(order);
    return order.id;
  }
}
```

**Adapter** — implements the port, imports from domain (dependency points inward):

```typescript
// infrastructure/adapters/postgres-order-repository.ts
import { Pool } from 'pg';
import { Order } from '../../domain/entities/order';
import { OrderRepository } from '../../domain/ports/order-repository';

export class PostgresOrderRepository implements OrderRepository {
  constructor(private pool: Pool) {}

  async findById(id: string): Promise<Order | null> {
    const { rows } = await this.pool.query(
      'SELECT * FROM orders WHERE id = $1', [id]
    );
    return rows[0] ? this.toDomain(rows[0]) : null;
  }

  async save(order: Order): Promise<void> {
    await this.pool.query(
      'INSERT INTO orders (id, customer_id, status, total) VALUES ($1, $2, $3, $4) ON CONFLICT (id) DO UPDATE SET status = $3, total = $4',
      [order.id, order.customerId, order.status, order.total.amount]
    );
  }

  private toDomain(row: any): Order { /* map DB row to domain entity */ }
}
```

**Enforcing the dependency rule at the code level:**

1. **Import discipline**: Domain files never import from `infrastructure/` or `application/`. Use cases never import from `infrastructure/`. This is the most fundamental rule.
2. **ESLint `import/no-restricted-paths`**: Automate enforcement so CI fails if someone adds a forbidden import:
   ```json
   {
     "rules": {
       "import/no-restricted-paths": ["error", {
         "zones": [
           { "target": "./src/domain", "from": "./src/infrastructure" },
           { "target": "./src/domain", "from": "./src/application" },
           { "target": "./src/application", "from": "./src/infrastructure" }
         ]
       }]
     }
   }
   ```
3. **Composition root**: All wiring happens in one place (`composition-root.ts`) that creates concrete implementations and injects them. This is the only file that knows about everything.

```typescript
// composition-root.ts
import { Pool } from 'pg';
import { PostgresOrderRepository } from './infrastructure/adapters/postgres-order-repository';
import { CreateOrderUseCase } from './application/use-cases/create-order';
import { OrderController } from './infrastructure/http/controllers/order-controller';

const pool = new Pool({ connectionString: process.env.DATABASE_URL });
const orderRepo = new PostgresOrderRepository(pool);
const createOrder = new CreateOrderUseCase(orderRepo);
const orderController = new OrderController(createOrder);
```

The composition root is the only place where outer-layer types flow inward — but it's wiring, not business logic. The domain and application layers remain pure.

</details>

<details>
<summary>15. Implement the repository pattern in TypeScript with dependency inversion — show the domain-layer interface, an infrastructure-layer implementation (e.g., using Prisma or a plain SQL client), and demonstrate how the application service depends only on the interface. Explain how this makes the service testable with a fake/mock repository.</summary>

Building on the concepts from questions 9 and 14, here is a complete working example.

**Domain-layer interface** — speaks only in domain terms:

```typescript
// domain/ports/product-repository.ts
import { Product } from '../entities/product';

export interface ProductRepository {
  findById(id: string): Promise<Product | null>;
  findByCategory(category: string): Promise<Product[]>;
  save(product: Product): Promise<void>;
  delete(id: string): Promise<void>;
}
```

**Domain entity** — pure, no infrastructure knowledge:

```typescript
// domain/entities/product.ts
export class Product {
  constructor(
    public readonly id: string,
    private _name: string,
    private _price: number,
    private _category: string,
  ) {
    if (_price < 0) throw new Error('Price cannot be negative');
  }

  rename(name: string): void {
    if (!name.trim()) throw new Error('Name cannot be empty');
    this._name = name;
  }

  get name() { return this._name; }
  get price() { return this._price; }
  get category() { return this._category; }
}
```

**Infrastructure implementation** — follows the same adapter pattern shown in Q14 (implements the port, maps between domain and DB rows). The Postgres adapter uses a plain SQL client and the `toDomain` mapping pattern.

**Application service** — depends only on the interface:

```typescript
// application/use-cases/rename-product.ts
import { ProductRepository } from '../../domain/ports/product-repository';

export class RenameProductUseCase {
  constructor(private productRepo: ProductRepository) {} // interface, not concrete class

  async execute(productId: string, newName: string): Promise<void> {
    const product = await this.productRepo.findById(productId);
    if (!product) throw new Error(`Product ${productId} not found`);
    product.rename(newName); // domain validation happens here
    await this.productRepo.save(product);
  }
}
```

**Testing with a fake repository** — no database, no mocks library needed:

```typescript
// tests/rename-product.test.ts
import { Product } from '../../domain/entities/product';
import { ProductRepository } from '../../domain/ports/product-repository';
import { RenameProductUseCase } from '../../application/use-cases/rename-product';

class FakeProductRepository implements ProductRepository {
  private store = new Map<string, Product>();

  async findById(id: string) { return this.store.get(id) ?? null; }
  async findByCategory(category: string) {
    return [...this.store.values()].filter(p => p.category === category);
  }
  async save(product: Product) { this.store.set(product.id, product); }
  async delete(id: string) { this.store.delete(id); }

  // Test helper — seed data
  seed(product: Product) { this.store.set(product.id, product); }
}

describe('RenameProductUseCase', () => {
  it('renames an existing product', async () => {
    const repo = new FakeProductRepository();
    repo.seed(new Product('p1', 'Old Name', 29.99, 'electronics'));
    const useCase = new RenameProductUseCase(repo);

    await useCase.execute('p1', 'New Name');

    const updated = await repo.findById('p1');
    expect(updated?.name).toBe('New Name');
  });

  it('throws when product does not exist', async () => {
    const repo = new FakeProductRepository();
    const useCase = new RenameProductUseCase(repo);

    await expect(useCase.execute('missing', 'Name')).rejects.toThrow('not found');
  });

  it('rejects empty names via domain validation', async () => {
    const repo = new FakeProductRepository();
    repo.seed(new Product('p1', 'Name', 10, 'books'));
    const useCase = new RenameProductUseCase(repo);

    await expect(useCase.execute('p1', '  ')).rejects.toThrow('cannot be empty');
  });
});
```

**Why this works:** The use case has zero knowledge of Postgres, pg, connection pools, or SQL. Tests run in milliseconds with no infrastructure. The fake implements the same interface, so it exercises the same code paths as the real adapter. You test the Postgres adapter separately with integration tests against a real database (or test container) — but that's a focused test of SQL correctness, not business logic.

</details>

<details>
<summary>16. Define an aggregate in TypeScript with proper boundary enforcement — show an aggregate root with child entities/value objects, enforce invariants inside the aggregate (e.g., order line quantity limits, total price validation), emit a domain event, and explain why external code should never modify child entities directly.</summary>

Building on the DDD concepts from question 7, here is a complete aggregate implementation.

**Value object** — immutable, equality by value:

```typescript
// domain/value-objects/money.ts
export class Money {
  constructor(
    public readonly amount: number,
    public readonly currency: string,
  ) {
    if (amount < 0) throw new Error('Amount cannot be negative');
  }

  add(other: Money): Money {
    if (this.currency !== other.currency) throw new Error('Currency mismatch');
    return new Money(this.amount + other.amount, this.currency);
  }

  multiply(factor: number): Money {
    return new Money(Math.round(this.amount * factor * 100) / 100, this.currency);
  }

  static zero(currency: string): Money {
    return new Money(0, currency);
  }

  equals(other: Money): boolean {
    return this.amount === other.amount && this.currency === other.currency;
  }
}
```

**Domain event** — describes something that happened:

```typescript
// domain/events/domain-event.ts
export interface DomainEvent {
  readonly eventType: string;
  readonly occurredAt: Date;
  readonly aggregateId: string;
}

export interface OrderPlacedEvent extends DomainEvent {
  eventType: 'OrderPlaced';
  customerId: string;
  totalAmount: number;
  totalCurrency: string;
  lineItemCount: number;
}
```

**Child entity** — never exposed mutably outside the aggregate:

```typescript
// domain/entities/order-line-item.ts
export class OrderLineItem {
  constructor(
    public readonly id: string,
    public readonly productId: string,
    private _quantity: number,
    public readonly unitPrice: Money,
  ) {
    this.validateQuantity(_quantity);
  }

  get quantity(): number { return this._quantity; }

  get subtotal(): Money {
    return this.unitPrice.multiply(this._quantity);
  }

  // Package-private: only the aggregate root should call this
  updateQuantity(newQuantity: number): void {
    this.validateQuantity(newQuantity);
    this._quantity = newQuantity;
  }

  private validateQuantity(qty: number): void {
    if (qty < 1) throw new Error('Quantity must be at least 1');
    if (qty > 99) throw new Error('Quantity cannot exceed 99 per line item');
  }
}
```

**Aggregate root** — the single entry point for all modifications:

```typescript
// domain/entities/order.ts
import { Money } from '../value-objects/money';
import { OrderLineItem } from './order-line-item';
import { DomainEvent, OrderPlacedEvent } from '../events/domain-event';

const MAX_LINE_ITEMS = 50;
const MAX_ORDER_TOTAL = new Money(10_000, 'EUR');

export class Order {
  private _lineItems: OrderLineItem[] = [];
  private _status: 'draft' | 'placed' | 'cancelled' = 'draft';
  private _domainEvents: DomainEvent[] = [];

  constructor(
    public readonly id: string,
    public readonly customerId: string,
  ) {}

  // Read-only access — returns a copy, not the internal array
  get lineItems(): ReadonlyArray<OrderLineItem> {
    return [...this._lineItems];
  }

  get status(): string { return this._status; }

  get total(): Money {
    return this._lineItems.reduce(
      (sum, item) => sum.add(item.subtotal),
      Money.zero('EUR'),
    );
  }

  addLineItem(id: string, productId: string, quantity: number, unitPrice: Money): void {
    this.assertDraft();
    if (this._lineItems.length >= MAX_LINE_ITEMS) {
      throw new Error(`Order cannot have more than ${MAX_LINE_ITEMS} line items`);
    }

    const item = new OrderLineItem(id, productId, quantity, unitPrice);

    // Validate total won't exceed limit with the new item
    const projectedTotal = this.total.add(item.subtotal);
    if (projectedTotal.amount > MAX_ORDER_TOTAL.amount) {
      throw new Error(`Order total cannot exceed ${MAX_ORDER_TOTAL.amount} ${MAX_ORDER_TOTAL.currency}`);
    }

    this._lineItems.push(item);
  }

  updateLineItemQuantity(lineItemId: string, newQuantity: number): void {
    this.assertDraft();
    const item = this._lineItems.find(li => li.id === lineItemId);
    if (!item) throw new Error(`Line item ${lineItemId} not found`);

    // Validate projected total BEFORE mutating — sum all other items + new subtotal
    const otherItemsTotal = this._lineItems
      .filter(li => li.id !== lineItemId)
      .reduce((sum, li) => sum.add(li.subtotal), Money.zero('EUR'));
    const newSubtotal = item.unitPrice.multiply(newQuantity);
    const projectedTotal = otherItemsTotal.add(newSubtotal);

    if (projectedTotal.amount > MAX_ORDER_TOTAL.amount) {
      throw new Error('Update would exceed maximum order total');
    }

    item.updateQuantity(newQuantity); // only mutate after validation passes
  }

  removeLineItem(lineItemId: string): void {
    this.assertDraft();
    this._lineItems = this._lineItems.filter(li => li.id !== lineItemId);
  }

  place(): void {
    this.assertDraft();
    if (this._lineItems.length === 0) throw new Error('Cannot place an empty order');
    this._status = 'placed';

    // Emit domain event
    this._domainEvents.push({
      eventType: 'OrderPlaced',
      occurredAt: new Date(),
      aggregateId: this.id,
      customerId: this.customerId,
      totalAmount: this.total.amount,
      totalCurrency: this.total.currency,
      lineItemCount: this._lineItems.length,
    } as OrderPlacedEvent);
  }

  // Called by the repository/event dispatcher after saving
  pullDomainEvents(): DomainEvent[] {
    const events = [...this._domainEvents];
    this._domainEvents = [];
    return events;
  }

  private assertDraft(): void {
    if (this._status !== 'draft') {
      throw new Error(`Cannot modify order in '${this._status}' status`);
    }
  }
}
```

**Why external code should never modify child entities directly:**

1. **Invariant bypass**: If external code could call `lineItem.updateQuantity(999)` directly, it skips the aggregate's total validation. The order could end up with a total exceeding the limit — the invariant is broken.
2. **Consistency boundary**: The aggregate root is the consistency boundary (as discussed in question 7). All changes flow through it so it can enforce cross-entity rules (like "total must stay under 10,000 EUR" which depends on *all* line items).
3. **Event consistency**: Domain events should only be emitted when the aggregate reaches a valid state. If external code mutates child entities, the aggregate doesn't know something changed and can't emit the appropriate events.

**Enforcement in TypeScript**: TypeScript doesn't have package-private visibility, so the main tools are:
- Return `ReadonlyArray` or spread copies from getters so external code can't mutate the internal collection
- Document that child entity mutation methods are internal (convention)
- The aggregate root is the only object the repository loads and saves — external code never gets a mutable reference to a child entity without going through the root

</details>

<details>
<summary>17. Show how you would set up a modular monolith in TypeScript — demonstrate explicit module boundaries (what each module exposes vs hides), the internal API surface between modules, and the rules you enforce (no direct database access across modules, no importing internal types). Explain how this structure makes future extraction to microservices straightforward.</summary>

Building on the concepts from question 11, here is a concrete implementation.

**Directory structure:**

```
src/
  modules/
    orders/
      index.ts              # PUBLIC API — re-exports types + service
      types.ts              # Public types (DTOs, event shapes)
      internal/
        entities/
          order.ts
        services/
          order-service.ts
        repositories/
          order-repository.ts
          postgres-order-repository.ts
        events/
          order-events.ts
    inventory/
      index.ts              # PUBLIC API
      internal/
        entities/
          stock-item.ts
        services/
          inventory-service.ts
        repositories/
          ...
    shared/
      event-bus.ts           # In-process event bus
      types.ts               # Shared primitive types (IDs, pagination)
  app.ts                     # Composition root
```

**Module public types** — defined in a separate file to avoid circular imports:

```typescript
// modules/orders/types.ts
export interface OrderDTO {
  id: string;
  customerId: string;
  status: string;
  total: number;
  lineItems: Array<{ productId: string; quantity: number }>;
}

export interface OrderEvents {
  OrderPlaced: { orderId: string; items: Array<{ productId: string; quantity: number }> };
  OrderCancelled: { orderId: string; items: Array<{ productId: string; quantity: number }> };
}
```

**Module public API** — re-exports types and exposes the module's service:

```typescript
// modules/orders/index.ts
export { OrderDTO, OrderEvents } from './types';
export { OrderModuleAPI, createOrderModule } from './internal/services/order-service';
```

Note: `Order` entity, repository interfaces, and internal implementation details are NOT exported.

**Module service** — imports types from `./types`, not from the module's own `index.ts`:

```typescript
// modules/orders/internal/services/order-service.ts
import { Order } from '../entities/order';
import { OrderRepository } from '../repositories/order-repository';
import { EventBus } from '../../../shared/event-bus';
import { OrderDTO, OrderEvents } from '../../types';

export interface OrderModuleAPI {
  createOrder(customerId: string, items: Array<{ productId: string; quantity: number; price: number }>): Promise<OrderDTO>;
  getOrder(orderId: string): Promise<OrderDTO | null>;
  cancelOrder(orderId: string): Promise<void>;
}

export function createOrderModule(deps: {
  orderRepo: OrderRepository;
  eventBus: EventBus;
}): OrderModuleAPI {
  return {
    async createOrder(customerId, items) {
      const order = Order.create(customerId);
      for (const item of items) {
        order.addLineItem(item.productId, item.quantity, item.price);
      }
      order.place();
      await deps.orderRepo.save(order);

      // Publish event — other modules react asynchronously
      deps.eventBus.publish<OrderEvents['OrderPlaced']>('OrderPlaced', {
        orderId: order.id,
        items: items.map(i => ({ productId: i.productId, quantity: i.quantity })),
      });

      return toDTO(order);
    },

    async getOrder(orderId) {
      const order = await deps.orderRepo.findById(orderId);
      return order ? toDTO(order) : null;
    },

    async cancelOrder(orderId) {
      const order = await deps.orderRepo.findById(orderId);
      if (!order) throw new Error('Order not found');
      const items = order.lineItems;
      order.cancel();
      await deps.orderRepo.save(order);
      deps.eventBus.publish<OrderEvents['OrderCancelled']>('OrderCancelled', {
        orderId, items: items.map(i => ({ productId: i.productId, quantity: i.quantity })),
      });
    },
  };
}

function toDTO(order: Order): OrderDTO { /* map internal entity to public DTO */ }
```

**Inter-module communication** — inventory reacts to order events:

```typescript
// modules/inventory/internal/services/inventory-service.ts
import { EventBus } from '../../../shared/event-bus';
import { OrderEvents } from '../../orders'; // imports only public types

export function createInventoryModule(deps: { stockRepo: StockRepository; eventBus: EventBus }) {
  // Subscribe to order events
  deps.eventBus.subscribe<OrderEvents['OrderPlaced']>('OrderPlaced', async (event) => {
    for (const item of event.items) {
      await deps.stockRepo.reserve(item.productId, item.quantity);
    }
  });

  deps.eventBus.subscribe<OrderEvents['OrderCancelled']>('OrderCancelled', async (event) => {
    for (const item of event.items) {
      await deps.stockRepo.release(item.productId, item.quantity);
    }
  });

  return { /* inventory module's public API */ };
}
```

**Shared event bus** — simple in-process implementation:

```typescript
// shared/event-bus.ts
type Handler<T = any> = (event: T) => Promise<void>;

export class EventBus {
  private handlers = new Map<string, Handler[]>();

  subscribe<T>(eventType: string, handler: Handler<T>): void {
    const existing = this.handlers.get(eventType) ?? [];
    this.handlers.set(eventType, [...existing, handler]);
  }

  async publish<T>(eventType: string, event: T): Promise<void> {
    const handlers = this.handlers.get(eventType) ?? [];
    await Promise.all(handlers.map(h => h(event)));
  }
}
```

**Rules enforced:**

1. **No cross-module database access**: Each module has its own repository that queries only its own tables. Inventory never runs `SELECT * FROM orders`. Enforced by giving each module its own database pool/schema, or at minimum by code review and linting.
2. **No importing internal types**: ESLint rule prevents importing from `modules/orders/internal/*`:
   ```json
   {
     "import/no-restricted-paths": ["error", {
       "zones": [{
         "target": "./src/modules/inventory",
         "from": "./src/modules/orders/internal"
       }]
     }]
   }
   ```
3. **Communication through public APIs or events**: Modules call each other's exported functions for synchronous queries. Side effects use events to avoid coupling.

**Extraction path to microservices:**

The extraction path works as described in Q11 — the module's public API becomes the service API, in-process events become queue messages. The calling code in the inventory module doesn't change — it still calls the same interface. Only the wiring in the composition root changes: instead of `createOrderModule()`, it creates an `HttpOrderClient` that implements `OrderModuleAPI` over the network.

</details>

---

## Experience-Based Questions

These questions test real-world experience. Prepare by mapping them to your own projects and situations.

<details>
<summary>18. Tell me about a time you chose or changed the architectural style for a project — what were the constraints (team size, complexity, timeline), what did you choose and why, and what would you do differently with hindsight?</summary>

**What the interviewer is looking for:**
- Ability to evaluate architectural tradeoffs against real constraints (not just textbook knowledge)
- Awareness that architecture is a decision with consequences, not a "correct answer"
- Intellectual honesty — willingness to admit what didn't work
- Pragmatism over purism

**Suggested structure (STAR-ish):**

1. **Context**: What was the project? What stage was it at? What were the concrete constraints (team size, timeline, domain complexity, existing codebase)?
2. **Decision**: What architectural style did you choose (or change to)? What alternatives did you consider?
3. **Reasoning**: Why did you choose this over alternatives? What tradeoffs did you explicitly accept?
4. **Outcome**: How did it work out? What was the impact on development velocity, maintainability, or team productivity?
5. **Hindsight**: What would you do differently? What did you learn about matching architecture to constraints?

**Key points to hit:**
- Show that constraints drove the decision, not dogma ("We chose a simple layered approach because we had 3 months and 2 engineers, not because it's the 'best' architecture")
- Mention what you explicitly chose NOT to do and why
- Connect the decision to measurable outcomes (faster onboarding, easier testing, reduced deployment risk)
- The hindsight section is the most important — it shows growth. Be specific: "I would have introduced module boundaries earlier because we hit coupling problems at month 6"

**Example outline to personalize:**
- "We were building [service X] with a 3-person team. The domain had moderate complexity but heavy integration requirements (5+ external APIs). We started with a simple layered architecture for speed, but after 6 months the service layer became a mess — business logic was scattered across controllers and repository queries. I proposed migrating to hexagonal architecture incrementally: we introduced port interfaces for external integrations first (highest pain), then gradually moved business logic into use cases. The migration took ~3 sprints of incremental work alongside features. In hindsight, I would have started with ports/adapters for the external integrations from day one — the layered approach was fine for the domain logic but the integration coupling was predictable and avoidable."

</details>

<details>
<summary>19. Tell me about a time you dealt with a poorly architected codebase — what were the symptoms, how did it impact the team's velocity, and what steps did you take to improve it without stopping feature work?</summary>

**What the interviewer is looking for:**
- Ability to diagnose architectural problems from symptoms (not just "the code was messy")
- Pragmatism — improving incrementally while shipping, not proposing a rewrite
- Technical leadership — convincing the team, getting buy-in, driving change
- Prioritization — knowing which problems to fix first for maximum impact

**Suggested structure:**

1. **Symptoms**: What concrete problems did you observe? (Tie to the patterns from question 1 — coupling, circular deps, untestable code, slow PRs, frequent regressions)
2. **Impact**: How did this affect the team quantifiably? (PR cycle time, bug rate, onboarding time, deploy frequency)
3. **Diagnosis**: What was the root cause? Not just "bad code" — what structural decisions (or lack thereof) led to this?
4. **Strategy**: What was your approach to improvement? How did you balance it with feature work?
5. **Execution**: What specific changes did you make, in what order, and why that order?
6. **Result**: What improved? How did you measure it?

**Key points to hit:**
- Name specific symptoms, not vague complaints: "Every PR touched 8+ files because our services had circular dependencies" not "the code was hard to work with"
- Show the incremental approach: "We didn't stop features. Instead, we adopted a 'boy scout rule' — every PR improves one small thing. Plus, we dedicated 20% of each sprint to targeted refactoring"
- Explain prioritization: "We started with X because it was the highest-leverage change — it unblocked Y and Z improvements"
- Common strategies to mention: introducing interfaces at key boundaries, breaking circular dependencies, extracting shared logic into modules, adding linting rules to prevent regression, writing tests before refactoring

**Example outline to personalize:**
- "Our audit-log service had grown organically — no module boundaries, shared database queries everywhere, and adding a new event type required changes in 4 files across 3 directories. PRs took 3+ days to review because changes cascaded unpredictably. I proposed an incremental plan: (1) First, we extracted a clear module boundary around the event processing pipeline — this was the most painful coupling point. (2) We added ESLint import rules so new code couldn't re-introduce cross-module imports. (3) Each sprint, we'd pick one legacy coupling point and refactor it as part of a related feature PR. After 2 months, new event types required changes in 1-2 files, PR cycle time dropped from 3 days to under 1 day, and the team could work on separate features in parallel without merge conflicts."

</details>

<details>
<summary>20. Describe a time you applied DDD concepts (aggregates, bounded contexts, domain events) to structure a complex domain — how did you identify the domain boundaries, what modeling decisions were hardest, and how did the architecture hold up as requirements evolved?</summary>

**What the interviewer is looking for:**
- Practical understanding of DDD — not textbook definitions, but real application with real tradeoffs
- Ability to identify domain boundaries from business conversations, not just code
- Awareness that modeling is iterative — early decisions get refined as understanding deepens
- Honest assessment of what worked and what didn't

**Suggested structure:**

1. **Domain context**: What was the business domain? Why was it complex enough to warrant DDD concepts?
2. **Boundary identification**: How did you discover the bounded contexts? What conversations, workshops, or analysis led to the boundaries?
3. **Modeling decisions**: What aggregates did you define? What were the hardest decisions around aggregate boundaries?
4. **Events**: How did you use domain events for cross-boundary communication? What tradeoffs did eventual consistency introduce?
5. **Evolution**: How did the model change as requirements evolved? What held up and what didn't?
6. **Lessons**: What would you do differently?

**Key points to hit:**
- Show that boundaries came from the domain, not from technical convenience: "We identified three bounded contexts by mapping the language — the operations team, sales team, and finance team all used 'contract' to mean different things"
- Discuss aggregate sizing tradeoffs: "We initially made the aggregate too large — it included X and Y — which caused locking issues. We split it into two aggregates connected by events, accepting eventual consistency between them"
- Mention concrete domain events and what they enabled: "The `ContractSigned` event let the finance context create an invoice independently, without the sales context knowing about billing"
- Be honest about what was hard: aggregate boundaries are never obvious upfront, and getting them wrong has real cost

**Example outline to personalize:**
- "In our platform, we were building change tracking for merchant resources. Initially, we modeled everything in one service — changes, audit trails, and notifications were all tangled together. We realized through team discussions that 'change' meant different things: the ingestion pipeline cared about raw change events, the audit service cared about who-changed-what, and the notification service cared about what-should-trigger-an-alert. We separated these into bounded contexts with domain events connecting them. The hardest modeling decision was where the aggregate boundary fell for change events — should a batch of changes be one aggregate or individual changes? We chose individual changes as aggregates (one transaction each) with batch processing handled by a domain service. This held up well when we added new resource types — each was just a new event type flowing through the same boundaries. What didn't hold up: we underestimated cross-context query needs. Reports that needed data from multiple contexts required a separate read model, which we hadn't planned for initially."

</details>
