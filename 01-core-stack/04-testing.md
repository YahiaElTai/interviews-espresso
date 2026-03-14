# Testing

> **25 questions** — 9 theory, 16 practical

- Testing pyramid vs testing trophy: unit vs integration vs E2E distribution, static analysis as the trophy's base layer (linting, type checking), when unit tests suffice vs real dependencies
- The role of tests beyond catching bugs: refactoring confidence, documentation, CI gating, verifying error paths and side effects
- TDD: red-green-refactor cycle, when TDD helps vs when it's overhead, outside-in vs inside-out approaches
- Test doubles: mocks vs stubs vs spies vs fakes — when to use each, behavior verification vs state verification, AAA (Arrange-Act-Assert) test structure
- Real dependencies vs mocks: confidence-per-test tradeoffs, when to use real databases/services vs test doubles, over-mocking pitfalls
- Test isolation and parallel execution: transaction rollback, truncation, per-test schemas, shared state prevention
- Testing error paths and edge cases: boundary conditions, invalid inputs, timeout behavior, verifying correct error types and messages
- Flaky tests: root causes (timing, shared state, order dependence, external services), CI-only failures, triage and diagnosis
- Deciding what NOT to test: coverage targets vs wasted effort
- HTTP-layer integration tests: testing routes end-to-end with supertest or similar, request/response validation against real databases
- Mocking external services: HTTP interception (nock/msw) vs dependency injection, tradeoffs of each isolation approach
- Testing async and event-driven flows: polling/retry helpers, deterministic event assertions, avoiding brittle timeouts and sleep-based waits
- Testing database interactions: repository layer tests, migration verification, testing complex queries and transactions
- CI with real services: Docker Compose and Testcontainers for ephemeral PostgreSQL/Redis, container lifecycle in test suites
- Testing auth/authorization middleware without a real auth provider
- Test data seeding: factories, fixtures, and builder patterns
- CI performance: parallelization, test splitting, caching, affected-only runs, optimizing feedback loops

---

## Foundational

<details>
<summary>1. How should you distribute tests across the testing pyramid (or testing trophy) — what are the tradeoffs between unit, integration, and E2E tests in terms of speed, confidence, and maintenance cost, what role does static analysis (linting, type checking) play as the trophy's base layer, when do unit tests give enough confidence on their own, and when must you test against real dependencies?</summary>

**Testing pyramid (traditional):** Many unit tests at the base, fewer integration tests in the middle, fewest E2E tests at the top. The idea is that unit tests are fast and cheap, so maximize them.

**Testing trophy (Kent C. Dodds):** Flips the emphasis. From bottom to top: static analysis → unit → **integration** (biggest layer) → E2E. The argument is that integration tests give the best confidence-per-test because they exercise how components actually work together, which is where most bugs live.

**Tradeoffs by test type:**

| | Speed | Confidence | Maintenance cost |
|---|---|---|---|
| **Unit** | Fastest (ms) | Low-medium — proves isolated logic works, but misses integration bugs | Low — no infrastructure, but mocks can couple to implementation |
| **Integration** | Medium (seconds) | High — tests real interactions between layers (HTTP → service → DB) | Medium — needs real databases, more setup |
| **E2E** | Slowest (seconds-minutes) | Highest — proves the full user flow works | High — brittle selectors, flaky timing, complex infrastructure |

**Static analysis as the trophy's base:** TypeScript's type system and ESLint catch entire categories of bugs (typos, null access, wrong argument types, unused variables) with zero runtime cost. This is the highest-ROI layer — it runs on every keystroke and catches bugs before you even write a test.

**When unit tests suffice:** Pure business logic — validation functions, data transformations, parsers, math/algorithm code. If the function takes input and returns output with no side effects, a unit test gives near-full confidence.

**When you need real dependencies:** Anything involving queries, transactions, or schema constraints. A unit test that mocks the database can't catch a wrong JOIN, a missing index, or a constraint violation. If the bug you're worried about lives in the interaction between your code and the dependency, you need the real dependency.

</details>

<details>
<summary>2. What role do tests play beyond catching bugs — how do they enable confident refactoring, serve as living documentation, gate CI/CD pipelines, and verify error paths and side effects that manual testing misses?</summary>

**Refactoring confidence:** Tests are the safety net that makes refactoring possible. Without them, renaming a function, extracting a module, or changing a data structure is a gamble — you won't know what broke until production tells you. With a solid test suite, you refactor freely and the tests immediately tell you if behavior changed. This is arguably the highest long-term value of tests.

**Living documentation:** Tests show how code is actually used. Reading a test for `createOrder()` tells you what inputs it expects, what it returns, and what side effects it has — more reliably than comments or docs, because tests break when behavior changes. Comments go stale; tests don't (or they fail, which alerts you).

**CI/CD gating:** Tests are the automated quality gate before code reaches production. A CI pipeline that runs tests on every PR prevents broken code from merging. Without this gate, you're relying on manual review to catch behavioral regressions — which doesn't scale. Tests enable deployment confidence: if the suite passes, you can deploy with reasonable assurance.

**Error paths and side effects:** Manual testing naturally follows the happy path — you enter valid data and check the result. Tests can systematically exercise what happens with invalid input, network timeouts, database constraint violations, and concurrent requests. They also verify side effects: did the audit log get written? Was the email event published? Did the cache get invalidated? These are nearly impossible to verify consistently by hand.

Tests aren't just a bug-catching mechanism — they're an engineering tool that enables velocity.

</details>

## Conceptual Depth

<details>
<summary>3. When should you use real dependencies (databases, services) vs mocks/stubs in tests — what are the confidence-per-test tradeoffs, what are the pitfalls of over-mocking (tests pass but production breaks), and how do you decide the right level of faking for each test type?</summary>

**The decision framework:**

| Dependency | Use real | Use mock/stub |
|---|---|---|
| **Your database (PostgreSQL, Redis)** | Yes — queries, constraints, and transactions are where bugs hide | Only for pure unit tests of business logic above the repo layer |
| **External HTTP APIs** (Stripe, Twilio, third-party) | No — slow, flaky, costs money, rate limits | Yes — use nock/msw to intercept HTTP or inject a fake client |
| **Message queues** (Kafka, RabbitMQ) | Use real in integration tests (via Testcontainers) | Stub for unit tests of message handlers |
| **File system** | Usually real (fast, local) | Mock only if testing in a constrained CI environment |

**Confidence-per-test tradeoffs:** A unit test with a mocked database runs in 1ms but can't catch a wrong SQL query, a missing migration column, or a unique constraint violation. An integration test against real PostgreSQL takes 50ms but catches all of those. The confidence gain per test is dramatically higher with real dependencies for anything that touches data.

**Over-mocking pitfalls:**

- **False confidence.** You mock the repository to return `{ id: 1, name: "Alice" }`. The test passes. But in production, the query has a typo in the WHERE clause and returns nothing. The mock hid the real bug.
- **Tests coupled to implementation.** If you mock every internal function call, renaming or restructuring code breaks every test — even when behavior is unchanged. The tests are testing *how* the code works, not *what* it does.
- **Behavior drift.** Mock responses reflect what you *think* the dependency returns, not what it *actually* returns. Over time, the real API changes but your mocks don't.

**How to decide the right level:**

1. **What can go wrong?** If the bug you're worried about is in the interaction with the dependency (SQL query logic, cache invalidation timing), you need the real thing.
2. **What's the cost of using the real thing?** If it's fast and local (database via Testcontainers), use it. If it's slow, flaky, or expensive (external API), mock it.
3. **Rule of thumb:** Mock at the boundary of what you own. Use real dependencies for everything inside your system; fake everything outside it.

</details>

<details>
<summary>4. How do you achieve test isolation when running tests in parallel — what are the approaches (transaction rollback, table truncation, per-test schemas), how does shared state cause test interference, and what are the tradeoffs of each isolation strategy in terms of speed and reliability?</summary>

**The problem:** When tests run in parallel and share a database, Test A inserts a user, Test B counts users and expects 1 but finds 2 because Test A's data is visible. Tests pass alone but fail together — the classic shared-state flakiness.

**Three isolation approaches:**

**1. Transaction rollback (recommended for most cases)**

Wrap each test in a transaction and roll it back after the test completes. Data is never committed, so other tests never see it.

```typescript
beforeEach(async () => {
  await db.query('BEGIN');
});

afterEach(async () => {
  await db.query('ROLLBACK');
});
```

- **Speed:** Fastest — rollback is nearly instant, no data touches disk.
- **Reliability:** Excellent isolation — uncommitted data is invisible to other connections.
- **Limitation:** Your code under test can't use its own transactions (nested `BEGIN`/`COMMIT` inside an already-open transaction behaves differently). If your service manages transactions explicitly, this approach can mask real behavior. Use `SAVEPOINT` to work around this.

**2. Table truncation**

After each test (or before each test), truncate all tables. Every test starts with a clean database.

```typescript
afterEach(async () => {
  await db.query('TRUNCATE users, orders, items RESTART IDENTITY CASCADE');
});
```

- **Speed:** Slower than rollback — truncation acquires locks and resets sequences. For tables with foreign keys, `CASCADE` adds overhead.
- **Reliability:** Good isolation, but tests running concurrently can still interfere between the test and the truncation. Works best with sequential execution or per-worker databases.
- **Advantage:** Works even when your code manages its own transactions.

**3. Per-test schemas (or per-worker databases)**

Each test worker gets its own PostgreSQL schema or database. Complete isolation — no interference possible.

```typescript
beforeAll(async () => {
  const schema = `test_worker_${process.env.VITEST_POOL_ID}`;
  await db.query(`CREATE SCHEMA IF NOT EXISTS ${schema}`);
  await db.query(`SET search_path TO ${schema}`);
  await runMigrations(db);
});
```

- **Speed:** Slowest setup — creating schemas and running migrations per worker adds seconds.
- **Reliability:** Perfect isolation — nothing is shared.
- **Best for:** Large test suites with many parallel workers where transaction rollback doesn't work (e.g., code under test commits its own transactions).

**Practical recommendation:** Start with transaction rollback. If your service manages its own transactions, use per-worker schemas with truncation between tests within each worker. Reserve per-test schemas for the rare case where even per-worker isolation isn't enough.

</details>

<details>
<summary>5. What causes flaky tests and why are they dangerous — what are the common root causes (timing dependencies, shared state, order dependence, external service reliance), why do some tests fail only in CI, and how do you systematically triage and fix flakiness?</summary>

**Why flaky tests are dangerous:** They erode trust in the test suite. Once developers learn to re-run CI and ignore failures, *real* failures slip through too. A flaky suite is almost worse than no suite — it has all the maintenance cost with none of the gating value.

**Common root causes:**

1. **Timing dependencies.** Tests that `await sleep(500)` to wait for an async operation. Works on your fast local machine, fails on a slow CI runner. Any test that depends on wall-clock time or assumes "this will finish within X ms" is flaky by nature.

2. **Shared state.** Tests share a database, file, global variable, or in-memory cache. Test A writes data that Test B doesn't expect. Passes when run alone, fails when run together.

3. **Order dependence.** Test B relies on data that Test A created. Tests pass when run in sequence (A then B), fail when run in reverse or parallel. This is a special case of shared state.

4. **External service reliance.** Tests that hit real APIs (payment gateway, email service) fail when the service is slow, down, or rate-limiting.

5. **Resource contention.** Port conflicts (two tests bind to the same port), file locks, or thread pool exhaustion — especially in parallel CI.

**Why CI-only failures happen:**

- CI runners are slower (shared, smaller machines) — timing-sensitive tests fail.
- CI runs tests in parallel by default — shared state surfaces.
- CI environments have different network behavior (no access to localhost services you run locally).
- CI containers may have less memory — tests that barely fit locally OOM in CI.

**Systematic triage:**

