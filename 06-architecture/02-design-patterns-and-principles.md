# Design Patterns & Principles

> **40 questions** — 27 theory, 13 practical

- Design patterns vs design principles: what each solves, how principles guide pattern selection, why patterns without principles lead to over-engineering
- SOLID principles: SRP, OCP, LSP, ISP, DIP — what each solves, violation examples, conflicts between them, and when rigid adherence hurts readability or adds unnecessary indirection
- DRY, KISS, YAGNI: when they conflict and how to decide which wins
- Singleton pattern: problems with testing, concurrency, hidden dependencies, DI alternatives
- Factory pattern: simple factory, Factory Method, when to use vs direct instantiation or object literals — and when builders are warranted
- Builder pattern: complex object construction, fluent APIs, test fixture builders — when Builder adds value vs factory functions or object literals
- Strategy and State patterns: identical structure, different intent, real-world signals
- Observer pattern: EventEmitter, DOM events, pub/sub, pitfalls of overuse
- Decorator and Adapter patterns: adding behavior vs changing interface, TypeScript decorators
- Proxy pattern: lazy loading, caching, access control — relationship to JS Proxy, distinction from Adapter and Decorator
- Facade pattern: managing complexity, API design, crossing into god object antipattern
- Command pattern: encapsulating requests as objects, undo/redo, task queuing, relationship to CQRS command handling
- Template Method pattern: defining algorithm skeleton with overridable steps, framework hooks (NestJS lifecycle), when inheritance-based extension is appropriate vs Strategy
- Chain of Responsibility: Express/NestJS middleware, request processing decoupling
- Inversion of Control and Dependency Injection: IoC containers, constructor vs property injection, service locator antipattern — pattern mechanics and testability benefits
- Repository pattern: separating business logic from data access, ORM abstraction tradeoffs
- Composition over inheritance: mixins, interfaces, when inheritance is still right
- Code smells: feature envy, shotgun surgery, primitive obsession, long parameter lists — recognizing smells, choosing the right refactoring, key techniques (extract method/class, move behavior, introduce value objects)
- Cost of abstraction: cognitive overhead, debugging difficulty, when abstractions hurt
- Common anti-patterns: god object, golden hammer, cargo cult, premature abstraction — recognizing and refactoring them
- Pattern selection: recognizing signals — growing switch statements (Strategy), repeated object setup (Builder/Factory), cross-cutting concerns (Decorator/Chain of Responsibility), notification/event propagation (Observer), complex conditional workflows (State)

---

## Foundational

<details>
<summary>1. What is the difference between a design pattern and a design principle — what problem does each solve, how do principles guide which patterns you select, and why does applying patterns without understanding the underlying principles lead to over-engineering?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>2. What are the five SOLID principles (SRP, OCP, LSP, ISP, DIP), what specific design problem does each one prevent, and how do they work together as a system — why does violating one principle often cause violations of the others?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>3. Why does "composition over inheritance" exist as a design guideline — what problems does deep inheritance create that composition solves, how do mixins and interfaces provide composition mechanisms in TypeScript, and when is inheritance still the right choice?</summary>

<!-- Answer will be added later -->

</details>

## Conceptual Depth

