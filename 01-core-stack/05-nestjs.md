# NestJS

> **25 questions** — 8 theory, 13 practical, 4 experience

- Core building blocks: modules, controllers, providers, and enforced structure
- Dependency injection: DI containers, provider scopes (DEFAULT, REQUEST, TRANSIENT), custom providers (useClass, useFactory, useValue, useExisting)
- NestJS vs Express/Fastify: when the framework overhead is worth it, Fastify adapter tradeoffs
- Request lifecycle: middleware → guards → interceptors → pipes → handler → exception filters
- Exception filters: built-in HttpException hierarchy, custom exception filters for domain errors, global vs controller-scoped filters, consistent error response formatting
- Module encapsulation: exports, dynamic modules (forRoot/forRootAsync), circular dependencies
- Guards: canActivate lifecycle, Passport JWT authentication, role-based authorization with custom decorators
- Pipes: validation with class-validator/class-transformer, built-in transform pipes, global pipe configuration
- Interceptors: Observable-based pipeline (tap/map), response transformation, timeout handling, caching with @nestjs/cache-manager
- Configuration management: @nestjs/config with validation and DI integration
- Structuring NestJS for clean architecture: mapping modules to domain boundaries, keeping framework at the edges
- Lifecycle hooks: onModuleInit, onModuleDestroy, enableShutdownHooks, graceful shutdown with K8s preStop
- Testing: Test.createTestingModule, mocking with overrideProvider, integration testing with supertest

---

## Foundational

<details>
<summary>1. What are the core building blocks of NestJS (modules, controllers, providers) and why does it enforce this structure — what problems does opinionated structure solve compared to Express's flexibility, and how do these pieces compose to form a complete application?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>2. How does the NestJS request lifecycle work — walk through the full pipeline (middleware → guards → interceptors → pipes → handler → interceptors → exception filters), explain what each layer is responsible for, and why does NestJS separate these concerns instead of using a single middleware chain like Express?</summary>

<!-- Answer will be added later -->

</details>

## Conceptual Depth

<details>
<summary>3. How does NestJS dependency injection work — what does the DI container manage, what are the provider scopes (DEFAULT singleton, REQUEST per-request, TRANSIENT per-injection), when would you use each scope, and what goes wrong when you inject a REQUEST-scoped provider into a DEFAULT-scoped one?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>4. How do custom providers (useClass, useFactory, useValue, useExisting) extend basic class injection in NestJS — when would you use each type, and how do they enable swapping implementations for testing or environment-specific behavior?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>5. When is NestJS worth its framework overhead compared to Express or Fastify — what specific problems does NestJS solve that justify the learning curve and abstraction cost, what are the tradeoffs of using the Fastify adapter instead of Express, and when should you choose plain Express or Fastify instead?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>6. How does module encapsulation work in NestJS — what do exports control, how do dynamic modules (forRoot/forRootAsync) enable configurable shared modules, and how do you detect and resolve circular dependency issues between modules?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>7. How should you structure a NestJS application for clean architecture — how do you map modules to domain boundaries, keep NestJS framework code at the edges (controllers, decorators) while domain logic stays framework-agnostic, and what's the practical boundary between "NestJS structure" and "over-abstraction"?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>8. How do NestJS lifecycle hooks enable clean startup and shutdown — what do onModuleInit and onModuleDestroy do, why is enableShutdownHooks necessary, and how do you implement graceful shutdown that works with Kubernetes preStop hooks?</summary>

<!-- Answer will be added later -->

</details>

## Practical — Guards, Pipes & Interceptors