1. **Identify:** CI tools can track tests that fail intermittently (most CI platforms show "flaky" badges, and you can use `vitest run --reporter=json --outputFile=results.json` to track results over time). Look for tests that fail < 100% of the time.
2. **Reproduce:** Run the test in isolation (`vitest run path/to/test`). If it passes alone, run it with its neighbors. If it only fails in CI, try running with the same parallelism and resource constraints.
3. **Categorize:** Is it timing? (Look for sleeps, timeouts.) Shared state? (Look for database writes without cleanup.) Order dependence? (Randomize test order.) External service? (Check for HTTP calls without mocks.)
4. **Fix by category:**
   - **Timing:** Replace `sleep()` with polling/retry helpers that wait for a condition.
   - **Shared state:** Add proper isolation (transaction rollback, truncation — see question 4).
   - **Order dependence:** Each test must set up its own data. No test should depend on another test's side effects.
   - **External services:** Mock them (nock/msw) or use contract tests instead.
   - **Resource contention:** Use dynamic port allocation, unique file names, per-worker resources.

</details>

<details>
<summary>6. How do you decide what NOT to test — when do coverage targets become counterproductive, what kinds of code yield low value from testing (trivial getters, framework glue), and how do you identify the highest-risk areas that deserve the most test investment?</summary>

**When coverage targets become counterproductive:** Chasing 100% coverage incentivizes writing tests for trivial code (getters, type definitions, framework wiring) that will never break in a meaningful way. These tests add maintenance burden — they break on refactoring but never catch real bugs. A team hitting 95% coverage with 30% of tests being trivial has worse velocity than a team at 80% coverage where every test is meaningful.

Coverage is useful as a *floor* (e.g., "don't drop below 70%") and as a way to spot untested areas, but it should never be a *target to maximize*.

**Low-value code to skip:**

- **Trivial getters/setters and DTOs.** TypeScript's type system already guarantees the shape. Testing `getUser() { return this.user; }` adds nothing.
- **Framework glue code.** NestJS module declarations, route registrations, middleware wiring — these are tested by the integration tests that exercise the routes. A unit test of a module definition is testing the framework, not your code.
- **Third-party library behavior.** Don't test that `bcrypt.hash()` works. Test that *your* code calls it correctly.
- **Configuration files.** YAML, JSON config files. Validate them with schemas, not unit tests.

**Highest-risk areas that deserve investment:**

1. **Business logic with conditionals.** Pricing calculations, permission checks, state machines — anywhere a wrong branch means wrong behavior.
2. **Data access layer.** SQL queries, especially complex ones with JOINs, filters, aggregations. Bugs here are silent (wrong data returned, not crashes).
3. **Error paths.** What happens when the database is down? When input validation fails? When a downstream service returns 500? These paths rarely get manual testing but frequently break in production.
4. **Integration boundaries.** Where your service talks to another service, a database, or a message queue. Contract mismatches live here.
5. **Code that has broken before.** Past bugs are the best predictor of future bugs. Add a regression test for every production bug fix.

**The heuristic:** Ask "if this code breaks in production, what's the impact?" High impact + low manual-testing likelihood = high test value.

</details>

<details>
<summary>7. When does TDD (test-driven development) genuinely improve code quality and design vs when is it overhead — explain the red-green-refactor cycle, compare outside-in vs inside-out TDD, and identify the types of code where TDD pays off vs where writing tests after is more practical.</summary>

**The red-green-refactor cycle:**

1. **Red:** Write a failing test that describes the behavior you want. The test must fail — this confirms it's actually testing something.
2. **Green:** Write the *minimum* code to make the test pass. No over-engineering, no extra features.
3. **Refactor:** Clean up the code (extract functions, rename variables, remove duplication) while keeping all tests green. The tests give you safety to restructure freely.

Repeat for each small increment of behavior.

**Outside-in vs inside-out TDD:**

- **Outside-in (London school):** Start from the outer layer (HTTP handler or use case) and work inward. You mock the inner layers (repository, services) and write them later. This produces well-defined interfaces between layers because you design each layer's API based on what the consumer actually needs. Downside: heavy mocking early on.

- **Inside-out (Chicago/Detroit school):** Start from the core domain logic (entities, value objects) and work outward. Each layer is built on top of already-tested lower layers — no mocks needed. Produces well-tested building blocks. Downside: you might build components that don't fit together well at the outer layer.

**When TDD pays off:**

- **Pure business logic.** Pricing engines, validation rules, state machines — the input/output contract is clear, and TDD forces you to handle edge cases (null input, boundary values) because you write them as test cases first.
- **Algorithm-heavy code.** Parsers, transformers, data processing pipelines. TDD's incremental approach prevents you from over-engineering.
- **Bug fixes.** Write a test that reproduces the bug (red), fix it (green). Guarantees the bug stays fixed. This is TDD's clearest ROI.

**When writing tests after is more practical:**

- **Exploratory/prototyping work.** When you don't know the API shape yet. Writing tests for an interface that changes every 10 minutes is wasted effort. Prototype first, stabilize the design, then add tests.
- **Framework integration code.** NestJS controllers, middleware wiring, database migrations — the behavior is largely dictated by the framework. Testing after (via integration tests) is faster and higher-value.
- **UI code.** Rapidly changing layouts and interactions. Write tests once the component stabilizes.

**The honest take:** TDD is a design tool, not a testing tool. It's most valuable when the design isn't obvious. If you already know the shape of the code, writing tests after is fine — what matters is that the tests exist and are good, not when they were written.

</details>

<details>
<summary>8. What are the different types of test doubles (mocks, stubs, spies, fakes) and when should you use each — what is the difference between behavior verification (asserting that a function was called with specific arguments) and state verification (asserting on the final state), how does the AAA (Arrange-Act-Assert) pattern structure tests that use these doubles, and how does choosing the wrong type of double lead to brittle or meaningless tests?</summary>

**The four types of test doubles:**

| Double | What it does | When to use | Vitest example |
|---|---|---|---|
| **Stub** | Returns canned data, no assertions on how it's called | When you need to control a dependency's output (e.g., "the config service returns this value") | `vi.fn().mockReturnValue(42)` |
| **Mock** | Pre-programmed with expectations — verifies it was called correctly | When the *interaction* is the behavior being tested (e.g., "email service was called with the right args") | `vi.fn()` + `expect(mock).toHaveBeenCalledWith(...)` |
| **Spy** | Wraps a real function — lets it execute but records calls | When you want real behavior but need to verify it was called | `vi.spyOn(service, 'save')` |
| **Fake** | A working but simplified implementation (in-memory database, fake SMTP server) | When you need realistic behavior without the real infrastructure cost | Hand-written class implementing the same interface |

**Behavior verification vs state verification:**

```typescript
// Behavior verification — did we CALL the right thing?
it('sends a welcome email on signup', async () => {
  const sendEmail = vi.fn();
  const service = new UserService({ sendEmail });

  await service.signup({ email: 'alice@test.com' });

  expect(sendEmail).toHaveBeenCalledWith({
    to: 'alice@test.com',
    template: 'welcome',
  });
});

// State verification — is the RESULT correct?
it('creates user in database on signup', async () => {
  const service = new UserService({ db, sendEmail: vi.fn() });

  await service.signup({ email: 'alice@test.com' });

  const user = await db.query('SELECT * FROM users WHERE email = $1', ['alice@test.com']);
  expect(user).toBeDefined();
  expect(user.email).toBe('alice@test.com');
});
```

State verification is generally preferred — it tests *what* happened, not *how*. Behavior verification is necessary when the side effect itself is the thing being tested (sending an email, publishing an event) and there's no observable state change to assert on.

**AAA pattern:**

```typescript
it('calculates order total with discount', () => {
  // Arrange — set up test data and doubles
  const priceService = { getDiscount: vi.fn().mockReturnValue(0.1) }; // stub
  const calculator = new OrderCalculator(priceService);

  // Act — execute the behavior being tested
  const total = calculator.calculateTotal([{ price: 100, qty: 2 }]);

  // Assert — verify the outcome
  expect(total).toBe(180); // 200 - 10% discount
});
```

AAA makes tests scannable — you immediately see what's being set up, what's being tested, and what's being verified. One act per test. If you need multiple acts, you need multiple tests.

**Wrong double = brittle or meaningless tests:**

- **Over-mocking (using mocks when stubs suffice):** If you mock the database and assert it was called with `SELECT * FROM users WHERE id = $1`, you're testing your SQL string, not your query behavior. Rename a column and the test breaks — but so does a refactor that produces the same result with different SQL.
- **Stubbing when you need real behavior:** Stubbing the database to return `[{id: 1}]` doesn't tell you if your actual query works. The test passes but the real query has a bug.
- **Faking when a spy suffices:** Building a full in-memory database implementation when you just need to verify a method was called is wasted effort.

</details>

<details>
<summary>9. Why does CI test performance matter for developer velocity — what's a good feedback loop target, how do you measure it, and what are the main levers for speeding up a large test suite (parallelization, caching, affected-only runs)? Explain when each lever gives diminishing returns.</summary>

**Why it matters:** CI test time directly impacts how often developers push code. A 30-minute CI pipeline means developers context-switch to other work while waiting, lose flow, and batch larger changes (which are riskier). A 5-minute pipeline means pushing small, frequent changes with immediate feedback — the foundation of continuous delivery.

**Feedback loop targets:**

- **< 5 minutes:** Excellent. Developers wait for CI before moving on.
- **5-10 minutes:** Acceptable. Most teams can live with this.
- **10-20 minutes:** Painful. Developers start batching PRs and ignoring CI until it finishes.
- **> 20 minutes:** Actively harmful. Developers merge without waiting, CI becomes an afterthought.

**How to measure:** Track wall-clock time from push to green/red result. Break it down by phase: checkout, install, build, test (unit), test (integration), deploy preview. The bottleneck is often not where you think — measure before optimizing.

**Main levers:**

**1. Parallelization — split tests across CI workers**

Run test files across multiple CI jobs simultaneously. A 20-minute sequential suite becomes 5 minutes across 4 workers.

- **Diminishing returns:** Adding workers has diminishing returns when individual slow tests become the bottleneck. If one test file takes 3 minutes, adding more workers can't make the suite faster than 3 minutes. Also, each worker has setup overhead (checkout, install, boot database). At some point, setup time dominates.

**2. Caching — avoid redundant work**

Cache `node_modules` (keyed by lockfile hash), Docker images, build artifacts, and database images. Skip the install step when dependencies haven't changed.

- **Diminishing returns:** Once the main caches (dependencies, Docker layers, build output) are in place, further caching adds complexity (cache invalidation bugs) with minimal time savings. Cache hit rates above 90% have little room for improvement.

**3. Affected-only runs — test what changed**

Only run tests related to files that changed in the PR. Tools like `vitest --changed` or Nx/Turborepo's dependency graph can identify which test files to run.

- **Diminishing returns:** Works well for monorepos with clear module boundaries. Falls apart when modules are heavily interconnected (changing a shared utility affects everything). Also creates a confidence gap — you need periodic full-suite runs (nightly or on main branch) to catch cross-module regressions that affected-only runs miss.

**Combining levers:** The best CI pipelines layer all three: cache dependencies, run only affected tests, and parallelize those across workers. The key insight is to measure each lever's impact independently — optimize the biggest bottleneck first.

</details>

## Practical — Test Writing Patterns

<details>
<summary>10. Write an HTTP-layer integration test using Jest or Vitest with supertest that tests a REST endpoint end-to-end against a real PostgreSQL database — show the test setup (database connection, migration), the test itself (seed data, make request, assert response), and teardown. Explain why this catches bugs that a unit test with mocked repositories would miss</summary>