<details>
<summary>4. When do SOLID principles conflict with each other — for example, when does following SRP push you to violate KISS by creating too many small classes, or when does strict OCP add unnecessary indirection? How do you decide which principle wins in practice, and what are the signs that rigid SOLID adherence is hurting your codebase more than helping it?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>5. When do DRY, KISS, and YAGNI conflict with each other — what does it look like when removing duplication (DRY) creates a complex shared abstraction that violates KISS, or when building for extensibility violates YAGNI? How do you decide which principle takes priority in a given situation?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>6. Why is the Singleton pattern considered problematic despite being one of the most well-known patterns — what specific problems does it cause for testing, concurrency, and hidden dependency management, and what alternatives (dependency injection, module-scoped instances) solve the same "single instance" need without Singleton's downsides?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>7. What problem does the Factory pattern solve that direct instantiation and object literals don't — what is the difference between a simple factory and the Factory Method pattern, when does a Factory add real value vs unnecessary indirection, and at what point does object creation complexity warrant a Builder pattern instead?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>8. When does the Builder pattern add real value over factory functions or object literals for complex object construction -- how do fluent APIs make Builders ergonomic, why are test fixture builders (e.g., building complex test data with sensible defaults and selective overrides) one of the most practical uses of Builder, and when does Builder become unnecessary indirection?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>9. Why do the Strategy and State patterns have identical class structures but different intents — what is the core difference in what they model, what real-world signals tell you "this is a Strategy problem" vs "this is a State problem," and what goes wrong when you use the wrong one?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>10. What problem does the Observer pattern solve, how do Node.js EventEmitter and DOM events implement it, and what are the pitfalls of overusing the Observer pattern — how does it create implicit coupling, make debugging harder, and lead to memory leaks when listeners aren't properly cleaned up?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>11. How does pub/sub differ from the basic Observer pattern — when does decoupling the observer from the subject through a message broker fundamentally change the architecture, what tradeoffs does that indirection introduce, and when is direct observer registration (EventEmitter) better than a pub/sub system?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>12. What is the difference between the Decorator pattern and the Adapter pattern — why does one add behavior while the other changes an interface, how do TypeScript decorators relate to the GoF Decorator pattern, and when would you use each in a real codebase?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>13. What problem does the Proxy pattern solve -- how does it differ from the Adapter and Decorator patterns despite all three wrapping another object, what are the common use cases (lazy loading, caching, access control), how does JavaScript's built-in Proxy relate to the GoF Proxy pattern, and when would you reach for a Proxy over a Decorator in a real codebase?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>14. What problem does the Facade pattern solve, how does it apply to API design and module boundaries, and when does a Facade cross the line into a god object — what signals tell you that your "simplified interface" has become a monolithic bottleneck that violates SRP?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>15. How does the Chain of Responsibility pattern decouple request processing — why do Express and NestJS use middleware chains instead of a single handler, what makes this pattern powerful for cross-cutting concerns, and when does a long chain become a debugging nightmare?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>16. What problem does the Command pattern solve — why does encapsulating a request as an object (with execute/undo methods) enable undo/redo, task queuing, and macro recording, how does Command relate to CQRS command handling, and when is Command overkill compared to a simple function call?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>17. What problem does the Template Method pattern solve — why does defining an algorithm skeleton in a base class with overridable steps work well for framework hooks (e.g., NestJS lifecycle methods), how does it differ from Strategy (inheritance-based vs composition-based variation), and when does Template Method's reliance on inheritance become a liability?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>18. What is Inversion of Control and why does it matter -- how does IoC differ from dependency injection (DI is one form of IoC), what role do IoC containers play (NestJS's DI container, for example), why does inverting control make systems more modular and extensible, and why is the Service Locator considered an antipattern despite also implementing IoC -- what problems does it cause compared to constructor injection?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>19. What are the tradeoffs between constructor injection and property injection for dependency injection — why is constructor injection generally preferred, when does property injection make sense (e.g., optional dependencies), and how does the Dependency Inversion Principle specifically enable testability and plugin architectures?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>20. Why does the Repository pattern exist — what problems does it solve by putting an abstraction between business logic and data access, what are the tradeoffs of using a repository over calling an ORM directly, and when does the repository abstraction become a leaky abstraction that adds cost without value?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>21. What are the code smells 'feature envy' and 'shotgun surgery' -- what design problem does each one indicate, what signals help you recognize them during code review, and why are they symptoms of deeper structural issues (wrong responsibility boundaries) rather than just style problems?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>22. What are the code smells 'primitive obsession' and 'long parameter lists' -- what design problem does each one indicate, how do you recognize them in a codebase, and what refactorings (value objects, parameter objects) address the root cause rather than just the symptom?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>23. How do code smells connect to specific pattern solutions and refactoring techniques — for example, why does a growing switch statement signal the need for Strategy, why does feature envy suggest extract method/move behavior to the class it envies, and why does primitive obsession point toward extract class/value objects? What is the process for mapping a smell to the right refactoring, and when should you NOT refactor (the code works, is rarely changed, or the abstraction cost exceeds the benefit)?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>24. What is the cost of abstraction — why do abstractions add cognitive overhead and debugging difficulty, how do you recognize when an abstraction is making code harder to understand rather than easier, and what principles help you decide when to abstract vs when to keep things concrete?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>25. What are the anti-patterns 'god object' and 'premature abstraction' -- what causes each one to emerge in a codebase, how do you recognize them, and what is the path to refactoring each one incrementally without rewriting everything?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>26. What are the anti-patterns 'golden hammer' and 'cargo cult programming' -- what causes each one, how do you recognize them in team behavior and technical decisions, and how do you course-correct when you spot them?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>27. How do you recognize which design pattern a problem calls for — what signals in code indicate Strategy (growing conditionals), Builder/Factory (repeated complex setup), Decorator/Chain of Responsibility (cross-cutting concerns), or Observer (event-driven communication)? What is the risk of pattern-hunting — looking for problems to fit patterns you already know?</summary>

