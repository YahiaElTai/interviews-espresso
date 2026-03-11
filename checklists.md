# Cross-Reference Checklists

Resources to validate question coverage *after* generating questions for a file. Not study material — run these against your questions to catch blind spots.

---

## Node.js Best Practices (goldbergyoni/nodebestpractices) — 102 practices

Check against: `01-core-stack/01-nodejs.md`, `01-core-stack/04-testing.md`, `01-core-stack/05-nestjs.md`, `03-infrastructure/08-authentication-and-security.md`, `03-infrastructure/01-docker-and-containers.md`, `04-production/01-observability-and-monitoring.md`, `04-production/04-performance-and-optimization.md`, `06-architecture/01-software-architecture.md`

| Section | Check Against | Key Practices to Ensure Coverage |
|---|---|---|
| §1 Project Architecture (6) | 06-architecture/01-software-architecture.md, 01-core-stack/05-nestjs.md | Component structure, 3-tier layering, config management |
| §2 Error Handling (12) | 01-core-stack/01-nodejs.md | Operational vs programmer errors, graceful shutdown, central error handling, unhandled promise rejections |
| §4 Testing (13) | 01-core-stack/04-testing.md | AAA pattern, component testing, test isolation, mock external services, 5 possible outcomes |
| §5 Going to Production (19) | 04-production/01-observability-and-monitoring.md, 04-production/04-performance-and-optimization.md | Statelessness, CPU utilization, memory monitoring, transaction IDs in logs, zero-downtime deploys |
| §6 Security (27) | 03-infrastructure/08-authentication-and-security.md | JWT blocklisting, ReDoS, brute-force prevention, secret management, query injection, payload limits |
| §8 Docker (15) | 03-infrastructure/01-docker-and-containers.md | Multi-stage builds, graceful shutdown in containers, secret handling, image scanning, cache optimization |

Skip: §3 (code style/linting — not interview material), §7 (WIP, only 2 items)

---

## JavaScript Runtime Quirks (from wtfjs + practical experience)

Check against: `01-core-stack/02-typescript.md`

Ensure a few questions cover JS quirks that cause real bugs (not trivia). Good framing: "What JavaScript runtime quirks does TypeScript NOT protect you from?"

- `typeof null === "object"` — affects type guards
- `NaN !== NaN` — use `Number.isNaN()`, not `=== NaN`
- `0.1 + 0.2 !== 0.3` — floating point in financial calculations
- `==` coercion traps (`[] == false` but `!![] === true`)
- `parseInt` gotchas (`parseInt("08")`, `parseInt(0.0000005)`)
- `var` hoisting vs `let`/`const` TDZ — still relevant in legacy code and closures in loops

---

## The Twelve-Factor App (12factor.net)

Check against: `06-architecture/03-microservices.md`, `06-architecture/01-software-architecture.md`

Short, authoritative methodology (12 pages). Ensure questions cover these factors — they come up in architecture discussions:

1. Codebase — one repo, many deploys
2. Dependencies — explicitly declare and isolate
3. Config — store in environment, never in code
4. Backing Services — treat as attached resources (DB, cache, queue)
5. Build, Release, Run — strict separation of stages
6. Processes — stateless, share-nothing
7. Port Binding — export services via port binding
8. Concurrency — scale out via process model
9. Disposability — fast startup, graceful shutdown
10. Dev/Prod Parity — keep environments as similar as possible
11. Logs — treat as event streams
12. Admin Processes — run admin tasks as one-off processes

---

## OWASP Cheat Sheet Series (cheatsheetseries.owasp.org)

Check against: `03-infrastructure/08-authentication-and-security.md`, `03-infrastructure/01-docker-and-containers.md`, `03-infrastructure/02-kubernetes.md`, `03-infrastructure/07-secret-management.md`, `04-production/01-observability-and-monitoring.md`

110 cheat sheets. Most relevant ones to cross-reference:

- Authentication, Authorization, Session Management → `03-infrastructure/08-authentication-and-security.md`
- Password Storage (bcrypt/scrypt best practices) → `03-infrastructure/08-authentication-and-security.md`
- JWT for Java (principles apply to Node.js too) → `03-infrastructure/08-authentication-and-security.md`
- REST Security, GraphQL Security, gRPC Security → `03-infrastructure/08-authentication-and-security.md`
- Input Validation, SQL Injection Prevention, XSS Prevention → `03-infrastructure/08-authentication-and-security.md`
- CSRF Prevention, Clickjacking Defense → `03-infrastructure/08-authentication-and-security.md`
- Secrets Management → `03-infrastructure/07-secret-management.md`
- Docker Security → `03-infrastructure/01-docker-and-containers.md`
- Kubernetes Security → `03-infrastructure/02-kubernetes.md`
- Content Security Policy, HTTP Headers → `03-infrastructure/08-authentication-and-security.md`
- Logging, Error Handling → `04-production/01-observability-and-monitoring.md`
- Multi-Tenant Security (relevant for commercetools-style SaaS) → `03-infrastructure/08-authentication-and-security.md`

---

## Microsoft Azure Cloud Design Patterns

Check against: `06-architecture/06-cloud-design-patterns.md`, `06-architecture/07-scaling-and-reliability.md`

40+ patterns, vendor-neutral despite Microsoft branding. Key patterns to ensure coverage:

**Reliability:** Circuit Breaker, Retry, Bulkhead, Health Endpoint Monitoring, Compensating Transaction, Saga
**Messaging:** Publisher/Subscriber, Competing Consumers, Queue-Based Load Leveling, Priority Queue, Claim Check, Sequential Convoy, Choreography
**Data:** Cache-Aside, CQRS, Event Sourcing, Materialized View, Sharding, Index Table
**Design:** Ambassador, Anti-Corruption Layer, Backends for Frontends, Gateway Aggregation/Offloading/Routing, Sidecar, Strangler Fig
**Scaling:** Deployment Stamps, Geode, Throttling, Rate Limiting, Static Content Hosting, Valet Key

---

## PostgreSQL Wiki "Don't Do This"

Check against: `02-data/02-postgresql.md`

19 antipatterns from the PG community. Key ones for questions:

- Don't use `SQL_ASCII` encoding
- Don't use `NOT IN` (NULL + performance trap)
- Don't use `BETWEEN` with timestamps (double-counting at midnight)
- Don't use `timestamp` without time zone — use `timestamptz`
- Don't use `char(n)` — use `text` with CHECK constraints
- Don't use `varchar(n)` by default — use `text` or unbounded `varchar`
- Don't use `money` type — use `integer` (cents) or `numeric`
- Don't use `serial` — use identity columns
- Don't use `rules` — use triggers
- Don't use `trust` authentication over TCP/IP

---

## Node.js Official Guides (nodejs.org)

Check against: `01-core-stack/01-nodejs.md`, `04-production/03-debugging-and-troubleshooting.md`

Key guides from nodejs.org to cross-reference:

**"Don't Block the Event Loop":**
- Computational complexity in callbacks (O(n²) examples)
- ReDoS (Regular Expression Denial of Service) — vulnerable patterns like `(a+)*`
- Sync API dangers (`crypto.pbkdf2Sync`, `zlib.inflateSync`, sync FS)
- JSON DoS (large `JSON.stringify` blocking)
- Solutions: partitioning with `setImmediate()`, offloading to worker threads
- Worker pool task partitioning (avoid large single FS reads)

**"Backpressuring in Streams":**
- What happens when data source produces faster than consumer can handle
- `.write()` return value and `drain` event
- `highWaterMark` configuration and its effect on memory
- `pipeline()` vs `.pipe()` — error handling and cleanup differences

**"Diagnostics" guides (memory leaks, CPU profiling):** → `01-core-stack/01-nodejs.md`, `04-production/03-debugging-and-troubleshooting.md`
- `--inspect` and Chrome DevTools for debugging
- Heap snapshots and heap timeline for memory leak detection
- CPU profiling and flame graph interpretation
- `process.memoryUsage()`, `v8.getHeapStatistics()`

**"Security Best Practices":**
- Prototype pollution prevention
- Child process execution risks (`exec` vs `execFile`)

---