```typescript
// tests/integration/users.test.ts
import { describe, it, expect, beforeAll, afterAll, beforeEach } from 'vitest';
import request from 'supertest';
import { PostgreSqlContainer, type StartedPostgreSqlContainer } from '@testcontainers/postgresql';
import { Pool } from 'pg';
import { createApp } from '../../src/app'; // Express app factory

let container: StartedPostgreSqlContainer;
let pool: Pool;
let app: Express.Application;

// --- Setup: start real PostgreSQL, run migrations ---
beforeAll(async () => {
  container = await new PostgreSqlContainer('postgres:16').start();

  pool = new Pool({ connectionString: container.getConnectionUri() });

  // Run migrations against the real database
  await pool.query(`
    CREATE TABLE users (
      id SERIAL PRIMARY KEY,
      email TEXT UNIQUE NOT NULL,
      name TEXT NOT NULL,
      created_at TIMESTAMPTZ DEFAULT NOW()
    );
    CREATE TABLE orders (
      id SERIAL PRIMARY KEY,
      user_id INTEGER REFERENCES users(id) ON DELETE CASCADE,
      total NUMERIC(10,2) NOT NULL
    );
  `);

  app = createApp({ pool }); // inject the real pool into the app
}, 30_000); // container startup can take a few seconds

// --- Isolation: clean slate for each test ---
beforeEach(async () => {
  await pool.query('TRUNCATE users, orders RESTART IDENTITY CASCADE');
});

// --- Teardown: clean up ---
afterAll(async () => {
  await pool.end();
  await container.stop();
});

// --- Tests ---
describe('POST /users', () => {
  it('creates a user and returns 201 with the user data', async () => {
    const res = await request(app)
      .post('/users')
      .send({ email: 'alice@example.com', name: 'Alice' })
      .expect(201);

    expect(res.body).toMatchObject({
      id: expect.any(Number),
      email: 'alice@example.com',
      name: 'Alice',
    });

    // Verify it's actually in the database
    const { rows } = await pool.query('SELECT * FROM users WHERE email = $1', ['alice@example.com']);
    expect(rows).toHaveLength(1);
  });

  it('returns 409 when email already exists', async () => {
    // Seed a user
    await pool.query("INSERT INTO users (email, name) VALUES ('alice@example.com', 'Alice')");

    const res = await request(app)
      .post('/users')
      .send({ email: 'alice@example.com', name: 'Alice Again' })
      .expect(409);

    expect(res.body.error).toContain('already exists');
  });
});

describe('GET /users/:id', () => {
  it('returns user with their order count', async () => {
    // Seed user + orders
    const { rows: [user] } = await pool.query(
      "INSERT INTO users (email, name) VALUES ('bob@example.com', 'Bob') RETURNING id"
    );
    await pool.query('INSERT INTO orders (user_id, total) VALUES ($1, 99.99), ($1, 49.50)', [user.id]);

    const res = await request(app)
      .get(`/users/${user.id}`)
      .expect(200);

    expect(res.body).toMatchObject({
      id: user.id,
      name: 'Bob',
      orderCount: 2,
    });
  });

  it('returns 404 for non-existent user', async () => {
    await request(app).get('/users/99999').expect(404);
  });
});
```

**Why this catches bugs that mocked tests miss:**

1. **SQL correctness.** If the `GET /users/:id` handler has a JOIN query that counts orders, a mocked repo would just return `{ orderCount: 2 }` — you'd never know the JOIN is wrong. The real database catches typos, wrong column names, and incorrect JOIN conditions.

2. **Constraint enforcement.** The 409 test works because PostgreSQL's `UNIQUE` constraint on email actually fires. A mock would need to manually simulate this — and might not match real behavior (e.g., case sensitivity of email comparison).

3. **Serialization/type issues.** PostgreSQL returns `NUMERIC` as a string in Node.js. A mock returning `99.99` as a number would pass, but the real response might have `"99.99"` as a string — causing bugs in the frontend.

4. **Migration correctness.** The test proves that the schema migrations actually produce a working database that the queries can run against. Mocked tests can't verify this at all.

</details>

<details>
<summary>11. Mock an external HTTP API dependency in tests using nock or msw — show both approaches (HTTP interceptor vs dependency injection of a fake client), compare them for maintainability, and demonstrate testing both the happy path and error responses from the external service</summary>

**Approach 1: HTTP interceptor (nock)**

Nock intercepts outgoing HTTP requests at the `http`/`https` module level. No code changes needed — your service makes real HTTP calls, nock catches them.

```typescript
import nock from 'nock';
import { describe, it, expect, afterEach } from 'vitest';
import { PaymentService } from '../../src/services/payment';

afterEach(() => {
  nock.cleanAll(); // remove all interceptors after each test
});

describe('PaymentService', () => {
  const service = new PaymentService({ baseUrl: 'https://api.stripe.com' });

  it('processes a payment (happy path)', async () => {
    nock('https://api.stripe.com')
      .post('/v1/charges', (body) => body.amount === 5000)
      .reply(200, {
        id: 'ch_123',
        status: 'succeeded',
        amount: 5000,
      });

    const result = await service.charge({ amount: 5000, currency: 'usd' });

    expect(result.status).toBe('succeeded');
    expect(result.id).toBe('ch_123');
  });

  it('handles API errors gracefully', async () => {
    nock('https://api.stripe.com')
      .post('/v1/charges')
      .reply(402, {
        error: { type: 'card_error', message: 'Card declined' },
      });

    const result = await service.charge({ amount: 5000, currency: 'usd' });

    expect(result.status).toBe('failed');
    expect(result.error).toContain('Card declined');
  });

  it('handles network errors', async () => {
    nock('https://api.stripe.com')
      .post('/v1/charges')
      .replyWithError('connection refused');

    await expect(service.charge({ amount: 5000, currency: 'usd' }))
      .rejects.toThrow('connection refused');
  });
});
```

**Approach 2: HTTP interceptor (msw)**

MSW intercepts at the network level using request handlers. Same principle as nock but with a different API that also works in browsers.

```typescript
import { http, HttpResponse } from 'msw';
import { setupServer } from 'msw/node';
import { describe, it, expect, beforeAll, afterAll, afterEach } from 'vitest';
import { PaymentService } from '../../src/services/payment';

const server = setupServer();

beforeAll(() => server.listen());
afterEach(() => server.resetHandlers());
afterAll(() => server.close());

describe('PaymentService (msw)', () => {
  const service = new PaymentService({ baseUrl: 'https://api.stripe.com' });

  it('processes a payment (happy path)', async () => {
    server.use(
      http.post('https://api.stripe.com/v1/charges', () => {
        return HttpResponse.json({ id: 'ch_123', status: 'succeeded', amount: 5000 });
      })
    );

    const result = await service.charge({ amount: 5000, currency: 'usd' });
    expect(result.status).toBe('succeeded');
  });

  it('handles API errors', async () => {
    server.use(
      http.post('https://api.stripe.com/v1/charges', () => {
        return HttpResponse.json(
          { error: { type: 'card_error', message: 'Card declined' } },
          { status: 402 }
        );
      })
    );

    const result = await service.charge({ amount: 5000, currency: 'usd' });
    expect(result.status).toBe('failed');
  });
});
```

**Approach 3: Dependency injection of a fake client**

Instead of intercepting HTTP, inject an interface that production code uses. In tests, provide a fake implementation.

```typescript
// src/services/payment.ts
interface PaymentClient {
  createCharge(params: ChargeParams): Promise<ChargeResult>;
}

class PaymentService {
  constructor(private client: PaymentClient) {}

  async charge(params: ChargeParams): Promise<ChargeResult> {
    return this.client.createCharge(params);
  }
}

// tests/fakes/fake-payment-client.ts
class FakePaymentClient implements PaymentClient {
  private nextResponse: ChargeResult | Error;

  succeedWith(result: ChargeResult) { this.nextResponse = result; }
  failWith(error: Error) { this.nextResponse = error; }

  async createCharge(): Promise<ChargeResult> {
    if (this.nextResponse instanceof Error) throw this.nextResponse;
    return this.nextResponse;
  }
}

// In tests:
const fakeClient = new FakePaymentClient();
const service = new PaymentService(fakeClient);

fakeClient.succeedWith({ id: 'ch_123', status: 'succeeded' });
const result = await service.charge({ amount: 5000, currency: 'usd' });
expect(result.status).toBe('succeeded');
```

**Comparison:**

| | HTTP interceptor (nock/msw) | Dependency injection (fake client) |
|---|---|---|
| **Setup cost** | Low — no code changes, just add interceptors | Medium — requires defining an interface and swapping implementations |
| **What it tests** | Full HTTP path: serialization, headers, status code parsing, retry logic | Business logic above the HTTP layer only |
| **Maintainability** | Mocks can drift from real API — URL or payload changes break silently | Interface changes break at compile time (TypeScript catches it) |
| **Best for** | Testing that your HTTP client handles real-world responses correctly | Testing business logic that uses the client, when HTTP details don't matter |

**Practical recommendation:** Use HTTP interceptors (nock or msw) for integration tests that verify your service handles real API responses. Use DI fakes for unit tests of business logic that sits above the client layer. MSW is preferable to nock for new projects — it works in both Node.js and browser environments and has a cleaner handler API.

</details>

<details>
<summary>12. Write a test for an async event-driven flow (e.g., service publishes an event → consumer processes it → database updated) without using brittle setTimeout waits — show the approach using polling, event callbacks, or test helpers that await the eventual state, and explain why hardcoded delays make tests flaky</summary>

**Why hardcoded delays are flaky:**

```typescript
// BAD — brittle timeout
await service.publishOrderCreated(order);
await new Promise((r) => setTimeout(r, 500)); // hope the consumer finishes in 500ms
const result = await db.query('SELECT * FROM projections WHERE order_id = $1', [order.id]);
expect(result.rows).toHaveLength(1);
```

This fails when: (1) CI runners are slower than your machine, (2) the consumer takes 600ms under load, or (3) garbage collection pauses extend processing. If you increase the delay to 2 seconds "just to be safe," your test suite becomes slow. There's no correct value — any hardcoded delay is either too short (flaky) or too long (slow).

**Approach 1: Polling helper (most universal)**

```typescript
// tests/helpers/wait-for.ts
async function waitFor<T>(
  fn: () => Promise<T>,
  options: { timeout?: number; interval?: number } = {}
): Promise<T> {
  const { timeout = 5000, interval = 50 } = options;
  const start = Date.now();

  while (true) {
    try {
      const result = await fn();
      return result;
    } catch (error) {
      if (Date.now() - start > timeout) {
        throw new Error(`waitFor timed out after ${timeout}ms: ${(error as Error).message}`);
      }
      await new Promise((r) => setTimeout(r, interval));
    }
  }
}

// Usage in test:
it('projects order after event is consumed', async () => {
  await service.publishOrderCreated({ id: 'order-1', total: 99.99 });

  const projection = await waitFor(async () => {
    const { rows } = await db.query(
      'SELECT * FROM order_projections WHERE order_id = $1', ['order-1']
    );
    if (rows.length === 0) throw new Error('Projection not yet created');
    return rows[0];
  });

  expect(projection.total).toBe('99.99');
  expect(projection.status).toBe('created');
});
```

The polling helper checks every 50ms and returns as soon as the condition is met. Fast machines finish in 50-100ms; slow CI runners take longer but still pass within the timeout. No wasted time, no false failures.

**Approach 2: Event callback / promise-based**

If your consumer emits events or returns promises, await the processing directly:

```typescript
it('processes order event and updates database', async () => {
  // Consumer exposes a promise that resolves when a message is processed
  const processed = consumer.onNextMessage();

  await service.publishOrderCreated({ id: 'order-1', total: 99.99 });

  await processed; // wait for the consumer to finish, not a timer

  const { rows } = await db.query(
    'SELECT * FROM order_projections WHERE order_id = $1', ['order-1']
  );
  expect(rows).toHaveLength(1);
});
```

This is the fastest and most deterministic approach, but requires the consumer to expose a signal (event emitter, promise, or callback). Design your consumers with testability in mind.

**Approach 3: In-process consumer for tests**

Instead of running the consumer as a separate process, import and run it in-process during tests. This lets you `await` the full publish-consume-write cycle synchronously:

```typescript
it('end-to-end event processing', async () => {
  const consumer = new OrderConsumer({ db, queue: testQueue });
  await consumer.start();

  await testQueue.publish('order.created', { id: 'order-1', total: 99.99 });
  await consumer.drain(); // process all pending messages

  const { rows } = await db.query(
    'SELECT * FROM order_projections WHERE order_id = $1', ['order-1']
  );
  expect(rows).toHaveLength(1);

  await consumer.stop();
});
```

**Which approach to use:**

- **Polling helper:** Default choice. Works with any async system without modifying production code.
- **Event callback:** When you control the consumer and can expose a "done" signal. Most deterministic.
- **In-process consumer:** For full integration tests where you want to test the entire pipeline without IPC overhead.

</details>

<details>
<summary>13. Set up test isolation for a test suite that runs in parallel against a shared PostgreSQL database — show the transaction-rollback approach (wrap each test in a transaction that rolls back), compare with truncation and per-test schema approaches, and demonstrate what breaks without isolation when tests run concurrently</summary>

Building on the isolation strategies from Q4, here's the practical implementation for parallel test suites.

**What breaks without isolation:**

```typescript
// Test A (worker 1)
it('creates a user', async () => {
  await db.query("INSERT INTO users (email) VALUES ('alice@test.com')");
  const { rows } = await db.query('SELECT COUNT(*) FROM users');
  expect(rows[0].count).toBe('1'); // FAILS — worker 2 also inserted a user
});

// Test B (worker 2) — runs at the same time
it('lists users returns empty for new system', async () => {
  const { rows } = await db.query('SELECT COUNT(*) FROM users');
  expect(rows[0].count).toBe('0'); // FAILS — worker 1 inserted a user
});
```

Both tests pass when run alone, both fail when run together. The database is the shared mutable state.

**Transaction rollback with SAVEPOINT (handling code that manages its own transactions):**

The basic transaction rollback approach (see Q4) breaks when your code under test calls `BEGIN`/`COMMIT` itself. `SAVEPOINT` solves this:

```typescript
import { Pool, PoolClient } from 'pg';
import { beforeEach, afterEach } from 'vitest';

let client: PoolClient;

beforeEach(async () => {
  client = await pool.connect();
  await client.query('BEGIN');
  await client.query('SAVEPOINT test_start');
});

afterEach(async () => {
  await client.query('ROLLBACK TO SAVEPOINT test_start');
  await client.query('ROLLBACK');
  client.release();
});
```

The `SAVEPOINT` creates a nested restore point inside the outer transaction. When the code under test issues its own `COMMIT`, it doesn't actually commit (the outer transaction holds it). Rolling back to the savepoint undoes everything cleanly.

**Per-worker schemas (for full parallel isolation):**

When transaction rollback isn't practical, give each Vitest worker its own schema:

```typescript
// tests/setup/per-worker-schema.ts
import { Pool } from 'pg';
import { beforeAll, afterAll, beforeEach } from 'vitest';

const workerId = process.env.VITEST_POOL_ID || '1';
const schema = `test_worker_${workerId}`;

let pool: Pool;

beforeAll(async () => {
  pool = new Pool({ connectionString: process.env.DATABASE_URL });

  // Safe: schema name comes from test runner, not user input
  await pool.query(`DROP SCHEMA IF EXISTS ${schema} CASCADE`);
  await pool.query(`CREATE SCHEMA ${schema}`);
  await pool.query(`SET search_path TO ${schema}`);

  await runMigrations(pool);
});

beforeEach(async () => {
  await pool.query(`
    DO $$ DECLARE r RECORD;
    BEGIN
      FOR r IN SELECT tablename FROM pg_tables WHERE schemaname = '${schema}' LOOP
        EXECUTE 'TRUNCATE TABLE ${schema}.' || r.tablename || ' RESTART IDENTITY CASCADE';
      END LOOP;
    END $$;
  `);
});

afterAll(async () => {
  await pool.query(`DROP SCHEMA IF EXISTS ${schema} CASCADE`);
  await pool.end();
});
```

**Recommendation:** Start with transaction rollback (Q4). Use the SAVEPOINT variant when your code manages its own transactions. Escalate to per-worker schemas for large parallel suites where rollback isn't practical. See Q4 for the full comparison of tradeoffs between all three approaches.

</details>

<details>
<summary>14. Write repository-layer tests for a service that uses PostgreSQL — show a test for a complex query (e.g., filtering with joins or aggregation), a test that verifies a multi-step transaction rolls back correctly on failure, and a test that validates a database migration produces the expected schema. Explain why testing at the repository layer catches bugs that HTTP-layer integration tests miss.</summary>

**Test setup (shared across all examples):**

```typescript
import { describe, it, expect, beforeAll, afterAll, beforeEach } from 'vitest';
import { PostgreSqlContainer, type StartedPostgreSqlContainer } from '@testcontainers/postgresql';
import { Pool } from 'pg';
import { OrderRepository } from '../../src/repositories/order';

let container: StartedPostgreSqlContainer;
let pool: Pool;
let repo: OrderRepository;

beforeAll(async () => {
  container = await new PostgreSqlContainer('postgres:16').start();
  pool = new Pool({ connectionString: container.getConnectionUri() });
  await runMigrations(pool);
  repo = new OrderRepository(pool);
}, 30_000);

beforeEach(async () => {
  await pool.query('TRUNCATE users, orders, order_items RESTART IDENTITY CASCADE');
});

afterAll(async () => {
  await pool.end();
  await container.stop();
});
```

**Test 1: Complex query with joins and aggregation**

```typescript
describe('OrderRepository.getOrderSummaries', () => {
  it('returns orders with item count and total, filtered by status', async () => {
    // Seed: 2 users, 3 orders, multiple items
    const { rows: [alice] } = await pool.query(
      "INSERT INTO users (email, name) VALUES ('alice@test.com', 'Alice') RETURNING id"
    );
    const { rows: [bob] } = await pool.query(
      "INSERT INTO users (email, name) VALUES ('bob@test.com', 'Bob') RETURNING id"
    );

    // Alice: 1 completed order with 2 items, 1 pending order
    const { rows: [order1] } = await pool.query(
      "INSERT INTO orders (user_id, status) VALUES ($1, 'completed') RETURNING id", [alice.id]
    );
    await pool.query(
      'INSERT INTO order_items (order_id, product, price) VALUES ($1, $2, $3), ($1, $4, $5)',
      [order1.id, 'Widget', 25.00, 'Gadget', 75.00]
    );
    await pool.query(
      "INSERT INTO orders (user_id, status) VALUES ($1, 'pending')", [alice.id]
    );

    // Bob: 1 completed order
    const { rows: [order3] } = await pool.query(
      "INSERT INTO orders (user_id, status) VALUES ($1, 'completed') RETURNING id", [bob.id]
    );
    await pool.query(
      'INSERT INTO order_items (order_id, product, price) VALUES ($1, $2, $3)',
      [order3.id, 'Gizmo', 50.00]
    );

    // Act: fetch completed orders only
    const summaries = await repo.getOrderSummaries({ status: 'completed' });

    // Assert
    expect(summaries).toHaveLength(2);
    expect(summaries).toEqual(
      expect.arrayContaining([
        expect.objectContaining({
          orderId: order1.id,
          userName: 'Alice',
          itemCount: 2,
          total: '100.00', // NUMERIC comes back as string in pg
        }),
        expect.objectContaining({
          orderId: order3.id,
          userName: 'Bob',
          itemCount: 1,
          total: '50.00',
        }),
      ])
    );
  });
});
```

**Test 2: Transaction rollback on failure**

```typescript
describe('OrderRepository.transferOrder', () => {
  it('rolls back both updates if the second fails', async () => {
    // Seed: user with one order
    const { rows: [user] } = await pool.query(
      "INSERT INTO users (email, name) VALUES ('alice@test.com', 'Alice') RETURNING id"
    );
    const { rows: [order] } = await pool.query(
      "INSERT INTO orders (user_id, status) VALUES ($1, 'active') RETURNING id", [user.id]
    );

    // transferOrder: debit source account, credit destination, update order owner
    // If destination user doesn't exist, the whole transaction should roll back
    await expect(
      repo.transferOrder({
        orderId: order.id,
        fromUserId: user.id,
        toUserId: 99999, // doesn't exist — FK violation
      })
    ).rejects.toThrow();

    // Verify: order still belongs to original user (transaction rolled back)
    const { rows: [unchanged] } = await pool.query(
      'SELECT user_id, status FROM orders WHERE id = $1', [order.id]
    );
    expect(unchanged.user_id).toBe(user.id);
    expect(unchanged.status).toBe('active'); // not partially updated
  });
});
```

**Test 3: Migration produces expected schema**

```typescript
describe('migrations', () => {
  it('creates expected tables with correct columns and constraints', async () => {
    // Query the information_schema to verify migration output
    const { rows: columns } = await pool.query(`
      SELECT column_name, data_type, is_nullable, column_default
      FROM information_schema.columns
      WHERE table_name = 'orders'
      ORDER BY ordinal_position
    `);

    expect(columns).toEqual(
      expect.arrayContaining([
        expect.objectContaining({ column_name: 'id', data_type: 'integer', is_nullable: 'NO' }),
        expect.objectContaining({ column_name: 'user_id', data_type: 'integer', is_nullable: 'NO' }),
        expect.objectContaining({ column_name: 'status', data_type: 'character varying', is_nullable: 'NO' }),
        expect.objectContaining({ column_name: 'created_at', is_nullable: 'NO' }),
      ])
    );

    // Verify foreign key exists
    const { rows: fks } = await pool.query(`
      SELECT tc.constraint_name, ccu.table_name AS foreign_table
      FROM information_schema.table_constraints tc
      JOIN information_schema.constraint_column_usage ccu ON tc.constraint_name = ccu.constraint_name
      WHERE tc.table_name = 'orders' AND tc.constraint_type = 'FOREIGN KEY'
    `);

    expect(fks).toEqual(
      expect.arrayContaining([
        expect.objectContaining({ foreign_table: 'users' }),
      ])
    );
  });

  it('enforces NOT NULL constraint on required fields', async () => {
    const { rows: [user] } = await pool.query(
      "INSERT INTO users (email, name) VALUES ('test@test.com', 'Test') RETURNING id"
    );

    // status is NOT NULL — should reject null
    await expect(
      pool.query('INSERT INTO orders (user_id, status) VALUES ($1, NULL)', [user.id])
    ).rejects.toThrow(/not-null/i);
  });
});
```

**Why repository-layer tests catch bugs that HTTP-layer tests miss:**

1. **Query logic isolation.** An HTTP test exercises the full stack: routing → validation → service → repository → database. When it fails, you don't know which layer broke. A repository test pinpoints the query itself — if the JOIN is wrong or the aggregation miscounts, you see it immediately.

2. **Edge cases in queries.** HTTP tests typically test a few representative cases. Repository tests can systematically test query edge cases: empty results, NULL handling, boundary values for LIMIT/OFFSET, case sensitivity in LIKE queries — without the overhead of HTTP request setup.

3. **Transaction behavior.** HTTP tests can verify that an endpoint returns 500 on failure, but they can't easily verify that partial state was cleaned up. Repository tests can inspect the database directly after a failed transaction to confirm nothing leaked (as shown in Test 2).

4. **Migration verification.** HTTP tests implicitly rely on migrations being correct — if a column is missing, the query fails. But they can't verify that the schema matches expectations (correct types, constraints, indexes). Repository-layer migration tests make this explicit.

5. **Performance-relevant queries.** Repository tests are the right place to verify that complex queries use expected indexes or return within acceptable time — something HTTP-layer tests obscure behind request handling overhead.

</details>

<details>
<summary>15. Test auth and authorization middleware without calling a real auth provider — show how to mock the JWT verification or session lookup, test that unauthenticated requests are rejected, test role-based access (admin vs regular user), and explain why you need both unit tests of the middleware and integration tests of protected routes</summary>

**Mock JWT verification:**

The key insight is that JWT verification is just a function call. You can mock the verification library or inject a custom verifier.