<!-- Answer will be added later -->

</details>

## Practical — Implementation & Refactoring

<details>
<summary>28. Given a function with a growing switch statement or if-else chain that selects behavior based on a type (e.g., calculating shipping cost by carrier), refactor it to use the Strategy pattern in TypeScript — show the interface, concrete strategies, and the context class, and explain why this refactoring makes the code easier to extend without modifying existing logic (OCP)</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>29. Implement a Factory pattern in TypeScript for creating different types of database connections (e.g., PostgreSQL, Redis, SQLite) — show both a simple factory function and a Factory Method approach, explain when you'd use each vs just passing config objects or using object literals, and show when the creation logic gets complex enough to warrant a Builder instead</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>30. Implement the Decorator pattern in TypeScript to add logging, caching, and retry behavior to a service class without modifying the original — show both the classical wrapper approach and TypeScript's decorator syntax (using experimental decorators or the TC39 proposal), and explain when each approach is more appropriate</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>31. Implement an Observer pattern using Node.js EventEmitter for a domain event system (e.g., order placed triggers inventory update, email notification, and analytics) — show the emitter setup, listener registration, and proper cleanup to avoid memory leaks, and explain what goes wrong when listeners accumulate without being removed</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>32. Implement a Repository pattern in TypeScript that wraps a database client (e.g., Prisma or a raw SQL client) — show the repository interface, a concrete implementation, and how a service class depends on the interface rather than the implementation. Demonstrate how this enables swapping the data source in tests without touching business logic</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>33. Implement a Chain of Responsibility pipeline in TypeScript inspired by Express/NestJS middleware — show how each handler in the chain decides whether to process the request or pass it to the next handler, demonstrate adding cross-cutting concerns (authentication, validation, logging), and explain how this pattern decouples request processing from routing</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>34. Given a tightly coupled TypeScript class that directly instantiates its dependencies (e.g., a UserService that creates its own database connection and email client inside the constructor), refactor it to use Dependency Inversion — show the before and after code, extract interfaces, inject dependencies through the constructor, and demonstrate how the refactored version is testable with mock implementations</summary>

<!-- Answer will be added later -->

</details>

## Practical — Smell Detection & Refactoring

<details>
<summary>35. Given a TypeScript class that exhibits multiple code smells (feature envy — methods that mostly use data from another class; primitive obsession — using raw strings for emails, currencies, or IDs; and a long parameter list in a constructor), identify each smell, explain what structural problem it indicates, and show the step-by-step refactoring for each — creating value objects, moving behavior to the right class, and introducing a parameter object or builder</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>36. Given a god object — a single TypeScript class that handles user authentication, profile management, email sending, and audit logging — identify why it became a god object, explain which SOLID principles it violates, and show how to decompose it into focused classes using the right patterns (Facade for the simple API, Strategy or separate services for each responsibility), while preserving the existing public interface so callers don't break</summary>

<!-- Answer will be added later -->

</details>

---

## Experience-Based Questions

These questions test real-world experience. Prepare by mapping them to your own projects and situations.

<details>
<summary>37. Tell me about a time you introduced a design pattern to solve a real problem in a codebase — what was the problem, which pattern did you choose and why, how did you convince the team it was the right approach, and what was the measurable impact on code quality or developer productivity?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>38. Describe a time you over-engineered a solution or applied the wrong design pattern — what led to that decision, how did you realize the abstraction was hurting more than helping, and what did you do to simplify or reverse course?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>39. Tell me about a time you refactored a significant portion of a codebase to improve its architecture — what was the original state (code smells, anti-patterns), how did you plan and execute the refactoring without breaking production, and how did you measure success?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>40. Describe a situation where you had to balance code quality and design patterns against delivery speed — how did you decide what to implement properly vs what to ship as technical debt, and how did you ensure the debt was eventually paid down?</summary>

<!-- Answer framework will be added later -->

</details>
