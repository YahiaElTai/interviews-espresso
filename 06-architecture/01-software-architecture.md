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
<summary>7. Explain the core DDD building blocks — what are entities, value objects, and aggregates, what rules govern aggregate boundaries (consistency boundary, single transaction, reference by ID), and how do domain events allow aggregates to communicate without coupling?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>8. What are bounded contexts in DDD — how do you identify them in a complex domain, how do they map to codebases and teams, and why does getting boundaries wrong lead to distributed monolith problems?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>9. What is the repository pattern — why does it define an interface in the domain layer but implement it in the infrastructure layer, how does this apply dependency inversion, and what problems emerge when repositories leak infrastructure concerns (query builders, ORM types) into the domain?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>10. Why is "monolith first" considered the pragmatic default — what goes wrong when teams start with microservices before understanding their domain boundaries, what does a well-structured monolith look like, and how does it differ from a "big ball of mud"?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>11. What is a modular monolith — how do you enforce explicit module boundaries and internal APIs within a single deployable, what rules prevent modules from becoming coupled, and how does this architecture create a clean extraction path to microservices when the time comes?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>12. What does pragmatic architecture mean in practice — how do you right-size architecture decisions for your team's size and codebase complexity, what are the concrete signs of over-engineering (premature abstraction, unnecessary indirection, pattern stuffing), and when is "good enough" the right call?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>13. Why should testability be treated as an architectural driver rather than an afterthought — how does dependency inversion enable test doubles at layer boundaries, what is the practical difference between testing pure domain logic and infrastructure-dependent code, and how do you choose which architectural boundaries become test boundaries?</summary>

<!-- Answer will be added later -->

</details>

## Practical — Implementation & Refactoring

<details>
<summary>14. Show how you would structure a TypeScript/Node.js project using clean or hexagonal architecture — lay out the directory structure, show a concrete example of an entity, a use case, a port (interface), and an adapter (implementation), and explain how the dependency rule is enforced at the code level so that inner layers never import from outer layers.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>15. Implement the repository pattern in TypeScript with dependency inversion — show the domain-layer interface, an infrastructure-layer implementation (e.g., using Prisma or a plain SQL client), and demonstrate how the application service depends only on the interface. Explain how this makes the service testable with a fake/mock repository.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>16. Define an aggregate in TypeScript with proper boundary enforcement — show an aggregate root with child entities/value objects, enforce invariants inside the aggregate (e.g., order line quantity limits, total price validation), emit a domain event, and explain why external code should never modify child entities directly.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>17. Show how you would set up a modular monolith in TypeScript — demonstrate explicit module boundaries (what each module exposes vs hides), the internal API surface between modules, and the rules you enforce (no direct database access across modules, no importing internal types). Explain how this structure makes future extraction to microservices straightforward.</summary>

<!-- Answer will be added later -->

</details>

---

## Experience-Based Questions

These questions test real-world experience. Prepare by mapping them to your own projects and situations.

<details>
<summary>18. Tell me about a time you chose or changed the architectural style for a project — what were the constraints (team size, complexity, timeline), what did you choose and why, and what would you do differently with hindsight?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>19. Tell me about a time you dealt with a poorly architected codebase — what were the symptoms, how did it impact the team's velocity, and what steps did you take to improve it without stopping feature work?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>20. Describe a time you applied DDD concepts (aggregates, bounded contexts, domain events) to structure a complex domain — how did you identify the domain boundaries, what modeling decisions were hardest, and how did the architecture hold up as requirements evolved?</summary>

<!-- Answer framework will be added later -->

</details>