```typescript
// src/middleware/auth.ts
import jwt from 'jsonwebtoken';
import { Request, Response, NextFunction } from 'express';

export function authMiddleware(secret: string) {
  return (req: Request, res: Response, next: NextFunction) => {
    const token = req.headers.authorization?.replace('Bearer ', '');
    if (!token) return res.status(401).json({ error: 'No token provided' });

    try {
      const payload = jwt.verify(token, secret) as { userId: string; role: string };
      req.user = payload;
      next();
    } catch {
      return res.status(401).json({ error: 'Invalid token' });
    }
  };
}

export function requireRole(...roles: string[]) {
  return (req: Request, res: Response, next: NextFunction) => {
    if (!req.user) return res.status(401).json({ error: 'Not authenticated' });
    if (!roles.includes(req.user.role)) {
      return res.status(403).json({ error: 'Insufficient permissions' });
    }
    next();
  };
}
```

**Unit tests of the middleware (fast, isolated):**

```typescript
import { describe, it, expect, vi } from 'vitest';
import jwt from 'jsonwebtoken';
import { authMiddleware, requireRole } from '../../src/middleware/auth';

const TEST_SECRET = 'test-secret';

// Helper to create a valid token without a real auth provider
function createToken(payload: { userId: string; role: string }): string {
  return jwt.sign(payload, TEST_SECRET, { expiresIn: '1h' });
}

// Helper to create mock Express req/res/next
function mockReqRes(overrides: Partial<Request> = {}) {
  const req = { headers: {}, user: undefined, ...overrides } as any;
  const res = { status: vi.fn().mockReturnThis(), json: vi.fn().mockReturnThis() } as any;
  const next = vi.fn();
  return { req, res, next };
}

describe('authMiddleware', () => {
  const middleware = authMiddleware(TEST_SECRET);

  it('rejects requests without a token', () => {
    const { req, res, next } = mockReqRes();

    middleware(req, res, next);

    expect(res.status).toHaveBeenCalledWith(401);
    expect(next).not.toHaveBeenCalled();
  });

  it('rejects requests with an invalid token', () => {
    const { req, res, next } = mockReqRes({
      headers: { authorization: 'Bearer invalid.token.here' },
    });

    middleware(req, res, next);

    expect(res.status).toHaveBeenCalledWith(401);
    expect(next).not.toHaveBeenCalled();
  });

  it('attaches user to request with a valid token', () => {
    const token = createToken({ userId: 'user-1', role: 'admin' });
    const { req, res, next } = mockReqRes({
      headers: { authorization: `Bearer ${token}` },
    });

    middleware(req, res, next);

    expect(next).toHaveBeenCalled();
    expect(req.user).toEqual(expect.objectContaining({ userId: 'user-1', role: 'admin' }));
  });

  it('rejects expired tokens', () => {
    const token = jwt.sign({ userId: 'user-1', role: 'admin' }, TEST_SECRET, { expiresIn: '-1s' });
    const { req, res, next } = mockReqRes({
      headers: { authorization: `Bearer ${token}` },
    });

    middleware(req, res, next);

    expect(res.status).toHaveBeenCalledWith(401);
  });
});

describe('requireRole', () => {
  it('allows users with the required role', () => {
    const { req, res, next } = mockReqRes();
    req.user = { userId: 'user-1', role: 'admin' };

    requireRole('admin')(req, res, next);

    expect(next).toHaveBeenCalled();
  });

  it('rejects users without the required role', () => {
    const { req, res, next } = mockReqRes();
    req.user = { userId: 'user-1', role: 'viewer' };

    requireRole('admin', 'editor')(req, res, next);

    expect(res.status).toHaveBeenCalledWith(403);
    expect(next).not.toHaveBeenCalled();
  });
});
```

**Integration tests of protected routes (with supertest):**

```typescript
import request from 'supertest';
import { createApp } from '../../src/app';

const TEST_SECRET = 'test-secret';
const app = createApp({ jwtSecret: TEST_SECRET, pool });

function authHeader(role: string): string {
  const token = jwt.sign({ userId: 'user-1', role }, TEST_SECRET);
  return `Bearer ${token}`;
}

describe('GET /admin/users (protected route)', () => {
  it('returns 401 without a token', async () => {
    await request(app).get('/admin/users').expect(401);
  });

  it('returns 403 for a non-admin user', async () => {
    await request(app)
      .get('/admin/users')
      .set('Authorization', authHeader('viewer'))
      .expect(403);
  });

  it('returns 200 with user list for admin', async () => {
    await pool.query("INSERT INTO users (email, name) VALUES ('alice@test.com', 'Alice')");

    const res = await request(app)
      .get('/admin/users')
      .set('Authorization', authHeader('admin'))
      .expect(200);

    expect(res.body.users).toHaveLength(1);
  });
});
```

**Why you need both:**

- **Unit tests of middleware** are fast (no HTTP, no database) and test the auth logic exhaustively: invalid tokens, expired tokens, missing headers, each role permutation. They catch logic bugs in the middleware itself.

- **Integration tests of protected routes** verify that the middleware is actually wired to the routes correctly. A unit test can't catch a missing `requireRole('admin')` on a route — the middleware works perfectly, but the route forgot to use it. Integration tests catch wiring bugs, middleware ordering issues (auth before rate limiting?), and the interaction between auth and the actual handler (does the handler correctly use `req.user`?).

Neither alone is sufficient. Unit tests without integration tests miss wiring. Integration tests without unit tests make it slow and hard to test every auth edge case.

</details>

<details>
<summary>16. Implement test data seeding using factories and builder patterns — show a factory function that creates a user with sensible defaults but allows overrides, demonstrate the builder pattern for complex related entities (user → orders → items), explain why factories are better than shared fixtures for test isolation, and explain what failures occur without proper test data management (stale fixtures, test interdependence, data leaks between tests)</summary>

**Factory function with sensible defaults:**

```typescript
// tests/factories/user.ts
import { Pool } from 'pg';

interface UserInput {
  email?: string;
  name?: string;
  role?: string;
}

interface User {
  id: number;
  email: string;
  name: string;
  role: string;
}

let counter = 0;

export async function createUser(pool: Pool, overrides: UserInput = {}): Promise<User> {
  counter++;
  const data = {
    email: overrides.email ?? `user-${counter}-${Date.now()}@test.com`,
    name: overrides.name ?? `Test User ${counter}`,
    role: overrides.role ?? 'viewer',
  };

  const { rows: [user] } = await pool.query(
    'INSERT INTO users (email, name, role) VALUES ($1, $2, $3) RETURNING *',
    [data.email, data.name, data.role]
  );
  return user;
}

// Usage: each test creates exactly the data it needs
it('promotes user to admin', async () => {
  const user = await createUser(pool, { role: 'viewer' });
  await service.promote(user.id);
  // ...
});
```

The factory generates unique emails automatically, so tests never collide on unique constraints. Overrides let you control only what matters for each test.

**Builder pattern for complex related entities:**

```typescript
// tests/factories/order-builder.ts
import { Pool } from 'pg';
import { createUser } from './user';

interface OrderItemInput {
  product?: string;
  price?: number;
  quantity?: number;
}

class OrderBuilder {
  private userOverrides: UserInput = {};
  private orderStatus = 'pending';
  private items: OrderItemInput[] = [];

  constructor(private pool: Pool) {}

  withUser(overrides: UserInput): this {
    this.userOverrides = overrides;
    return this;
  }

  withStatus(status: string): this {
    this.orderStatus = status;
    return this;
  }

  withItem(item: OrderItemInput = {}): this {
    this.items.push(item);
    return this;
  }

  withItems(count: number): this {
    for (let i = 0; i < count; i++) {
      this.items.push({ product: `Product ${i + 1}`, price: 10 + i, quantity: 1 });
    }
    return this;
  }

  async build() {
    const user = await createUser(this.pool, this.userOverrides);

    const { rows: [order] } = await this.pool.query(
      'INSERT INTO orders (user_id, status) VALUES ($1, $2) RETURNING *',
      [user.id, this.orderStatus]
    );

    const items = [];
    for (const item of this.items) {
      const { rows: [created] } = await this.pool.query(
        'INSERT INTO order_items (order_id, product, price, quantity) VALUES ($1, $2, $3, $4) RETURNING *',
        [order.id, item.product ?? 'Widget', item.price ?? 19.99, item.quantity ?? 1]
      );
      items.push(created);
    }

    return { user, order, items };
  }
}

// Factory function for convenience
export function buildOrder(pool: Pool): OrderBuilder {
  return new OrderBuilder(pool);
}
```

Usage:

```typescript
it('calculates order total with multiple items', async () => {
  const { order } = await buildOrder(pool)
    .withUser({ name: 'Alice' })
    .withStatus('completed')
    .withItem({ product: 'Widget', price: 25.00, quantity: 2 })
    .withItem({ product: 'Gadget', price: 50.00, quantity: 1 })
    .build();

  const total = await repo.calculateTotal(order.id);
  expect(total).toBe('100.00'); // 25*2 + 50*1
});

it('lists orders with 5+ items for bulk processing', async () => {
  await buildOrder(pool).withItems(3).build(); // should NOT appear
  const { order } = await buildOrder(pool).withItems(6).build(); // should appear

  const bulkOrders = await repo.getBulkOrders();
  expect(bulkOrders).toHaveLength(1);
  expect(bulkOrders[0].id).toBe(order.id);
});
```

**Why factories are better than shared fixtures:**

| Shared fixtures | Factories |
|---|---|
| One `seed.sql` file loaded before tests — all tests share the same "Alice" and "Bob" | Each test creates its own data with unique identifiers |
| Adding a test requires modifying the shared fixture, potentially breaking other tests | Adding a test is self-contained — no shared file to update |
| Hard to understand what data a test depends on — you have to cross-reference the fixture | Test reads top-to-bottom: the data setup is right there |
| Stale fixtures: schema changes require updating the fixture file, and it's easy to miss columns | Factories use INSERT — schema changes cause compile/runtime errors immediately |

**What goes wrong without proper test data management:**

1. **Stale fixtures.** You add a `NOT NULL` column to `users`. The shared `seed.sql` doesn't include it. All tests break with a migration error, not a test failure — confusing and time-consuming to debug.

2. **Test interdependence.** Test A creates "Alice" in the fixture and expects `id: 1`. Test B also expects `id: 1` to be Alice. Someone adds a new user to the fixture before Alice — now she's `id: 2` and both tests break.

3. **Data leaks between tests.** Test A inserts a user and doesn't clean up. Test B queries "all users" and gets an unexpected extra result. This manifests as tests that pass alone but fail when run together — the hardest flakiness to debug.

4. **Unique constraint collisions.** Two tests both insert `email: 'test@test.com'`. They pass sequentially (truncation between tests) but fail in parallel. Factories with unique auto-generated values prevent this entirely.

</details>

## Practical — CI & Test Infrastructure

<details>
<summary>17. Why do ephemeral containers (Testcontainers) matter for CI testing over alternatives like shared test databases or in-memory fakes — set up a CI pipeline that uses Testcontainers to spin up real PostgreSQL and Redis instances for integration tests, show the test setup and teardown, and explain what breaks when teams share persistent test databases instead?</summary>

**Why ephemeral containers matter:**

Shared test databases are the #1 source of test infrastructure pain. Two developers run tests at the same time and interfere with each other's data. A CI run leaves dirty state that breaks the next run. Someone runs a migration on the shared database and breaks the branch that hasn't merged that migration yet.

Ephemeral containers solve all of this: each test run gets a fresh, isolated database that's destroyed when tests finish. No shared state, no stale data, no version conflicts.

**In-memory fakes** (like SQLite in-memory for PostgreSQL tests) are tempting but dangerous — they have different SQL syntax, different type handling, no support for PostgreSQL-specific features (JSONB, arrays, CTEs, `ON CONFLICT`). Tests pass against SQLite and fail in production against PostgreSQL. You're testing a different system.

**Test setup with Testcontainers:**