<details>
<summary>9. Why does NestJS use guards for authentication instead of middleware or inline controller checks — implement JWT authentication with Passport showing the AuthGuard using @nestjs/passport, the JWT strategy configuration, how to extract the user from the token and attach it to the request, and how to make specific routes public with a custom decorator</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>10. Implement role-based authorization with a custom guard and decorator — show a @Roles() decorator, a RolesGuard that checks the user's roles against the required roles, and demonstrate how to use it on controllers and individual routes. Explain why checking permissions is better than checking role names</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>11. Why does NestJS use pipes for validation and transformation instead of handling it in controllers — show a DTO with class-validator decorators, configure the global ValidationPipe with whitelist and transform options, demonstrate what happens with invalid input, and explain why whitelist prevents mass assignment vulnerabilities</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>12. Why are cross-cutting concerns like logging, correlation IDs, and response shaping handled in interceptors rather than in controllers or middleware — show an interceptor implementation using the Observable pipeline (tap for logging, map for transformation), how to generate and propagate a correlation ID, and how to register it globally</summary>

<!-- Answer will be added later -->

</details>

## Practical — Modules & Configuration

<details>
<summary>13. Create a dynamic module using forRoot/forRootAsync pattern — show a DatabaseModule that accepts configuration asynchronously (e.g., reading from ConfigService), demonstrate the module factory, and explain when you need forRootAsync vs forRoot and what the difference is</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>14. Set up @nestjs/config with validation and DI integration — show the ConfigModule setup with a validation schema (using Joi or class-validator), demonstrate injecting ConfigService into a provider, and explain why validating configuration at startup prevents runtime errors in production</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>15. Implement custom providers for different scenarios — show useFactory for creating a provider that depends on configuration, useClass for swapping implementations (e.g., different storage backends), and useExisting for aliasing. Demonstrate how this enables testing and environment-specific behavior</summary>

<!-- Answer will be added later -->

</details>

## Practical — Data Access & Lifecycle

<details>
<summary>16. Integrate Prisma with NestJS — show the PrismaService extending PrismaClient with onModuleInit and onModuleDestroy, inject it into a domain service, and implement a repository-like abstraction on top of it. Explain why wrapping Prisma in a service matters for testability and lifecycle management.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>17. Implement graceful shutdown for a NestJS application running in Kubernetes — show enableShutdownHooks, the onModuleDestroy hooks that close database connections and drain active requests, the Kubernetes preStop hook configuration, and explain what happens to in-flight requests without proper shutdown</summary>

<!-- Answer will be added later -->

</details>

## Practical — Testing & Debugging

<details>
<summary>18. Write unit tests for a NestJS service using Test.createTestingModule — show how to create the testing module, mock dependencies with overrideProvider, test the service methods, and explain when to mock vs when to use real implementations</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>19. Write integration tests for a NestJS controller using supertest — show the test setup that creates the full NestJS application (with real database or test database), make HTTP requests to endpoints, assert on responses, and demonstrate testing the full request lifecycle (guards, pipes, interceptors)</summary>

<!-- Answer will be added later -->

</details><details>
<summary>20. Build a custom exception filter that handles domain-specific errors — show an exception filter that catches custom application errors (NotFoundError, ValidationError, AuthorizationError), maps them to appropriate HTTP responses with consistent error format, and explain how exception filters differ from try/catch in controllers</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>21. Set up a NestJS application with proper module boundaries for a domain-driven structure — show how to organize modules around business domains (users, orders, payments) rather than technical layers, demonstrate how modules communicate through well-defined exports, and what the module dependency graph should look like</summary>

<!-- Answer will be added later -->

</details>

---

## Experience-Based Questions

These questions test real-world experience. Prepare by mapping them to your own projects and situations.

<details>
<summary>22. Tell me about a time you chose NestJS for a project (or inherited a NestJS codebase) — what were the benefits and drawbacks compared to plain Express, and how did the framework's opinions help or hinder the team?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>23. Describe a time you structured a NestJS application to scale across a team — how did you organize modules, what conventions did you establish, and what architectural decisions did you make about domain boundaries?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>24. Tell me about a time you debugged a complex NestJS issue (DI problems, lifecycle issues, middleware ordering) — what was the symptom, how did you diagnose it, and what did you learn about NestJS internals?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>25. Describe a time you set up testing for a NestJS application — what testing strategy did you choose, how did you handle mocking vs real dependencies, and what was the most challenging aspect?</summary>

<!-- Answer framework will be added later -->

</details>
