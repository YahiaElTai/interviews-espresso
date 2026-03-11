# Software Architecture

> **36 questions** — 24 theory, 12 practical

- Problems without architecture: coupling explosion, circular dependencies, untestable code, what breaks as codebases grow
- Coupling and cohesion: types of coupling (data, stamp, control, temporal, platform), cohesion as a design goal, how architecture patterns reduce coupling while increasing cohesion
- Dependency Rule: why dependencies must point inward toward higher-level policy, how dependency inversion (not injection) enforces this at the architectural level, relationship to all layered architecture styles
- Testability as an architectural driver: how dependency inversion enables test doubles, pure domain logic vs infrastructure-dependent code, choosing test boundaries based on architecture layers
- Layered (n-tier) architecture: presentation/business/data layers, strengths (simplicity, familiarity), weaknesses (cross-cutting concerns, rigid boundaries), signals a team has outgrown it
- Clean architecture: concentric layers (entities, use cases, interface adapters, frameworks)
- Hexagonal architecture: ports and adapters, how it differs from clean architecture in practice, repository pattern as a canonical adapter example
- Vertical slice architecture: organizing by feature instead of layer, tradeoffs vs clean architecture
- DDD core: entities, value objects, aggregates, aggregate boundary rules, domain events
- Bounded contexts: identification, mapping to codebases and teams, context mapping, anti-corruption layers, Conway's Law and how org structure shapes boundaries
- CQRS as an architectural choice: when to separate read and write models, complexity cost, how it changes your architecture's data flow — detailed mechanics covered in cloud design patterns
- Application service layer: orchestrating domain objects, repositories, and transactions — what belongs here vs in the domain, Unit of Work pattern for transaction coordination
- Domain Model vs Transaction Script: when rich domain objects with behavior are worth the complexity vs simple procedural services operating on data structures, how this choice shapes the entire architecture
- Twelve-Factor App: key principles that shape backend service design and deployment
- Monolith first: why starting with microservices is premature, well-structured monolith design
- Modular monolith: explicit module boundaries, internal APIs, path to microservice extraction
- Pragmatic architecture: right-sizing complexity to team size and stage, signs of over-engineering (unused abstractions, layers with no logic), architecture as a tool not a goal
- Architecture decision records (ADRs): documenting architectural decisions, lightweight format, why "we considered X but chose Y because Z" matters for team alignment and onboarding
- Incremental migration: deciding what to extract first, identifying seams in a monolith, modular decomposition strategies, when migration isn't worth it

---

## Foundational

<details>
<summary>1. What happens to a codebase that grows without intentional architecture — why do coupling explosion, circular dependencies, and untestable code emerge, what specific symptoms signal each problem, and why does the cost of change grow non-linearly as these problems compound?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>2. What is the Dependency Rule — why must dependencies always point inward toward higher-level policy, how does dependency inversion (as distinct from dependency injection) enforce this rule at the architectural level, and why is this rule the common thread across layered, clean, and hexagonal architectures?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>3. How do layered, clean, hexagonal, and vertical slice architectures relate to each other — what core problem do they all attempt to solve, how do they differ in the way they organize code and enforce boundaries, and why is it important to understand the family rather than just pick one?</summary>

<!-- Answer will be added later -->

</details>

## Conceptual Depth