```typescript
// tests/setup/containers.ts
import { PostgreSqlContainer, type StartedPostgreSqlContainer } from '@testcontainers/postgresql';
import { RedisContainer, type StartedRedisContainer } from '@testcontainers/redis';
import { Pool } from 'pg';
import { createClient, type RedisClientType } from 'redis';

let pgContainer: StartedPostgreSqlContainer;
let redisContainer: StartedRedisContainer;
let pool: Pool;
let redis: RedisClientType;

export async function setup() {
  // Start both containers in parallel
  [pgContainer, redisContainer] = await Promise.all([
    new PostgreSqlContainer('postgres:16').start(),
    new RedisContainer('redis:7').start(),
  ]);

  pool = new Pool({ connectionString: pgContainer.getConnectionUri() });
  await runMigrations(pool);

  redis = createClient({ url: redisContainer.getConnectionUrl() });
  await redis.connect();

  // Make available to tests via globalThis
  globalThis.__TEST_POOL__ = pool;
  globalThis.__TEST_REDIS__ = redis;
}

// Type declarations for globalThis usage
declare global {
  var __TEST_POOL__: Pool;
  var __TEST_REDIS__: RedisClientType;
}

export async function teardown() {
  await pool.end();
  await redis.disconnect();
  await pgContainer.stop();
  await redisContainer.stop();
}
```

```typescript
// vitest.config.ts
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    globalSetup: ['./tests/setup/containers.ts'],
    testTimeout: 15_000,
  },
});
```

```typescript
// tests/integration/cache.test.ts
import { describe, it, expect, beforeEach } from 'vitest';

const pool = globalThis.__TEST_POOL__;
const redis = globalThis.__TEST_REDIS__;

beforeEach(async () => {
  await pool.query('TRUNCATE users RESTART IDENTITY CASCADE');
  await redis.flushAll();
});

describe('UserService with cache', () => {
  it('caches user after first fetch', async () => {
    await pool.query("INSERT INTO users (email, name) VALUES ('alice@test.com', 'Alice')");

    const service = new UserService({ pool, redis });

    await service.getUser(1); // fetches from DB, caches in Redis
    const cached = await redis.get('user:1');
    expect(cached).toBeTruthy();
    expect(JSON.parse(cached!).name).toBe('Alice');
  });
});
```

**CI pipeline (GitHub Actions):**

```yaml
# .github/workflows/test.yml
name: Tests
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'npm'

      - run: npm ci

      # Testcontainers uses Docker — GitHub Actions runners have Docker pre-installed
      - name: Run integration tests
        run: npm run test:integration
        env:
          TESTCONTAINERS_RYUK_DISABLED: 'false' # cleanup container ensures no orphans
```

No `services:` block needed in the CI config. Testcontainers manages container lifecycle from within the test code — the CI pipeline just needs Docker available.

**What breaks with shared persistent test databases:**

1. **Cross-run contamination.** CI run #1 inserts test data and fails halfway through teardown. CI run #2 starts with leftover data and gets unexpected results. Tests become order-dependent on CI history.

2. **Migration conflicts.** Branch A adds a column. Branch B doesn't have that migration. Both branches run tests against the same database — one of them breaks because the schema doesn't match their code.

3. **Parallel PR interference.** Two PRs run CI simultaneously against the same database. Their test data collides — unique constraint violations, wrong row counts, phantom reads.

4. **"Works for me" debugging.** The shared database has different data depending on who ran what last. Local results don't match CI results. Nobody can reproduce the failure because the database state is non-deterministic.

5. **No version pinning.** Shared databases run one PostgreSQL version. You can't test against PostgreSQL 15 and 16 in parallel. Testcontainers let each test suite specify its own image version.

</details>

<details>
<summary>18. A test suite has several flaky tests that only fail in CI — walk through the diagnosis: identifying the flaky tests from CI logs, categorizing the root cause (timing, shared state, order dependence, resource constraints), and applying the fix for each category. Show how to quarantine flaky tests without losing coverage</summary>

**Step 1: Identify the flaky tests**

Most CI platforms mark tests that pass on retry as flaky. Start with CI-level data:

```bash
# Vitest: enable JSON reporter to track failures over time
vitest run --reporter=json --outputFile=test-results.json

# In CI, compare across runs. If a test fails in < 100% of runs, it's flaky.
# GitHub Actions: check the "Re-run failed jobs" pattern — tests that pass on re-run are flaky.
```

Look for tests with these patterns in CI logs:
- `FAIL` on first run, `PASS` on re-run
- Different tests fail on each run (non-deterministic)
- Tests that always pass locally but sometimes fail in CI

**Step 2: Categorize the root cause**

Run the flaky test in different configurations to isolate the cause:

```bash
# Test alone (passes?) → shared state or order dependence
vitest run tests/integration/orders.test.ts

# Test with its neighbors (fails?) → shared state between files
vitest run tests/integration/

# Test with --sequence.shuffle (fails?) → order dependence
vitest run --sequence.shuffle

# Test with --maxWorkers=1 (passes?) → parallelism issue
vitest run --maxWorkers=1
```

**Step 3: Apply fixes by category**

**Timing issues — replace sleeps with polling:**

```typescript
// BEFORE: brittle
await publishEvent('order.created', order);
await new Promise((r) => setTimeout(r, 1000));
const result = await db.query('SELECT * FROM projections WHERE order_id = $1', [order.id]);

// AFTER: deterministic (using waitFor helper from question 12)
await publishEvent('order.created', order);
const result = await waitFor(async () => {
  const { rows } = await db.query('SELECT * FROM projections WHERE order_id = $1', [order.id]);
  if (rows.length === 0) throw new Error('Not yet projected');
  return rows[0];
});
```

**Shared state — add proper isolation:**

```typescript
// BEFORE: tests share data
describe('orders', () => {
  it('creates an order', async () => {
    await pool.query("INSERT INTO users (email) VALUES ('alice@test.com')");
    // ...
  });

  it('lists orders for a user', async () => {
    // Assumes the user from the previous test exists!
    const user = await pool.query("SELECT * FROM users WHERE email = 'alice@test.com'");
  });
});

// AFTER: each test owns its data
beforeEach(async () => {
  await pool.query('TRUNCATE users, orders RESTART IDENTITY CASCADE');
});

it('lists orders for a user', async () => {
  // Create its own user — no dependency on other tests
  const { rows: [user] } = await pool.query(
    "INSERT INTO users (email) VALUES ('alice@test.com') RETURNING id"
  );
  await pool.query('INSERT INTO orders (user_id, total) VALUES ($1, 99.99)', [user.id]);
  // ...
});
```

**Order dependence — ensure each test is self-contained:**

```typescript
// Add to vitest.config.ts to catch order-dependent tests early:
export default defineConfig({
  test: {
    sequence: { shuffle: true }, // randomize test order
  },
});
```

**Resource constraints (port conflicts, memory):**

```typescript
// BEFORE: hardcoded port
const server = app.listen(3000);

// AFTER: dynamic port
const server = app.listen(0); // OS assigns a free port
const port = (server.address() as AddressInfo).port;
```

**Step 4: Quarantine flaky tests without losing coverage**

```typescript
// vitest.config.ts — use test tags or file patterns
export default defineConfig({
  test: {
    include: ['tests/**/*.test.ts'],
    exclude: ['tests/**/*.flaky.test.ts'], // quarantined by default
  },
});
```

```typescript
// Rename flaky tests: orders.test.ts → orders.flaky.test.ts
// Run them in a separate CI job that doesn't block merging:
```

```yaml
# .github/workflows/test.yml
jobs:
  tests:
    runs-on: ubuntu-latest
    steps:
      - run: npm run test # excludes *.flaky.test.ts — this gates PRs

  flaky-tests:
    runs-on: ubuntu-latest
    continue-on-error: true # doesn't block merge
    steps:
      - run: npx vitest run tests/**/*.flaky.test.ts
```

**The quarantine discipline:** Quarantine is a temporary state, not a graveyard. Track quarantined tests in a list. Each sprint, pick the top flaky test, diagnose it, fix it, and move it back. If a test stays quarantined for more than 2 sprints, either fix it or delete it — a permanently quarantined test provides zero value.

</details>

<details>
<summary>19. Set up test parallelization for a CI pipeline — show how to split tests across CI workers, explain what goes wrong when parallelization is done poorly (shared state, port conflicts, flaky ordering), and demonstrate how affected-only runs (only running tests related to changed files) reduce feedback time without sacrificing confidence.</summary>

**Splitting tests across CI workers (GitHub Actions matrix):**

```yaml
# .github/workflows/test.yml
name: Tests
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        shard: [1, 2, 3, 4]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'npm'

      - run: npm ci

      - name: Run tests (shard ${{ matrix.shard }}/4)
        run: npx vitest run --shard=${{ matrix.shard }}/4
        env:
          VITEST_POOL_ID: ${{ matrix.shard }}
```

Vitest's `--shard` flag splits test files evenly across workers. A 20-minute sequential suite becomes ~5 minutes across 4 shards.

**For smarter splitting (balance by test duration):**

```yaml
      # Use timing data from previous runs to balance shards
      - name: Run tests with timing-based split
        run: |
          npx vitest run --shard=${{ matrix.shard }}/4 \
            --reporter=json --outputFile=results-${{ matrix.shard }}.json
```

Some teams use tools like `split-tests` or Knapsack Pro that read historical timing data and distribute tests so each shard takes roughly the same time, avoiding the "one slow shard" problem.

**What goes wrong with poor parallelization:**

**1. Shared database state**

```
Shard 1: INSERT INTO users (email) VALUES ('alice@test.com')
Shard 2: INSERT INTO users (email) VALUES ('alice@test.com')
→ UNIQUE constraint violation — one shard fails randomly
```

Fix: Per-worker database schemas (as shown in question 13) or unique factory data per shard.

**2. Port conflicts**

```
Shard 1: app.listen(3000) ✓
Shard 2: app.listen(3000) → EADDRINUSE
```

Fix: Use dynamic ports — `app.listen(0)` lets the OS assign a free port.

**3. Flaky test ordering**

Tests that depend on running in a specific sequence break when shards rearrange them. Shard 1 gets tests A, C, E; shard 2 gets B, D, F. Test D depends on data from test C, which is now on a different shard.

Fix: Every test must set up its own data. No test should depend on another test's side effects. Use `--sequence.shuffle` locally to catch this early.

**4. Resource exhaustion**

Four shards each start a Testcontainers PostgreSQL instance. The CI runner has limited memory and Docker overhead — containers start timing out or OOM-killing each other.

Fix: Use the Vitest global setup to start one container per shard (not per test file). Share the container within a shard via `globalSetup`.

**Affected-only runs:**

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0 # need full history for change detection

      - run: npm ci

      - name: Run affected tests only
        run: npx vitest run --changed HEAD~1
        # --changed runs only test files that import modules
        # that were modified in the latest commit
```

For monorepos, use workspace-aware tools:

```yaml
      # Turborepo: only test packages that changed
      - run: npx turbo run test --filter='...[HEAD~1]'

      # Nx: affected command uses dependency graph
      - run: npx nx affected --target=test --base=origin/main --head=HEAD
```

**The confidence gap:** Affected-only runs miss cross-module regressions. A change in a shared utility might not be detected as affecting a downstream test. Mitigate this with:

1. **Full suite on main branch.** Every merge to main runs all tests — catches anything affected-only missed.
2. **Nightly full runs.** Scheduled pipeline runs the complete suite, alerting on failures.
3. **Conservative change detection.** If a shared package changes, run all consumers' tests.

```yaml
  # Nightly full suite — catches what affected-only misses
  full-test:
    runs-on: ubuntu-latest
    if: github.event_name == 'schedule'
    steps:
      - run: npx vitest run # no --changed, no --shard — everything
```

**Combining everything for optimal CI:**

```yaml
# PR pipeline: fast feedback
on: pull_request
jobs:
  test:
    strategy:
      matrix:
        shard: [1, 2, 3, 4]
    steps:
      - run: npx vitest run --changed origin/main --shard=${{ matrix.shard }}/4

