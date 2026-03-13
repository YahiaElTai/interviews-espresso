# API Design

> **26 questions** — 16 theory, 6 practical, 4 experience

- REST fundamentals: resource modeling, HTTP method semantics, status code selection, URI naming conventions, field naming (camelCase vs snake_case), pluralization rules
- API authentication patterns at the design level: API keys, Bearer tokens, Authorization header conventions, 401 vs 403 semantics
- Partial update strategies: PUT full replacement vs PATCH vs field masks vs JSON Merge Patch, output-only and immutable field semantics
- Idempotency keys, retry safety, and idempotency implementation patterns
- API versioning strategies (URL path, header, content negotiation) and backwards compatibility
- Pagination: offset, cursor, keyset — tradeoffs and implementation, total count tradeoffs
- Error response design: structure, codes, correlation IDs across paradigms
- Request validation: schema validation (Zod, Joi), input sanitization, where validation lives in the request lifecycle, validation error response format
- GraphQL fundamentals: schema-first design, resolvers, the N+1 problem, DataLoader batching
- GraphQL at scale: query cost analysis (depth limiting, complexity scoring), federation vs schema stitching, persisted queries
- gRPC: protobufs, HTTP/2, streaming modes, service-to-service use cases
- Server-Sent Events (SSE) vs WebSockets for real-time push
- API gateways: routing, auth offloading, protocol translation, rate limiting — and when a BFF (Backend for Frontend) is the better pattern
- Rate limiting at the API layer: token bucket, sliding window, per-user/per-IP/per-endpoint dimensions, rate limit response headers
- Choosing between REST, GraphQL, gRPC, and SSE: decision framework based on client types, team size, latency needs, and evolution speed

---

## Foundational

<details>
<summary>1. How should you model REST APIs around resources rather than actions — what are the rules for URI naming conventions, HTTP method semantics (GET/POST/PUT/PATCH/DELETE), and status code selection, and why does getting these right matter for API predictability and cacheability?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>2. When should you choose REST vs GraphQL vs gRPC vs SSE — what's the decision framework based on client types (browser, mobile, internal service), team size, latency requirements, and how quickly the API needs to evolve?</summary>

<!-- Answer will be added later -->

</details>

## REST — Concepts

<details>
<summary>3. Why is idempotency critical for API safety — what makes an API call idempotent, which HTTP methods are naturally idempotent, why do POST and PATCH need explicit idempotency keys, and what implementation patterns prevent duplicate processing on retries?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>4. Why is API versioning necessary and what are the tradeoffs of URL path versioning vs custom headers vs content negotiation — how do you maintain backwards compatibility as an API evolves, and when is versioning overkill?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>5. Why do pagination approaches differ and what are the tradeoffs — when is offset pagination acceptable, why does cursor-based pagination scale better for large datasets, how does keyset pagination work, and what breaks when you paginate mutable data?</summary>

<!-- Answer will be added later -->

</details><details>
<summary>6. Why do partial update strategies matter for API usability and backward compatibility — what are the tradeoffs between PUT (full replacement), PATCH with JSON Merge Patch, and PATCH with field masks, how do output-only and immutable fields affect each approach, and when is one strategy clearly better than the others?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>7. Why does error response design matter across API paradigms — what should a well-designed error response contain (structure, error codes, human-readable messages, correlation IDs), and how do error conventions differ between REST status codes, GraphQL errors, and gRPC status codes?</summary>

<!-- Answer will be added later -->

</details>## GraphQL — Concepts

<details>
<summary>8. Why does GraphQL use a schema and resolver architecture instead of predefined endpoints like REST — how do type definitions, resolvers, and the execution engine work together, and what are the common schema design mistakes that hurt performance?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>9. Why does the N+1 problem emerge naturally in GraphQL resolver execution, how does DataLoader solve it through batching and caching, and what are the limits of DataLoader — when does it not help?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>10. Why does GraphQL need protection against abusive queries — how does query cost analysis work (depth limiting, complexity scoring, field-level cost), what are persisted queries and how do they improve both security and performance, and what are the tradeoffs between federation and schema stitching for composing multiple GraphQL services?</summary>

<!-- Answer will be added later -->

</details>

## gRPC & Real-Time — Concepts