## Refactoring.guru Design Patterns

Check against: `06-architecture/02-design-patterns-and-principles.md`

22 GoF patterns with excellent visual explanations. Ensure coverage of the practical ones:

**Creational:** Factory Method, Builder, Singleton (and why it's problematic)
**Structural:** Adapter, Decorator, Facade, Proxy
**Behavioral:** Observer, Strategy, Command, Chain of Responsibility, Iterator, State, Template Method

Skip for interviews: Abstract Factory, Bridge, Composite, Flyweight, Memento, Visitor, Mediator (too academic, rarely asked)

---

## Martin Fowler's Blog (martinfowler.com)

Check against: `06-architecture/03-microservices.md`, `06-architecture/04-event-driven-architecture.md`, `06-architecture/07-scaling-and-reliability.md`, `06-architecture/01-software-architecture.md`, `06-architecture/02-design-patterns-and-principles.md`, `01-core-stack/04-testing.md`, `03-infrastructure/04-ci-cd.md`

Key articles to cross-reference — verify these specific concepts appear in questions:

| Article/Series | Check Against | Key Concepts to Verify |
|---|---|---|
| Microservices guide, Monolith First | 06-architecture/03-microservices.md | Monolith-first approach, premature decomposition risks, distributed monolith antipattern |
| Event-Driven, Event Sourcing, CQRS | 06-architecture/04-event-driven-architecture.md | Eventual consistency tradeoffs, event store vs message broker, when CQRS is overkill |
| Strangler Fig Application | 06-architecture/07-scaling-and-reliability.md, 06-architecture/03-microservices.md | Incremental migration strategy, routing layer, feature-by-feature extraction |
| Patterns of Enterprise Application Architecture | 06-architecture/01-software-architecture.md | Repository pattern, Unit of Work, Domain Model vs Transaction Script |
| Refactoring catalog | 06-architecture/02-design-patterns-and-principles.md | Code smells as refactoring triggers, extract method/class, when NOT to refactor |
| Testing articles (Test Pyramid) | 01-core-stack/04-testing.md | Pyramid vs trophy shape, integration test ROI, contract testing for microservices |
| Continuous Delivery articles | 03-infrastructure/04-ci-cd.md | Deployment vs release separation, feature toggles, trunk-based development |

---

## Microservices.io Patterns (Chris Richardson)

Check against: `06-architecture/03-microservices.md`, `06-architecture/04-event-driven-architecture.md`, `04-production/01-observability-and-monitoring.md`, `01-core-stack/04-testing.md`

Comprehensive pattern catalog. Key patterns to ensure coverage:

**Decomposition:** By business capability, by subdomain → `06-architecture/03-microservices.md`
**Data:** Database per service, Saga, CQRS, Event Sourcing, Transactional Outbox → `06-architecture/03-microservices.md`, `06-architecture/04-event-driven-architecture.md`
**Communication:** API Gateway, Backend for Frontend, Messaging, Idempotent Consumer → `06-architecture/03-microservices.md`, `06-architecture/04-event-driven-architecture.md`
**Deployment:** Service per container, Serverless, Service deployment platform → `06-architecture/03-microservices.md`
**Observability:** Log aggregation, Distributed tracing, Health check API, Application metrics → `04-production/01-observability-and-monitoring.md`
**Testing:** Consumer-driven contract test, Service component test → `01-core-stack/04-testing.md`
**Reliability:** Circuit Breaker → `06-architecture/03-microservices.md`
**Discovery:** Client-side discovery, Server-side discovery, Service registry → `06-architecture/03-microservices.md`

---

## Kent C. Dodds Testing Trophy

Check against: `01-core-stack/04-testing.md`

Ensure questions cover this testing philosophy:

- **Static** (bottom) — linting + type checking (ESLint, TypeScript)
- **Unit** — isolated functions/classes, fast, cheap
- **Integration** (largest section) — multiple units together, highest ROI
- **E2E** (top) — full workflows, minimal mocking, most confidence per test but slowest

Key principle: "The more your tests resemble the way your software is used, the more confidence they can give you."

---

## web.dev Core Web Vitals

Check against: `08-frontend/02-browser-and-web-performance.md`

Ensure frontend performance questions cover:

- LCP (Largest Contentful Paint) — loading performance
- CLS (Cumulative Layout Shift) — visual stability
- INP (Interaction to Next Paint) — responsiveness (replaced FID)
- Lab vs field data differences
- Optimization strategies for each metric
- Font, image, and third-party script impact

---

## System Design Primer (donnemartin/system-design-primer)

Check against: `07-system-design/01-fundamentals.md`, `02-data/03-redis.md`, `06-architecture/05-message-queues-and-streaming.md`, `02-data/01-databases.md`

Ensure the fundamentals file covers all building blocks from this primer:

- DNS (how it works, as a system design component) → `07-system-design/01-fundamentals.md`
- CDNs (push vs pull, when to use) → `07-system-design/01-fundamentals.md`
- Load Balancers (L4 vs L7, algorithms, horizontal scaling) → `07-system-design/01-fundamentals.md`
- Reverse Proxy (vs load balancer, when to use) → `07-system-design/01-fundamentals.md`
- Caching strategies (cache-aside, write-through, write-behind, refresh-ahead) → `07-system-design/01-fundamentals.md`, `02-data/03-redis.md`
- Caching layers (client, CDN, web server, database, application) → `07-system-design/01-fundamentals.md`, `02-data/03-redis.md`
- Asynchronism (message queues, task queues, back pressure) → `07-system-design/01-fundamentals.md`, `06-architecture/05-message-queues-and-streaming.md`
- Communication protocols (HTTP, TCP, UDP, RPC, REST comparison) → `07-system-design/01-fundamentals.md`
- Consistency patterns (weak, eventual, strong) → `07-system-design/01-fundamentals.md`
- Availability patterns (active-passive, active-active, replication) → `07-system-design/01-fundamentals.md`
- Back-of-the-envelope calculations (powers of two, latency numbers) → `07-system-design/01-fundamentals.md`
- Database scaling (federation, sharding, denormalization, SQL tuning) → `07-system-design/01-fundamentals.md`, `02-data/01-databases.md`
- Also has Anki flashcards — useful for memorization phase

---

## Google API Design Guide (cloud.google.com/apis/design)

Check against: `01-core-stack/03-api-design.md`

Industry-standard reference for resource-oriented API design. Ensure coverage of:

- Resource-oriented design: resources, collections, and standard methods (List, Get, Create, Update, Delete)
- Custom methods: when standard CRUD isn't enough, naming convention (`:verb` suffix)
- Error model: canonical error codes, error details, mapping HTTP status codes to gRPC codes
- Field masks for partial updates (vs PUT full replacement vs JSON Merge Patch)
- Pagination: `page_token` + `page_size` pattern, total count tradeoffs
- Filtering and ordering: filter expressions, sort semantics
- Long-running operations: polling pattern, operation resource
- Naming conventions: resource names, field names (camelCase vs snake_case), pluralization
- Output-only vs input-only vs immutable fields
- Request validation: required vs optional, default values

---

## Kubernetes Official Concepts (k8s.io/docs/concepts)

Check against: `03-infrastructure/02-kubernetes.md`

Structured reference for K8s internals. Ensure questions cover these concepts beyond the basics:

- Pod lifecycle: Pending → Running → Succeeded/Failed, termination flow (preStop → SIGTERM → grace period → SIGKILL)
- Container lifecycle hooks: postStart, preStop — why preStop matters for zero-downtime
- API access control flow: Authentication → Authorization (RBAC) → Admission Control (webhooks, OPA)
- Garbage collection: owner references, finalizers, cascading vs orphan deletion
- Pod topology spread constraints — distributing pods across zones/nodes
- Init containers: ordering guarantees, use cases (DB migration, config fetching)
- Ephemeral containers for debugging (kubectl debug)
- QoS classes: Guaranteed, Burstable, BestEffort — how requests/limits determine class and eviction priority
- Pod Disruption Budgets: voluntary vs involuntary disruptions, interaction with cluster upgrades
- Downward API: exposing pod metadata to containers via env vars or volume mounts