# Main branch: full confidence
on:
  push:
    branches: [main]
jobs:
  test:
    strategy:
      matrix:
        shard: [1, 2, 3, 4]
    steps:
      - run: npx vitest run --shard=${{ matrix.shard }}/4
```

</details>

## Practical — Testing Strategies & Decisions

<details>
<summary>20. You're starting a new service — show the testing strategy document: what percentage of each test type (unit, integration, E2E), what gets mocked vs uses real dependencies, what CI infrastructure is needed, and where you'd invest testing effort first for the highest confidence-per-hour</summary>

**Testing strategy: Order Processing Service**

**Test distribution (testing trophy model):**

| Layer | % of tests | What it covers | Speed |
|---|---|---|---|
| **Static analysis** | n/a (always on) | TypeScript strict mode, ESLint, Prettier | Instant |
| **Unit** | ~25% | Pure business logic: pricing, validation, state machines | < 1s total |
| **Integration** | ~60% | HTTP routes → service → repository → real PostgreSQL/Redis | 30-90s total |
| **E2E** | ~15% | Critical user flows against deployed staging environment | Minutes |

The integration layer is the biggest investment. For a backend service, most bugs live in the interaction between your code and the database/cache/queue — not in isolated functions.

**What gets mocked vs real:**

| Dependency | Test approach | Why |
|---|---|---|
| **PostgreSQL** | Real (Testcontainers) | Query bugs are the #1 source of production issues — mocks can't catch them |
| **Redis** | Real (Testcontainers) | Cache TTL behavior, atomic operations, and data types differ from mocks |
| **External APIs** (Stripe, email, etc.) | Mocked (msw) | Slow, costly, flaky, rate-limited — mock at HTTP layer |
| **Message queue** (Kafka/RabbitMQ) | Real in integration, mocked in unit | Test message serialization and consumer logic against real broker |
| **Internal microservices** | Mocked (msw) in tests, contract tests for schema | You don't own them — mock responses but verify contracts |

**CI infrastructure:**

```yaml
# Required CI services:
# - Docker (for Testcontainers — no shared test databases)
# - Node.js 20+
# - GitHub Actions or equivalent with matrix support for parallelization

# Pipeline stages:
# 1. Lint + type check (30s) — fails fast, catches typos
# 2. Unit tests (15s) — pure logic
# 3. Integration tests (60-90s) — real databases via Testcontainers
# 4. E2E tests (on merge to main only) — full environment
```

**Where to invest first (highest confidence-per-hour):**

**Week 1: Foundation**

1. **TypeScript strict mode + ESLint** — zero-cost bug prevention. Enable `strict: true`, `noUncheckedIndexedAccess`, and the recommended ESLint rules. This catches 20% of bugs with 0 test-writing effort.

2. **Integration test scaffold** — set up Testcontainers with global setup, create the factory pattern (as in question 16), write the first test for your most critical endpoint. This template will be copied for every future test.

3. **CI pipeline with test gating** — PR can't merge without passing tests. Start this from day one, even if you only have 3 tests. The habit matters more than the count.

**Week 2-3: Core coverage**

4. **Repository-layer tests for every query** — test each database query against real PostgreSQL. Cover: happy path, empty results, edge cases (NULL handling, boundary values), and error paths (constraint violations). This is the highest-ROI test writing you'll do.

5. **HTTP integration tests for each endpoint** — test the full request-response cycle with supertest. Cover: valid requests, validation errors (400), not found (404), conflict (409), and unauthorized (401). Use the auth helper pattern from question 15.

6. **Unit tests for business logic** — pricing calculations, validation rules, state machine transitions. These are fast to write and maintain.

**Week 4+: Hardening**

7. **Error path tests** — what happens when PostgreSQL is slow? When Redis is down? When an external API returns 500? Test circuit breakers, retry logic, and graceful degradation.

8. **E2E tests for critical flows** — the 3-5 most important user journeys (e.g., create account → place order → process payment). Run these on merge to main, not on every PR.

**What NOT to do:**

- Don't chase coverage percentages. Aim for 70-80% as a floor, not a target.
- Don't write unit tests for framework glue (controller decorators, module definitions).
- Don't write E2E tests for every endpoint — that's what integration tests are for.
- Don't mock the database. The confidence loss isn't worth the speed gain.

</details>

<details>
<summary>21. A production bug slipped through because mocked data didn't match real database query behavior — walk through refactoring an over-mocked test suite (mocked database, mocked HTTP clients, mocked dependencies) to use real dependencies where it matters. Show a before/after comparison of a test, explain what bugs the mocked version misses, and identify which mocks to keep (external services) vs which to replace (database)</summary>

**The scenario:** Your test suite mocks the database repository, and a bug slips through because the mock returns `{ id: 1, status: 'active' }` but the real query has a wrong WHERE clause and returns nothing. Tests are green, production is broken.

**Before — over-mocked test:**

```typescript
// BEFORE: everything is mocked
import { describe, it, expect, vi } from 'vitest';
import { OrderService } from '../../src/services/order';

describe('OrderService.getActiveOrders', () => {
  it('returns active orders for a user', async () => {
    const mockRepo = {
      findOrders: vi.fn().mockResolvedValue([
        { id: 1, userId: 'user-1', status: 'active', total: '99.99' },
        { id: 2, userId: 'user-1', status: 'active', total: '49.50' },
      ]),
    };
    const mockCache = {
      get: vi.fn().mockResolvedValue(null),
      set: vi.fn(),
    };

    const service = new OrderService({ repo: mockRepo, cache: mockCache });
    const orders = await service.getActiveOrders('user-1');

    expect(orders).toHaveLength(2);
    expect(mockRepo.findOrders).toHaveBeenCalledWith({ userId: 'user-1', status: 'active' });
    expect(mockCache.set).toHaveBeenCalled();
  });
});
```

**What bugs this misses:**

1. **SQL query bugs.** The real `findOrders` query has `WHERE status = 'Active'` (wrong case) or joins on the wrong column. The mock happily returns the canned data regardless.
2. **Type mismatches.** PostgreSQL returns `NUMERIC` as strings (`'99.99'`), but the mock returns numbers (`99.99`). The service code does `total > 50` which works with numbers but fails with string comparison.
3. **Cache serialization.** `cache.set` receives the data, but does `JSON.stringify` handle `Date` objects from the real query? The mock skips this entirely.
4. **Missing columns.** A migration added a `cancelled_at` column. The mock doesn't include it. The service tries to read `order.cancelledAt` and gets `undefined` — silently wrong.

**After — real database, keep external service mocks:**

```typescript
// AFTER: real PostgreSQL, real Redis, mock only external APIs
import { describe, it, expect, beforeAll, afterAll, beforeEach } from 'vitest';
import { PostgreSqlContainer, type StartedPostgreSqlContainer } from '@testcontainers/postgresql';
import { Pool } from 'pg';
import { createClient, type RedisClientType } from 'redis';
import { RedisContainer, type StartedRedisContainer } from '@testcontainers/redis';
import { OrderService } from '../../src/services/order';
import { OrderRepository } from '../../src/repositories/order';
import { createUser, createOrder } from '../factories';

let pgContainer: StartedPostgreSqlContainer;
let redisContainer: StartedRedisContainer;
let pool: Pool;
let redis: RedisClientType;

beforeAll(async () => {
  [pgContainer, redisContainer] = await Promise.all([
    new PostgreSqlContainer('postgres:16').start(),
    new RedisContainer('redis:7').start(),
  ]);
  pool = new Pool({ connectionString: pgContainer.getConnectionUri() });
  await runMigrations(pool);
  redis = createClient({ url: redisContainer.getConnectionUrl() });
  await redis.connect();
}, 30_000);

beforeEach(async () => {
  await pool.query('TRUNCATE users, orders, order_items RESTART IDENTITY CASCADE');
  await redis.flushAll();
});

afterAll(async () => {
  await pool.end();
  await redis.disconnect();
  await pgContainer.stop();
  await redisContainer.stop();
});

describe('OrderService.getActiveOrders', () => {
  it('returns active orders for a user', async () => {
    const user = await createUser(pool);
    await createOrder(pool, { userId: user.id, status: 'active', total: 99.99 });
    await createOrder(pool, { userId: user.id, status: 'active', total: 49.50 });
    await createOrder(pool, { userId: user.id, status: 'cancelled', total: 25.00 }); // should be excluded

    const repo = new OrderRepository(pool);
    const service = new OrderService({ repo, cache: redis });

    const orders = await service.getActiveOrders(user.id);

    expect(orders).toHaveLength(2);
    expect(orders.every((o) => o.status === 'active')).toBe(true);
  });

  it('caches results and returns from cache on second call', async () => {
    const user = await createUser(pool);
    await createOrder(pool, { userId: user.id, status: 'active', total: 99.99 });

    const repo = new OrderRepository(pool);
    const service = new OrderService({ repo, cache: redis });

    await service.getActiveOrders(user.id); // populates cache
    const cached = await redis.get(`orders:active:${user.id}`);
    expect(cached).toBeTruthy();

    // Delete from DB — second call should still return from cache
    await pool.query('TRUNCATE orders CASCADE');
    const orders = await service.getActiveOrders(user.id);
    expect(orders).toHaveLength(1); // served from cache
  });
});
```

**What to keep mocked vs replace with real:**

| Dependency | Verdict | Reason |
|---|---|---|
| **Database (PostgreSQL)** | Replace with real | Query correctness is where production bugs live. Mocking it gives false confidence. |
| **Cache (Redis)** | Replace with real | Serialization, TTL behavior, and atomic operations differ between mocks and real Redis. |
| **External HTTP APIs** (Stripe, email) | Keep mocked (msw/nock) | Slow, costly, rate-limited. Mock at HTTP layer to test your error handling of their responses. |
| **Internal services** | Keep mocked | You don't own them. Use contract tests separately to verify schema compatibility. |

**The refactoring approach:**

1. Set up Testcontainers infrastructure (one-time cost — as shown in question 17).
2. Start with the test that caused the production bug. Rewrite it with real dependencies. Verify the bug would have been caught.
3. Work outward from there — refactor repository tests first (highest ROI), then service tests that touch database/cache.
4. Keep unit tests with mocks for pure business logic that doesn't touch infrastructure.
5. Don't refactor all tests at once. Prioritize by risk: which tests protect the most critical code paths?

</details>

<details>
<summary>22. Write tests that verify error paths and side effects — show a test that confirms invalid input returns the correct error type and status code, and a test that verifies a failed transaction doesn't leave partial state in the database. For each, show the test code and explain why these error-path tests catch more production bugs than happy-path-only testing.</summary>

**Test 1: Invalid input returns correct error type and status code**

```typescript
import { describe, it, expect, beforeAll, afterAll, beforeEach } from 'vitest';
import request from 'supertest';
import { createApp } from '../../src/app';

// Assuming Testcontainers setup from question 10

