# Testing

> **32 questions** — 11 theory, 21 practical

- Testing pyramid vs testing trophy: unit vs integration vs E2E distribution, static analysis as the trophy's base layer (linting, type checking), when unit tests suffice vs real dependencies
- The role of tests beyond catching bugs: refactoring confidence, documentation, CI gating, verifying error paths and side effects
- TDD: red-green-refactor cycle, when TDD helps vs when it's overhead, outside-in vs inside-out approaches
- Test doubles: mocks vs stubs vs spies vs fakes — when to use each, behavior verification vs state verification, AAA (Arrange-Act-Assert) test structure
- Real dependencies vs mocks: confidence-per-test tradeoffs, when to use real databases/services vs test doubles, over-mocking pitfalls
- Test isolation and parallel execution: transaction rollback, truncation, per-test schemas, shared state prevention
- Contract testing between microservices (Pact)
- Testing error paths and edge cases: boundary conditions, invalid inputs, timeout behavior, verifying correct error types and messages
- Flaky tests: root causes (timing, shared state, order dependence, external services), CI-only failures, triage and diagnosis
- Deciding what NOT to test: coverage targets vs wasted effort
- Code coverage: line vs branch vs statement coverage, why 100% coverage is misleading, using coverage to find untested paths rather than as a target
- HTTP-layer integration tests: testing routes end-to-end with supertest or similar, request/response validation against real databases
- Mocking external services: HTTP interception (nock/msw) vs dependency injection, tradeoffs of each isolation approach
- Testing async and event-driven flows: polling/retry helpers, deterministic event assertions, avoiding brittle timeouts and sleep-based waits
- Testing database interactions: repository layer tests, migration verification, testing complex queries and transactions
- CI with real services: Docker Compose and Testcontainers for ephemeral PostgreSQL/Redis, container lifecycle in test suites
- Test data seeding: factories, fixtures, and builder patterns
- CI performance: parallelization, test splitting, caching, affected-only runs, optimizing feedback loops
- Snapshot testing: when it helps (API contracts, serialization) vs when it's noise, and snapshot maintenance discipline
- Testing auth/authorization middleware without a real auth provider
- Testing background jobs and scheduled tasks: verifying job execution, retry behavior, and failure handling without real queues

---

## Foundational

<details>
<summary>1. How should you distribute tests across the testing pyramid (or testing trophy) — what are the tradeoffs between unit, integration, and E2E tests in terms of speed, confidence, and maintenance cost, what role does static analysis (linting, type checking) play as the trophy's base layer, when do unit tests give enough confidence on their own, and when must you test against real dependencies?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>2. What role do tests play beyond catching bugs — how do they enable confident refactoring, serve as living documentation, gate CI/CD pipelines, and verify error paths and side effects that manual testing misses?</summary>

<!-- Answer will be added later -->

</details>

## Conceptual Depth

<details>
<summary>3. When should you use real dependencies (databases, services) vs mocks/stubs in tests — what are the confidence-per-test tradeoffs, what are the pitfalls of over-mocking (tests pass but production breaks), and how do you decide the right level of faking for each test type?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>4. How do you achieve test isolation when running tests in parallel — what are the approaches (transaction rollback, table truncation, per-test schemas), how does shared state cause test interference, and what are the tradeoffs of each isolation strategy in terms of speed and reliability?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>5. What causes flaky tests and why are they dangerous — what are the common root causes (timing dependencies, shared state, order dependence, external service reliance), why do some tests fail only in CI, and how do you systematically triage and fix flakiness?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>6. Why is contract testing important for microservices — how does consumer-driven contract testing (Pact) work, what does it catch that integration tests miss, and when is it overkill for smaller teams?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>7. How do you decide what NOT to test — when do coverage targets become counterproductive, what kinds of code yield low value from testing (trivial getters, framework glue), and how do you identify the highest-risk areas that deserve the most test investment?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>8. When does snapshot testing help vs create noise — what are the good use cases (API response contracts, serialization formats), when do snapshots become meaningless (large HTML blobs, frequently changing output), and what snapshot maintenance discipline prevents "just update all snapshots" from hiding regressions?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>9. When does TDD (test-driven development) genuinely improve code quality and design vs when is it overhead — explain the red-green-refactor cycle, compare outside-in vs inside-out TDD, and identify the types of code where TDD pays off vs where writing tests after is more practical.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>10. What are the different types of test doubles (mocks, stubs, spies, fakes) and when should you use each — what is the difference between behavior verification (asserting that a function was called with specific arguments) and state verification (asserting on the final state), how does the AAA (Arrange-Act-Assert) pattern structure tests that use these doubles, and how does choosing the wrong type of double lead to brittle or meaningless tests?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>11. Why does CI test performance matter for developer velocity — what's a good feedback loop target, how do you measure it, and what are the main levers for speeding up a large test suite (parallelization, caching, affected-only runs)? Explain when each lever gives diminishing returns.</summary>

<!-- Answer will be added later -->

</details>

## Practical — Test Writing Patterns