<details>
<summary>4. What is layered (n-tier) architecture, what are its real tradeoffs (both strengths and weaknesses), and what specific signals tell you a team has outgrown it — why do cross-cutting concerns and rigid layer boundaries become painful as complexity grows?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>5. Explain clean architecture's concentric layers (entities, use cases, interface adapters, frameworks) — what belongs in each layer, why does the dependency direction always point inward, and what goes wrong when teams put business logic in the wrong layer?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>6. What is hexagonal architecture (ports and adapters) — how do ports and adapters work, why does it make infrastructure swappable, and how does it differ from clean architecture in practice despite sharing the same dependency direction principle?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>7. What is vertical slice architecture — why does it organize code by feature instead of by layer, what tradeoffs does it make compared to clean architecture (code duplication vs decoupling), and when does vertical slicing produce better outcomes than horizontal layering?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>8. Explain the core DDD building blocks — what are entities, value objects, and aggregates, what rules govern aggregate boundaries (consistency boundary, single transaction, reference by ID), and how do domain events allow aggregates to communicate without coupling?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>9. What are bounded contexts in DDD — how do you identify them in a complex domain, how do they map to codebases and teams, and why does getting boundaries wrong lead to distributed monolith problems?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>10. What is context mapping and why do you need it when bounded contexts interact — explain the key relationship types (shared kernel, customer-supplier, conformist, anti-corruption layer), when you would choose each, and why anti-corruption layers are critical at boundaries where models diverge?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>11. What is CQRS — why would you separate read and write models, when is the added complexity justified vs when is it over-engineering, and what is CQRS's relationship to event sourcing (why are they often paired but not the same thing)?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>12. What is the repository pattern — why does it define an interface in the domain layer but implement it in the infrastructure layer, how does this apply dependency inversion, and what problems emerge when repositories leak infrastructure concerns (query builders, ORM types) into the domain?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>13. What belongs in the application service layer vs the domain layer — how does an application service orchestrate domain objects, repositories, and transactions (Unit of Work) without containing business logic itself, and what are the symptoms that business logic has leaked into the service layer?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>14. What is the difference between the Domain Model pattern and the Transaction Script pattern — when is the complexity of rich domain objects with encapsulated behavior justified over simpler procedural services that operate on plain data structures, how does this choice ripple through the rest of your architecture (testing, team onboarding, refactoring cost), and what signals tell you a Transaction Script approach has outgrown its usefulness?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>15. Which Twelve-Factor App principles cause the most operational pain when violated in backend services — pick the 3-4 that matter most in your experience (e.g., config, backing services, disposability, dev/prod parity), explain why each matters, and describe what goes wrong in production when teams ignore them?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>16. Why is "monolith first" considered the pragmatic default — what goes wrong when teams start with microservices before understanding their domain boundaries, what does a well-structured monolith look like, and how does it differ from a "big ball of mud"?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>17. What is a modular monolith — how do you enforce explicit module boundaries and internal APIs within a single deployable, what rules prevent modules from becoming coupled, and how does this architecture create a clean extraction path to microservices when the time comes?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>18. How does Conway's Law influence software architecture — why do system boundaries tend to mirror organizational structure, how does this affect bounded context boundaries in practice, and should you fight Conway's Law or use it to your advantage (inverse Conway maneuver)?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>19. What does pragmatic architecture mean in practice — how do you right-size architecture decisions for your team's size and codebase complexity, what are the concrete signs of over-engineering (premature abstraction, unnecessary indirection, pattern stuffing), and when is "good enough" the right call?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>20. How do you incrementally migrate a monolith — explain the strangler fig pattern, how you identify seams and decide what to extract first, and what modular decomposition strategies work in practice vs look good only on paper?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>21. What are the hardest parts of monolith-to-microservice migration that teams consistently underestimate — explain how shared databases, distributed transactions, and data consistency become problems, and when is migration simply not worth the cost?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>22. What are the different types of coupling (data, stamp, control, temporal, platform) and why does understanding them matter for architecture decisions — how does high cohesion relate to low coupling, and how do architecture patterns like clean architecture and hexagonal architecture systematically reduce coupling while increasing cohesion?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>23. Why should testability be treated as an architectural driver rather than an afterthought — how does dependency inversion enable test doubles at layer boundaries, what is the practical difference between testing pure domain logic and infrastructure-dependent code, and how do you choose which architectural boundaries become test boundaries?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>24. What are Architecture Decision Records (ADRs), why is documenting "we considered X but chose Y because Z" more valuable than just documenting the final choice, and how do ADRs improve team alignment and onboarding — what lightweight format works in practice and when do teams stop maintaining them?</summary>

<!-- Answer will be added later -->

</details>

## Practical — Implementation & Refactoring

<details>
<summary>25. Show how you would structure a TypeScript/Node.js project using clean or hexagonal architecture — lay out the directory structure, show a concrete example of an entity, a use case, a port (interface), and an adapter (implementation), and explain how the dependency rule is enforced at the code level so that inner layers never import from outer layers.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>26. Implement the repository pattern in TypeScript with dependency inversion — show the domain-layer interface, an infrastructure-layer implementation (e.g., using Prisma or a plain SQL client), and demonstrate how the application service depends only on the interface. Explain how this makes the service testable with a fake/mock repository.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>27. Implement an application service in TypeScript that orchestrates domain objects, a repository, and a transaction — show a concrete use case (e.g., placing an order), demonstrate how the service coordinates without containing business logic, and explain where you draw the line between orchestration and domain logic.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>28. Define an aggregate in TypeScript with proper boundary enforcement — show an aggregate root with child entities/value objects, enforce invariants inside the aggregate (e.g., order line quantity limits, total price validation), emit a domain event, and explain why external code should never modify child entities directly.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>29. Show how you would set up a modular monolith in TypeScript — demonstrate explicit module boundaries (what each module exposes vs hides), the internal API surface between modules, and the rules you enforce (no direct database access across modules, no importing internal types). Explain how this structure makes future extraction to microservices straightforward.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>30. Implement an anti-corruption layer (ACL) between two bounded contexts in TypeScript — show a scenario where one context needs data from another that uses a different model, demonstrate the translation layer that prevents the foreign model from leaking in, and explain when an ACL is worth the extra code vs when a shared kernel is acceptable.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>31. You inherit a tangled codebase where controllers contain business logic, database queries are scattered everywhere, and there are circular dependencies between modules — walk through a concrete refactoring strategy to introduce layered architecture incrementally, show before/after code for one slice, and explain how you avoid a risky big-bang rewrite.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>32. Implement CQRS with separate read and write models in TypeScript — show a write model that enforces domain invariants and persists through a repository, a read model optimized for query performance (denormalized), and the mechanism that keeps them in sync. Explain when this separation pays off and when a single model is sufficient.</summary>

<!-- Answer will be added later -->

</details>

---

## Experience-Based Questions

These questions test real-world experience. Prepare by mapping them to your own projects and situations.

<details>
<summary>33. Tell me about a time you chose or changed the architectural style for a project — what were the constraints (team size, complexity, timeline), what did you choose and why, and what would you do differently with hindsight?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>34. Describe a time you migrated a system from a monolith to microservices or a modular monolith — what triggered the migration, how did you decide where to draw boundaries, what went wrong during the process, and what did you learn?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>35. Tell me about a time you dealt with a poorly architected codebase — what were the symptoms, how did it impact the team's velocity, and what steps did you take to improve it without stopping feature work?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>36. Describe a time you applied DDD concepts (aggregates, bounded contexts, domain events) to structure a complex domain — how did you identify the domain boundaries, what modeling decisions were hardest, and how did the architecture hold up as requirements evolved?</summary>

<!-- Answer framework will be added later -->

</details>