describe('POST /orders — input validation errors', () => {
  it('returns 400 with specific error for missing required fields', async () => {
    const user = await createUser(pool);

    const res = await request(app)
      .post('/orders')
      .set('Authorization', authHeader(user.id))
      .send({}) // missing required "items" field
      .expect(400);

    expect(res.body).toMatchObject({
      error: 'ValidationError',
      message: expect.stringContaining('items'),
      details: expect.arrayContaining([
        expect.objectContaining({ field: 'items', code: 'required' }),
      ]),
    });
  });

  it('returns 400 for negative item quantities', async () => {
    const user = await createUser(pool);

    const res = await request(app)
      .post('/orders')
      .set('Authorization', authHeader(user.id))
      .send({ items: [{ productId: 'p-1', quantity: -3, price: 10 }] })
      .expect(400);

    expect(res.body.error).toBe('ValidationError');
    expect(res.body.details).toEqual(
      expect.arrayContaining([
        expect.objectContaining({ field: 'items[0].quantity', code: 'minimum' }),
      ])
    );
  });

  it('returns 404 when referencing a non-existent product', async () => {
    const user = await createUser(pool);

    const res = await request(app)
      .post('/orders')
      .set('Authorization', authHeader(user.id))
      .send({ items: [{ productId: 'nonexistent', quantity: 1, price: 10 }] })
      .expect(404);

    expect(res.body.error).toBe('NotFoundError');
    expect(res.body.message).toContain('Product');
  });

  it('returns 409 when item is out of stock', async () => {
    const user = await createUser(pool);
    await pool.query(
      "INSERT INTO products (id, name, stock) VALUES ('p-1', 'Widget', 0)"
    );

    const res = await request(app)
      .post('/orders')
      .set('Authorization', authHeader(user.id))
      .send({ items: [{ productId: 'p-1', quantity: 1, price: 10 }] })
      .expect(409);

    expect(res.body.error).toBe('ConflictError');
    expect(res.body.message).toContain('out of stock');
  });
});
```

**Test 2: Failed transaction leaves no partial state**

```typescript
describe('OrderService.placeOrder — transaction atomicity', () => {
  it('rolls back order and stock deduction when payment fails', async () => {
    // Arrange: user with a product in stock
    const user = await createUser(pool);
    await pool.query(
      "INSERT INTO products (id, name, stock) VALUES ('p-1', 'Widget', 10)"
    );

    // Mock external payment to fail
    server.use(
      http.post('https://api.payments.com/charge', () => {
        return HttpResponse.json(
          { error: 'insufficient_funds' },
          { status: 402 }
        );
      })
    );

    // Act: attempt to place order (should fail at payment step)
    const res = await request(app)
      .post('/orders')
      .set('Authorization', authHeader(user.id))
      .send({ items: [{ productId: 'p-1', quantity: 2, price: 25 }] })
      .expect(402);

    // Assert: no order was created
    const { rows: orders } = await pool.query(
      'SELECT * FROM orders WHERE user_id = $1', [user.id]
    );
    expect(orders).toHaveLength(0);

    // Assert: stock was NOT deducted (transaction rolled back)
    const { rows: [product] } = await pool.query(
      "SELECT stock FROM products WHERE id = 'p-1'"
    );
    expect(product.stock).toBe(10); // still 10, not 8

    // Assert: no order items were created
    const { rows: items } = await pool.query('SELECT * FROM order_items');
    expect(items).toHaveLength(0);
  });

  it('rolls back when a database constraint is violated mid-transaction', async () => {
    const user = await createUser(pool);
    await pool.query(
      "INSERT INTO products (id, name, stock) VALUES ('p-1', 'Widget', 5)"
    );

    // Request 6 items when only 5 in stock — CHECK constraint violation
    const res = await request(app)
      .post('/orders')
      .set('Authorization', authHeader(user.id))
      .send({ items: [{ productId: 'p-1', quantity: 6, price: 25 }] });

    expect(res.status).toBeGreaterThanOrEqual(400);

    // Verify no partial state
    const { rows: orders } = await pool.query('SELECT * FROM orders WHERE user_id = $1', [user.id]);
    expect(orders).toHaveLength(0);

    const { rows: [product] } = await pool.query("SELECT stock FROM products WHERE id = 'p-1'");
    expect(product.stock).toBe(5); // unchanged
  });
});
```

**Why error-path tests catch more production bugs:**

1. **Happy paths are naturally tested.** Developers test them manually during development, QA tests them in staging, and users exercise them constantly. Bugs in happy paths get noticed quickly. Error paths are exercised rarely — a payment failure, a database timeout, a race condition — and often only surface under production load.

2. **Error paths have the most complex logic.** The happy path is usually straightforward: validate → process → return. The error path involves cleanup (rolling back transactions, reverting cache, undoing side effects), fallback behavior (retries, circuit breakers), and user-facing error formatting. Each of these is a separate thing that can break.

3. **Partial state is the worst kind of bug.** If an order is created but stock isn't deducted, or stock is deducted but the order fails, you have data inconsistency that's hard to detect and harder to fix. Transaction rollback tests (like Test 2) are the only reliable way to verify atomicity — you can't check this manually because the failure condition is hard to reproduce.

4. **Error responses are part of your API contract.** Clients depend on specific error codes and message formats to show appropriate UI. If your 400 response changes shape, frontend error handling breaks silently. Testing error response structure prevents this.

</details>

---

## Experience-Based Questions

These questions test real-world experience. Prepare by mapping them to your own projects and situations.

<details>
<summary>23. Tell me about a time you improved a team's testing strategy — what was the before state, what changes did you make (different test types, CI infrastructure, test data management), and what impact did it have on confidence and velocity?</summary>

**What the interviewer is looking for:** Evidence that you think about testing as a system, not just writing individual tests. They want to see that you can assess an existing strategy, identify gaps, prioritize improvements, and measure impact.

**Key points to hit:**

- Concrete description of the before state (what was broken or missing)
- Why you identified this as a problem (data: bug frequency, CI failures, developer complaints)
- What specific changes you made and why those changes over alternatives
- Measurable impact (CI time, bug escape rate, developer confidence)

**Suggested structure (STAR):**

1. **Situation:** Describe the team and codebase. What testing existed? What were the pain points?
2. **Task:** What specific problem were you solving? Was it your initiative or assigned?
3. **Action:** Walk through the changes step by step. Cover:
   - What test types you added or rebalanced (e.g., shifted from unit-heavy with mocked DBs to integration tests with real databases)
   - Infrastructure changes (introduced Testcontainers, set up CI parallelization, added caching)
   - Process changes (test data factories, test review guidelines, flaky test quarantine)
   - How you got buy-in from the team
4. **Result:** Quantify the impact. Examples: "CI time dropped from 20 min to 6 min," "Bug escape rate to production dropped by 40% over 3 months," "Developers started trusting the test suite enough to refactor confidently."

**Example outline to personalize:**

> Our service had ~200 unit tests that mocked the database. CI was fast (2 minutes) but we kept getting production bugs from SQL query issues — wrong JOINs, missing WHERE clauses, type mismatches. The mocked tests all passed but tested nothing real.
>
> I proposed shifting to integration tests with Testcontainers. Spent a week building the infrastructure: global setup for PostgreSQL containers, factory functions for test data, transaction-rollback isolation. Then I converted the 15 most critical repository tests from mocked to real-database tests.
>
> CI time went from 2 minutes to 5 minutes (acceptable), but we caught 3 existing query bugs during the migration. Over the next quarter, SQL-related production incidents dropped from ~2/month to 0. The team adopted the pattern for all new tests.

**What to avoid:**

- Don't describe just adding more tests. The interviewer wants strategic thinking — *which* tests and *why*.
- Don't skip the "before" state. The improvement only has context if the interviewer understands what was lacking.
- Don't claim zero bugs. Be honest about tradeoffs (e.g., "CI got slower but the confidence gain was worth it").

</details>

<details>
<summary>24. Describe a time you dealt with a persistent flaky test problem — what was the root cause, how did you diagnose it, and what systemic changes did you make to prevent similar flakiness?</summary>

**What the interviewer is looking for:** Systematic debugging skills applied to test infrastructure. They want to see that you don't just re-run CI and hope — you diagnose root causes and fix them structurally.

**Key points to hit:**

- How you identified the flaky tests (monitoring, CI data, team reports)
- Your diagnosis methodology (not trial-and-error, but systematic isolation)
- The root cause and why it only manifested in CI
- Structural fix, not a band-aid (e.g., not just increasing timeouts)
- Preventive measures so similar flakiness doesn't recur

**Suggested structure (STAR):**

1. **Situation:** "Our CI had a ~15% failure rate on the integration test suite. Developers were re-running pipelines 2-3 times per PR. Trust in the test suite was eroding."
2. **Task:** "I took ownership of diagnosing and fixing the flakiness problem."
3. **Action:** Walk through your debugging process:
   - **Identification:** How you found which specific tests were flaky (CI logs, JSON test reports, tracking re-runs)
   - **Isolation:** Running tests alone vs together, with and without parallelism, with shuffled order
   - **Root cause:** The specific technical cause (shared database state between parallel workers, timing-dependent assertions, hardcoded ports, test order dependence)
   - **Fix:** The concrete changes (transaction rollback isolation, polling helpers instead of `setTimeout`, dynamic port allocation, per-worker database schemas)
   - **Prevention:** Systemic changes (enabled `--sequence.shuffle` by default, added a quarantine process, set up flaky test tracking in CI)
4. **Result:** "CI failure rate dropped from ~15% to < 1%. Developers stopped needing to re-run pipelines. We saved roughly 30 minutes of developer time per PR."

**Example outline to personalize:**

> Three integration tests were failing intermittently — only in CI, never locally. I started by running the tests with `--maxWorkers=1` in CI — they passed, which pointed to a parallelism issue. Then I ran them with shuffled order — one test that depended on data from a previous test started failing consistently.
>
> Root cause: two test files shared a database without proper isolation. One file inserted a user with email "test@test.com", and another file expected to be the only user in the database. Locally they ran sequentially (same file order every time). In CI with 4 workers, they overlapped.
>
> Fix: I added `beforeEach` truncation to every integration test file and introduced factory functions that generate unique emails. I also enabled `--sequence.shuffle` in our vitest config so order-dependent tests would fail locally too, catching the problem before it reached CI.

**What to avoid:**

- Don't say "I just increased the timeout." That's a band-aid, not a fix.
- Don't skip the diagnosis process. The interviewer cares as much about *how* you found the problem as *what* the problem was.
- Don't make it sound easy. Flaky tests are genuinely hard to debug — showing the difficulty demonstrates real experience.

</details>

<details>
<summary>25. Tell me about a time a production bug slipped through your test suite — what kind of test would have caught it, why didn't you have that test, and what did you change afterward?</summary>

**What the interviewer is looking for:** Intellectual honesty about testing gaps, ability to do a root-cause analysis on your own processes (not just code), and evidence that you improve systems after failures.

**Key points to hit:**

- What the production bug was and its impact
- Why existing tests didn't catch it (specific gap in coverage or test design)
- What kind of test would have caught it (be specific — not "more tests" but "an integration test against real PostgreSQL that tested the JOIN with NULL foreign keys")
- Why that test didn't exist (honest assessment — time pressure, over-reliance on mocks, blind spot in test strategy)
- What you changed systemically (not just "added a test for this bug" but process/strategy changes)

**Suggested structure (STAR):**

1. **Situation:** Describe the bug and its production impact. Keep it concise — the interviewer cares more about your response than the bug details.
2. **Task:** You owned the investigation and the fix.
3. **Action:** Walk through:
   - The bug: what went wrong and why
   - The gap analysis: why didn't tests catch it?
   - The immediate fix: the regression test you added
   - The systemic fix: what you changed about the testing strategy to prevent the *category* of bug, not just this specific instance
4. **Result:** "Since adding [specific type of tests], we haven't had a [category] bug reach production."

**Example outline to personalize:**

> We had a query that returned order totals using a SUM with a JOIN. In production, orders with no items returned NULL instead of 0, which caused a downstream calculation to throw. Our tests all used mocked repositories that returned `{ total: 150 }` — they never tested the actual SQL.
>
> The specific test that would have caught it: an integration test against real PostgreSQL where we create an order with zero items and assert the total is 0 (or the response handles it gracefully).
>
> Why we didn't have it: we'd been writing unit tests with mocked repos for speed. The mock always returned valid data, so we never tested what the real query returns for edge cases.
>
> What I changed: I advocated for shifting our repository tests to use Testcontainers. I set up the infrastructure and wrote a testing guide for the team that included "always test empty/null cases for aggregate queries." We also added a rule: any production bug gets a regression test against real infrastructure, not a mocked test.

**What to avoid:**

- Don't blame others or external factors. Own the gap.
- Don't just say "we added a test." That's the minimum. The interviewer wants to hear about the systemic improvement.
- Don't pick a trivial bug. Choose one where the testing gap was genuinely interesting and the fix improved your testing strategy meaningfully.

</details>