<details>
<summary>12. Write an HTTP-layer integration test using Jest or Vitest with supertest that tests a REST endpoint end-to-end against a real PostgreSQL database — show the test setup (database connection, migration), the test itself (seed data, make request, assert response), and teardown. Explain why this catches bugs that a unit test with mocked repositories would miss</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>13. Mock an external HTTP API dependency in tests using nock or msw — show both approaches (HTTP interceptor vs dependency injection of a fake client), compare them for maintainability, and demonstrate testing both the happy path and error responses from the external service</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>14. Write a test for an async event-driven flow (e.g., service publishes an event → consumer processes it → database updated) without using brittle setTimeout waits — show the approach using polling, event callbacks, or test helpers that await the eventual state, and explain why hardcoded delays make tests flaky</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>15. Test auth and authorization middleware without calling a real auth provider — show how to mock the JWT verification or session lookup, test that unauthenticated requests are rejected, test role-based access (admin vs regular user), and explain why you need both unit tests of the middleware and integration tests of protected routes</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>16. Test a Bull/BullMQ background job processor — show how to set up a test that enqueues a job, waits for it to complete, and asserts on the side effects (database changes, external calls), how to test retry behavior and failure handling, and what makes testing job processors harder than testing synchronous code</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>17. Implement test data seeding using factories and builder patterns — show a factory function that creates a user with sensible defaults but allows overrides, demonstrate the builder pattern for complex related entities (user → orders → items), explain why factories are better than shared fixtures for test isolation, and explain what failures occur without proper test data management (stale fixtures, test interdependence, data leaks between tests)</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>18. Write snapshot tests for an API response contract — show the test that snapshots a serialized response, demonstrate how to update snapshots intentionally when the contract changes, and set up a review process that prevents "update all snapshots" from hiding regressions</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>19. Set up test isolation for a test suite that runs in parallel against a shared PostgreSQL database — show the transaction-rollback approach (wrap each test in a transaction that rolls back), compare with truncation and per-test schema approaches, and demonstrate what breaks without isolation when tests run concurrently</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>20. Write repository-layer tests for a service that uses PostgreSQL — show a test for a complex query (e.g., filtering with joins or aggregation), a test that verifies a multi-step transaction rolls back correctly on failure, and a test that validates a database migration produces the expected schema. Explain why testing at the repository layer catches bugs that HTTP-layer integration tests miss.</summary>

<!-- Answer will be added later -->

</details>

## Practical — CI & Test Infrastructure

<details>
<summary>21. Why do ephemeral containers (Testcontainers) matter for CI testing over alternatives like shared test databases or in-memory fakes — set up a CI pipeline that uses Testcontainers to spin up real PostgreSQL and Redis instances for integration tests, show the test setup and teardown, and explain what breaks when teams share persistent test databases instead?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>22. Set up test parallelization for a CI pipeline — show how to split tests across CI workers, explain what goes wrong when parallelization is done poorly (shared state, port conflicts, flaky ordering), and demonstrate how affected-only runs (only running tests related to changed files) reduce feedback time without sacrificing confidence.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>23. Set up consumer-driven contract tests with Pact between two services — show the consumer test (defining expectations), the provider verification (replaying expectations against the real service), how contracts are shared (Pact Broker or CI artifacts), and what happens when a provider breaks a contract</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>24. A test suite has several flaky tests that only fail in CI — walk through the diagnosis: identifying the flaky tests from CI logs, categorizing the root cause (timing, shared state, order dependence, resource constraints), and applying the fix for each category. Show how to quarantine flaky tests without losing coverage</summary>

<!-- Answer will be added later -->

</details>

## Practical — Testing Strategies & Decisions

<details>
<summary>25. You're starting a new service — show the testing strategy document: what percentage of each test type (unit, integration, E2E), what gets mocked vs uses real dependencies, what CI infrastructure is needed, and where you'd invest testing effort first for the highest confidence-per-hour</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>26. A production bug slipped through because mocked data didn't match real database query behavior — walk through refactoring an over-mocked test suite (mocked database, mocked HTTP clients, mocked dependencies) to use real dependencies where it matters. Show a before/after comparison of a test, explain what bugs the mocked version misses, and identify which mocks to keep (external services) vs which to replace (database)</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>27. Write tests that verify error paths and side effects — show a test that confirms invalid input returns the correct error type and status code, and a test that verifies a failed transaction doesn't leave partial state in the database. For each, show the test code and explain why these error-path tests catch more production bugs than happy-path-only testing.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>28. Set up test coverage measurement and decide on a target — show the coverage configuration, explain why 100% coverage is counterproductive, how to identify which uncovered lines actually matter, and demonstrate a coverage report that shows meaningful gaps vs irrelevant misses</summary>

<!-- Answer will be added later -->

</details>

---

## Experience-Based Questions

These questions test real-world experience. Prepare by mapping them to your own projects and situations.

<details>
<summary>29. Tell me about a time you improved a team's testing strategy — what was the before state, what changes did you make (different test types, CI infrastructure, test data management), and what impact did it have on confidence and velocity?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>30. Describe a time you dealt with a persistent flaky test problem — what was the root cause, how did you diagnose it, and what systemic changes did you make to prevent similar flakiness?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>31. Tell me about a time a production bug slipped through your test suite — what kind of test would have caught it, why didn't you have that test, and what did you change afterward?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>32. Describe a time you set up testing infrastructure for CI (Testcontainers, Docker Compose, contract tests) — what problem were you solving, what was the implementation, and what tradeoffs did you accept?</summary>

<!-- Answer framework will be added later -->

</details>