<details>
<summary>11. Why was gRPC built on HTTP/2 with Protocol Buffers instead of JSON over HTTP/1.1 — what advantages does this give for service-to-service communication (strong typing, streaming, performance), what are the four streaming modes (unary, server, client, bidirectional), and when is gRPC a poor fit?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>12. When should you use Server-Sent Events vs WebSockets for real-time communication — what are the architectural differences (unidirectional vs bidirectional, HTTP/1.1 compatibility, automatic reconnection), what are the tradeoffs in scaling and infrastructure support, and when is SSE sufficient vs when do you need full WebSockets?</summary>

<!-- Answer will be added later -->

</details>

## Cross-Cutting — Concepts

<details>
<summary>13. What problems do API gateways solve and when is the BFF (Backend for Frontend) pattern the better choice — how do gateways handle routing, authentication, rate limiting, and protocol translation, and what goes wrong when the gateway becomes a bottleneck or takes on too much business logic?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>14. Why does rate limiting at the API layer need different strategies for different dimensions — how do per-user, per-IP, and per-endpoint rate limits serve different purposes, what algorithms work for each (token bucket, sliding window), and where should rate limiting live (gateway vs application)?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>15. How do API authentication mechanisms differ at the design level — when should you use API keys vs Bearer tokens vs OAuth2, what are the conventions for the Authorization header, and how do you decide between 401 Unauthorized and 403 Forbidden for different failure scenarios?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>16. Why does request validation need to happen at a specific point in the request lifecycle — where should schema validation (using tools like Zod or Joi) run relative to authentication and routing, what should validation error responses look like, and what are the risks of missing input sanitization?</summary>

<!-- Answer will be added later -->

</details>

## Practical — REST Patterns

<details>
<summary>17. Design a RESTful API for an order management system — show the resource hierarchy (orders, line items, payments), URI structure, HTTP methods for each operation, appropriate status codes for success and failure cases, and explain why you modeled sub-resources the way you did</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>18. Implement idempotency for a payment creation endpoint — show the idempotency key flow (header, storage, deduplication), how to handle concurrent requests with the same key, what storage backend you'd use, and what the client retry experience looks like</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>19. Implement cursor-based pagination for a list endpoint that returns timestamped records — show the API contract (request params, response shape with next cursor), the database query approach, and explain why the cursor is opaque, what encoding to use, and how this handles records inserted between pages</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>20. Design a consistent error response format and implement it across multiple endpoints — show the JSON structure (error code, message, details, correlation ID), how you map domain errors to HTTP status codes, and how correlation IDs flow from client through gateway to backend for debugging</summary>

<!-- Answer will be added later -->

</details>

## Practical — GraphQL, gRPC & Real-Time

<details>
<summary>21. Build a GraphQL API for a blog (posts, authors, comments) using a schema-first approach — show the schema definition, implement resolvers with DataLoader to prevent N+1 queries, and demonstrate how a single query that fetches posts with their authors and comments would execute without vs with DataLoader</summary>

<!-- Answer will be added later -->

</details>## Practical — Production & Cross-Cutting

<details>
<summary>22. Implement rate limiting for a REST API with different limits per endpoint and per authenticated user — show the middleware code using a sliding window or token bucket approach with Redis as the backing store, the response headers (X-RateLimit-Limit, X-RateLimit-Remaining, Retry-After), and what the client experience looks like when they hit the limit</summary>

<!-- Answer will be added later -->

</details>---

## Experience-Based Questions

These questions test real-world experience. Prepare by mapping them to your own projects and situations.

<details>
<summary>23. Tell me about a time you designed an API from scratch for a new product or feature — what design decisions did you make, what tradeoffs did you face, and what would you change in hindsight?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>24. Describe a time you dealt with API backwards compatibility issues — what was breaking, how did you manage the migration, and what did you learn about API evolution?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>25. Tell me about a time you had to choose between REST and GraphQL (or another paradigm) for a project — what factors drove the decision, and how did it work out?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>26. Describe a production incident caused by an API design issue (missing idempotency, poor error handling, pagination problems, or rate limiting gaps) — what went wrong, how did you fix it, and what design changes did you make to prevent it from recurring?</summary>

<!-- Answer framework will be added later -->

</details>
