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

**Resource modeling** means URIs represent nouns (things), not verbs (actions). The HTTP method is the verb.

Bad: `POST /createOrder`, `GET /getUsers`
Good: `POST /orders`, `GET /users`

**URI naming rules:**

- Use plural nouns: `/orders`, `/users`, `/products`
- Nest sub-resources under parents: `/orders/{orderId}/line-items`
- Use kebab-case for multi-word paths: `/line-items`, not `/lineItems`
- Keep URIs shallow — max 2-3 levels deep. Beyond that, promote the sub-resource to a top-level resource with a filter: `/line-items?orderId=123`
- No trailing slashes, no file extensions

**HTTP method semantics:**

| Method | Purpose | Idempotent | Safe | Request Body |
|--------|---------|-----------|------|-------------|
| GET | Read resource(s) | Yes | Yes | No |
| POST | Create resource / trigger action | No | No | Yes |
| PUT | Full replacement of resource | Yes | No | Yes |
| PATCH | Partial update | No* | No | Yes |
| DELETE | Remove resource | Yes | No | No |

*PATCH can be made idempotent with field masks or JSON Merge Patch, but the spec doesn't guarantee it.

**Status code selection:**

- `200 OK` — successful GET, PUT, PATCH
- `201 Created` — successful POST that creates a resource (include `Location` header)
- `204 No Content` — successful DELETE or update with no response body
- `400 Bad Request` — validation failure, malformed input
- `401 Unauthorized` — missing or invalid authentication
- `403 Forbidden` — authenticated but not authorized
- `404 Not Found` — resource doesn't exist
- `409 Conflict` — state conflict (duplicate, version mismatch)
- `422 Unprocessable Entity` — valid JSON but semantically wrong
- `429 Too Many Requests` — rate limited

**Why it matters:** When an API follows these conventions, developers can predict behavior without reading docs. GET is always safe to retry. PUT is always idempotent. Caches (CDNs, browsers, proxies) can automatically cache GET responses and invalidate on POST/PUT/DELETE because the semantics are standardized. Breaking these conventions — like making a GET that mutates state — breaks caching and confuses every HTTP-aware tool in the chain.

</details>

<details>
<summary>2. When should you choose REST vs GraphQL vs gRPC vs SSE — what's the decision framework based on client types (browser, mobile, internal service), team size, latency requirements, and how quickly the API needs to evolve?</summary>

**Decision framework:**

| Factor | REST | GraphQL | gRPC | SSE |
|--------|------|---------|------|-----|
| **Client types** | Any (universal) | Browser, mobile (flexible queries) | Service-to-service | Browser (unidirectional push) |
| **Team size** | Small teams, simple APIs | Larger teams, multiple client teams | Backend teams with strong typing needs | Any |
| **Latency** | Good (cacheable) | Moderate (no HTTP caching) | Lowest (binary, HTTP/2 multiplexing) | Good for push |
| **Evolution speed** | Versioning needed | Additive by default | Proto versioning | N/A (transport) |
| **Tooling maturity** | Highest | High (Apollo, etc.) | Growing (less browser support) | Built into browsers |

**Choose REST when:**
- You have a simple, resource-oriented domain
- You need HTTP caching (CDN, browser cache)
- Your clients are diverse and you want maximum compatibility
- Small team that doesn't need the complexity of GraphQL

**Choose GraphQL when:**
- Multiple client types need different data shapes from the same API (mobile wants less data, web wants more)
- Frontend teams want to iterate on data requirements without backend changes
- Your domain has deeply nested, interconnected entities
- You're aggregating data from multiple backend services into one API

**Choose gRPC when:**
- Internal service-to-service communication where latency matters
- You need streaming (real-time data feeds, log streaming)
- Strong typing and code generation across multiple languages is valuable
- You control both ends of the connection (not browser-facing)

**Choose SSE when:**
- You need server-to-client push but the client doesn't need to send data back (notifications, live feeds, progress updates)
- You want simpler infrastructure than WebSockets (works over HTTP/1.1, auto-reconnect built in)
- Use WebSockets instead if you need bidirectional communication (chat, collaborative editing)

**Combinations are normal.** A typical architecture might use REST for public APIs, gRPC for internal service mesh, GraphQL as a BFF aggregation layer, and SSE for real-time notifications. The choice isn't mutually exclusive.

</details>

## REST — Concepts

<details>
<summary>3. Why is idempotency critical for API safety — what makes an API call idempotent, which HTTP methods are naturally idempotent, why do POST and PATCH need explicit idempotency keys, and what implementation patterns prevent duplicate processing on retries?</summary>

An API call is **idempotent** if making it once produces the same result as making it N times. This is critical because networks are unreliable — clients will retry when they don't get a response, and without idempotency, retries cause duplicate side effects (double charges, duplicate orders).

**Naturally idempotent methods:**

- **GET** — reading doesn't change state
- **PUT** — "set this resource to exactly this state" — doing it twice gives the same result
- **DELETE** — deleting an already-deleted resource is a no-op (return 204 or 404, both are fine)

**Not naturally idempotent:**

- **POST** — "create a new resource" — retrying creates duplicates
- **PATCH** — depends on the operation. `{"balance": 100}` (set) is idempotent. `{"balance": "+10"}` (increment) is not.

**Idempotency key pattern** for POST/PATCH:

1. Client generates a unique key (UUID) and sends it in a header: `Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000`
2. Server checks if this key has been seen before (lookup in Redis or database)
3. If new: process the request, store the key + response
4. If seen: return the stored response without re-processing
5. Key expires after a TTL (typically 24-48 hours)

**Handling concurrent requests with the same key:**

The tricky case is two requests arriving simultaneously with the same idempotency key. Use an atomic lock (Redis `SET NX` or database `INSERT ... ON CONFLICT`) — the first request acquires the lock and processes; the second sees the lock and either waits or returns 409 Conflict.

```typescript
// Simplified idempotency middleware pattern
async function idempotencyMiddleware(req: Request, res: Response, next: NextFunction) {
  const key = req.headers['idempotency-key'] as string;
  if (!key) return next(); // no key = no idempotency guarantee

  const cached = await redis.get(`idempotency:${key}`);
  if (cached) {
    const { statusCode, body } = JSON.parse(cached);
    return res.status(statusCode).json(body);
  }

  // Acquire lock to prevent concurrent duplicates
  const locked = await redis.set(`idempotency-lock:${key}`, '1', 'NX', 'EX', 30);
  if (!locked) return res.status(409).json({ error: 'Request in progress' });

  // Capture the response to store it
  const originalJson = res.json.bind(res);
  res.json = (body: any) => {
    redis.set(`idempotency:${key}`, JSON.stringify({ statusCode: res.statusCode, body }), 'EX', 86400);
    redis.del(`idempotency-lock:${key}`);
    return originalJson(body);
  };

  next();
}
```

**Key design decisions:** Store the full response (not just "seen"), so retried requests get back exactly what the first request returned. This matters for payment APIs — the client needs the payment ID even on a retry.

</details>

<details>
<summary>4. Why is API versioning necessary and what are the tradeoffs of URL path versioning vs custom headers vs content negotiation — how do you maintain backwards compatibility as an API evolves, and when is versioning overkill?</summary>

APIs are contracts. Once clients depend on a response shape, changing it breaks them. Versioning gives you a way to evolve the API without breaking existing consumers.

**Three main strategies:**

**1. URL path versioning** (`/v1/orders`, `/v2/orders`)
- Pros: Dead simple, visible, easy to route, easy to cache (different URLs = different cache entries)
- Cons: Clients must update URLs. Encourages "big bang" version bumps instead of incremental evolution. Duplicate routes across versions.
- Best for: Public APIs where simplicity and discoverability matter most. This is what most teams should use.

**2. Custom header versioning** (`X-API-Version: 2` or `Api-Version: 2`)
- Pros: Clean URIs, can version independently of resource structure
- Cons: Hidden — not visible in browser or logs without inspection. Harder to cache (requires `Vary` header). Easy for clients to forget.
- Best for: Internal APIs where you control the clients.

**3. Content negotiation** (`Accept: application/vnd.myapi.v2+json`)
- Pros: Most "RESTful" — uses HTTP content negotiation as designed. Can version at the representation level, not the resource level.
- Cons: Verbose, easy to get wrong, poor tooling support, confusing for most developers.
- Best for: Almost never in practice. Theoretically elegant, practically painful.

**Maintaining backwards compatibility:**

- **Additive changes are safe**: new fields, new endpoints, new optional parameters
- **Breaking changes require a version bump**: removing fields, renaming fields, changing field types, changing required/optional status, changing error formats
- **Deprecation flow**: announce deprecation, set a sunset date (use `Sunset` and `Deprecation` headers), monitor usage, remove after the deadline
- Use default values for new required fields to avoid forcing client updates

**When versioning is overkill:**

- Internal APIs with a single consumer you control — just coordinate deploys
- Early-stage products with no external consumers yet
- APIs behind a BFF layer that you own — the BFF absorbs changes

**Practical recommendation:** Start with URL path versioning (`/v1/`). Stay on v1 as long as possible by making additive-only changes. Only bump to v2 when you need a fundamentally different resource model. Most APIs never need v3.

</details>

<details>
<summary>5. Why do pagination approaches differ and what are the tradeoffs — when is offset pagination acceptable, why does cursor-based pagination scale better for large datasets, how does keyset pagination work, and what breaks when you paginate mutable data?</summary>

**Offset pagination** (`?offset=20&limit=10`)

- The database runs `SELECT ... LIMIT 10 OFFSET 20` — it must scan and discard the first 20 rows
- Pros: Simple to implement, supports "jump to page N"
- Cons: Performance degrades linearly with offset (offset 10,000 scans 10,000 rows). If data is inserted/deleted between pages, you get duplicates or missed items.
- Acceptable when: datasets are small (<10k rows), data doesn't change often, and you need page-number navigation (admin dashboards)

**Cursor-based pagination** (`?cursor=abc123&limit=10`)

- The cursor is an opaque token encoding the position (typically a base64-encoded value of the last item's sort key)
- Server decodes the cursor and uses it in a WHERE clause: `WHERE created_at < :cursor_value ORDER BY created_at DESC LIMIT 10`
- Pros: Constant performance regardless of position. Stable under inserts/deletes — you always continue from where you left off.
- Cons: No "jump to page N". Can't easily compute total count without a separate query. Cursor is tied to the sort order.

**Keyset pagination** is the underlying technique behind cursor-based pagination — using indexed column values (like `created_at` + `id` for uniqueness) in a WHERE clause instead of OFFSET. The cursor is just the encoded keyset values.

```typescript
// Keyset query — always uses an index, O(1) seek
const items = await db.query(`
  SELECT * FROM orders
  WHERE (created_at, id) < ($1, $2)
  ORDER BY created_at DESC, id DESC
  LIMIT $3
`, [cursorTimestamp, cursorId, limit]);
```

**What breaks with mutable data:**

- **Offset pagination**: If row 15 is deleted while you're on page 2 (offset=10), page 3 skips a row. If a row is inserted before your offset, you see a duplicate.
- **Cursor-based**: Handles this gracefully — you continue from the last seen item, so inserts/deletes before or after your position don't cause duplicates or gaps.

**Total count tradeoff:** `SELECT COUNT(*)` on large tables is expensive (full table scan in PostgreSQL). Options: return an approximate count, cache the count and refresh periodically, or use `hasNextPage` boolean instead (fetch limit+1 rows, return limit rows, if you got limit+1 then there's a next page).

**Recommendation:** Use cursor-based for any client-facing list API. Reserve offset for internal admin tools where jumping to page N is genuinely needed.

</details><details>
<summary>6. Why do partial update strategies matter for API usability and backward compatibility — what are the tradeoffs between PUT (full replacement), PATCH with JSON Merge Patch, and PATCH with field masks, how do output-only and immutable fields affect each approach, and when is one strategy clearly better than the others?</summary>

Partial updates determine how clients communicate "change this, leave that alone." The wrong strategy creates painful backwards compatibility problems when you add new fields.

**PUT (full replacement):**

Client sends the complete resource. Server replaces the stored resource entirely.

- Pros: Simple semantics — what you send is what gets stored. Idempotent by definition.
- Cons: Read-modify-write cycle required — client must GET, modify, PUT. Race conditions between concurrent writers. When you add a new field, old clients that don't know about it will send PUTs that erase it (because they send the full resource without the new field).
- Output-only fields (like `createdAt`, `id`): Server must silently ignore them in the PUT body, or clients have to include them correctly.

**PATCH with JSON Merge Patch** (RFC 7396):

Client sends only the fields to change. `{"name": "New Name"}` updates name, leaves everything else untouched.

```typescript
// Client sends: PATCH /orders/123
// Content-Type: application/merge-patch+json
{ "shippingAddress": { "city": "Berlin" } }
```

- Pros: Intuitive, easy for clients. Adding new fields is backwards-compatible — old clients simply don't send them.
- Cons: Cannot set a field to `null` vs removing it (both are `"field": null`). Cannot distinguish "don't update this nested object" from "replace this nested object entirely". Nested merging behavior can be surprising.
- Best for: Most CRUD APIs with flat or shallow resources.

**PATCH with field masks** (Google API style):

Client sends the new values AND a list of which fields to update: `{ updateMask: ["name", "address.city"], name: "New", address: { city: "Berlin" } }`

- Pros: Explicit about intent. Handles null correctly (field in mask + null value = clear it). No ambiguity with nested objects.
- Cons: More complex API contract. Clients must build the mask correctly.
- Best for: Complex resources with deep nesting, APIs where precision matters (Google Cloud APIs use this pattern extensively).

**Immutable fields** (like `type`, `currency` set at creation):

- With PUT: Server must validate they haven't changed and reject if they have
- With Merge Patch: Server silently ignores them
- With field masks: Server rejects if they appear in the update mask

**Recommendation:** Use JSON Merge Patch for most APIs — it covers 90% of cases with minimal complexity. Switch to field masks when you have deeply nested resources or need precise null semantics. Avoid PUT-as-update unless your resources are simple and concurrent writes aren't a concern.

</details>

<details>
<summary>7. Why does error response design matter across API paradigms — what should a well-designed error response contain (structure, error codes, human-readable messages, correlation IDs), and how do error conventions differ between REST status codes, GraphQL errors, and gRPC status codes?</summary>

Good error responses are the difference between "I can debug this in 5 minutes" and "I have no idea what went wrong." They matter for developer experience, debugging speed, and client-side error handling.

**A well-designed error response should contain:**

```json
{
  "error": {
    "code": "ORDER_ALREADY_SHIPPED",
    "message": "Cannot cancel order that has already been shipped",
    "details": [
      { "field": "orderId", "issue": "Order 456 has status SHIPPED" }
    ],
    "correlationId": "req-abc-123-def",
    "docs": "https://api.example.com/errors/ORDER_ALREADY_SHIPPED"
  }
}
```

- **Machine-readable error code**: `ORDER_ALREADY_SHIPPED`, not just the HTTP status. Clients switch on these codes for programmatic handling.
- **Human-readable message**: Explains what went wrong in plain language. Never expose internal details (stack traces, SQL errors).
- **Details array**: For validation errors, lists each field and its issue. Clients can map these to form fields.
- **Correlation ID**: Unique request identifier that appears in logs, traces, and the response. When a client reports an error, you can find the exact request across all services.
- **Optional docs link**: Points to documentation explaining the error and resolution steps.

**How conventions differ across paradigms:**

**REST** uses HTTP status codes as the primary error signal:
- Status code tells you the category (4xx client error, 5xx server error)
- Response body provides the specific error code and details
- Caches and proxies understand status codes automatically

**GraphQL** always returns HTTP 200 — errors live in the response body:
```json
{
  "data": { "order": null },
  "errors": [
    {
      "message": "Order not found",
      "path": ["order"],
      "extensions": { "code": "NOT_FOUND", "correlationId": "req-abc" }
    }
  ]
}
```
GraphQL can return partial data with partial errors — some fields resolve while others fail. The `extensions` object is where you put machine-readable codes. This makes generic HTTP error handling (status code checks) useless — clients must always parse the body.

**gRPC** has a fixed set of ~16 status codes (OK, NOT_FOUND, PERMISSION_DENIED, INTERNAL, etc.):
- Status code + status message in the response trailer
- Richer details via the `google.rpc.Status` proto with typed detail messages
- More constrained than REST (fewer codes) but well-defined across all languages

**Key principle:** Keep error formats consistent across your entire API. One structure, one set of conventions. Clients should never have to guess which error format a given endpoint uses.

</details>## GraphQL — Concepts

<details>
<summary>8. Why does GraphQL use a schema and resolver architecture instead of predefined endpoints like REST — how do type definitions, resolvers, and the execution engine work together, and what are the common schema design mistakes that hurt performance?</summary>

GraphQL decouples **what data the client wants** from **how the server fetches it**. Instead of the server deciding the response shape per endpoint, the client specifies exactly what fields it needs, and the server resolves them.

**How the pieces work together:**

1. **Schema (type definitions)** defines the data graph — types, fields, relationships, and entry points (Query, Mutation, Subscription):
```graphql
type Post {
  id: ID!
  title: String!
  author: Author!
  comments: [Comment!]!
}

type Query {
  posts(limit: Int): [Post!]!
}
```

2. **Resolvers** are functions that fetch data for each field. Every field in the schema has a resolver (or uses a default that reads the property from the parent object):
```typescript
const resolvers = {
  Query: {
    posts: (_, { limit }) => db.posts.findMany({ take: limit }),
  },
  Post: {
    author: (post) => db.users.findById(post.authorId),
    comments: (post) => db.comments.findByPostId(post.id),
  },
};
```

3. **Execution engine** receives a query, validates it against the schema, then walks the query tree calling resolvers depth-first. Each resolver receives the parent's result, so data flows down the tree.

**Common schema design mistakes:**

- **Exposing database structure directly**: The schema should model the domain, not your tables. Don't expose join tables or internal IDs that clients don't need.
- **Deeply nested types without cost controls**: `posts { author { posts { author { ... } } } }` creates infinite nesting. Without depth limits, a single query can bring down the server.
- **Missing pagination on list fields**: `comments: [Comment!]!` with no pagination returns unbounded results. Always add `(first: Int, after: String)` arguments to list fields.
- **Fat resolvers on frequently-queried fields**: If `author` on Post triggers a DB call, and you fetch 50 posts, you get 50 DB calls (the N+1 problem — covered in question 9).
- **Overly granular mutations**: Don't mirror REST CRUD. Design mutations around business operations: `placeOrder` instead of `createOrder` + `addLineItem` + `setShippingAddress`.

</details>

<details>
<summary>9. Why does the N+1 problem emerge naturally in GraphQL resolver execution, how does DataLoader solve it through batching and caching, and what are the limits of DataLoader — when does it not help?</summary>

**Why N+1 happens naturally in GraphQL:**

GraphQL resolves fields independently. When you query `posts { author { name } }` and get 50 posts, the execution engine calls the `author` resolver once per post — that's 1 query for posts + 50 individual queries for authors = 51 queries. Each resolver doesn't know about the others running in parallel at the same level.

**How DataLoader solves it:**

DataLoader collects all individual `load(id)` calls made during a single tick of the event loop and batches them into one `loadMany(ids)` call.

```typescript
import DataLoader from 'dataloader';

// Batch function: receives array of IDs, returns array of results in same order
const authorLoader = new DataLoader<string, Author>(async (authorIds) => {
  const authors = await db.users.findMany({ where: { id: { in: authorIds } } });
  // Must return results in the same order as the input IDs
  const authorMap = new Map(authors.map(a => [a.id, a]));
  return authorIds.map(id => authorMap.get(id) ?? new Error(`Author ${id} not found`));
});

// Resolver now uses the loader instead of direct DB call
const resolvers = {
  Post: {
    author: (post) => authorLoader.load(post.authorId),
  },
};
```

Instead of 50 queries, DataLoader batches all 50 author IDs into a single `SELECT * FROM users WHERE id IN (...)` query. Result: 2 queries instead of 51.

**DataLoader also caches within a request** — if two posts share the same author, the author is fetched once and the cached result is returned for the second.

**Critical rule:** Create a new DataLoader instance per request. The cache is request-scoped. Sharing a DataLoader across requests means stale data and potential cross-user data leaks.

**When DataLoader doesn't help:**

- **Non-ID-based lookups**: DataLoader batches by key. If your resolver does a complex query with filters, joins, or aggregations, it can't be batched by simply collecting IDs.
- **Mutations**: DataLoader's caching can return stale data after a mutation within the same request.
- **Cross-service calls with no batch API**: If the downstream service only supports fetching one item at a time (no batch endpoint), DataLoader can't reduce the number of network calls — it just delays them.
- **Already-joined data**: If you pre-fetch related data in the parent resolver (eager loading), DataLoader adds unnecessary complexity.

</details>

<details>
<summary>10. Why does GraphQL need protection against abusive queries — how does query cost analysis work (depth limiting, complexity scoring, field-level cost), what are persisted queries and how do they improve both security and performance, and what are the tradeoffs between federation and schema stitching for composing multiple GraphQL services?</summary>

GraphQL's flexibility is also its vulnerability. A client can craft a single query that fetches deeply nested data, triggers thousands of resolver calls, and exhausts server resources. Unlike REST where each endpoint has a predictable cost, a GraphQL query's cost depends entirely on its shape.

**Protection mechanisms:**

**1. Depth limiting** — Reject queries beyond a maximum nesting depth (typically 5-10 levels). Prevents recursive queries like `user { friends { friends { friends { ... } } } }`. Simple to implement but coarse — a shallow query on an expensive field passes, while a deep but cheap query gets rejected.

**2. Complexity scoring** — Assign a cost to each field and sum them up before execution:
- Scalar fields: cost 1
- Object fields: cost 1 + children's cost
- List fields: cost multiplied by expected/max page size

```typescript
// Example: posts(first: 50) { author { name } }
// posts field: 1 * 50 (list multiplier) = 50
// author field: 1 * 50 = 50
// name field: 1 * 50 = 50
// Total: 150 — compare against max allowed (e.g., 1000)
```

Reject the query before execution if total cost exceeds the budget. This is more precise than depth limiting.

**3. Field-level cost annotations** — Some fields are inherently expensive (aggregations, full-text search). Annotate them with higher costs in the schema so the complexity calculator weights them properly.

**Persisted queries:**

Instead of sending the full query string, the client sends a hash. The server looks up the pre-registered query by hash and executes it.

- **Security**: Blocks arbitrary queries — only pre-approved queries run. Eliminates injection and abuse surface entirely.
- **Performance**: Skips parsing and validation (already done at registration time). Reduces network payload (hash vs full query string, especially for mobile).
- **Tradeoff**: Requires a build step to extract and register queries. Less flexible for development (use allowlisting in production, open in dev).

**Federation vs schema stitching:**

Both compose multiple GraphQL services into a unified API.

**Schema stitching** (older approach):
- A gateway fetches schemas from services and merges them at runtime
- The gateway handles cross-service relationships with custom resolvers
- Pros: Simple for small service counts
- Cons: Gateway becomes complex and fragile. Cross-service type resolution is manual. Tight coupling between gateway and services.

**Federation** (Apollo Federation model):
- Each service owns its types and declares how they extend types from other services using `@key` directives
- A router/gateway composes services using a query plan — it knows which service owns which fields
- Pros: Services are autonomous — they declare their own relationships. No custom gateway logic. Better separation of concerns.
- Cons: More upfront setup. Requires federation-aware libraries. Query planning adds latency. Debugging cross-service queries is harder.

**Recommendation:** Federation for any multi-team setup where services own their domains. Schema stitching only for simple, few-service cases or migration stepping stones.

</details>

## gRPC & Real-Time — Concepts

<details>
<summary>11. Why was gRPC built on HTTP/2 with Protocol Buffers instead of JSON over HTTP/1.1 — what advantages does this give for service-to-service communication (strong typing, streaming, performance), what are the four streaming modes (unary, server, client, bidirectional), and when is gRPC a poor fit?</summary>

gRPC's two foundational choices — HTTP/2 and Protocol Buffers — each solve specific problems that JSON over HTTP/1.1 can't.

**Protocol Buffers (protobuf):**

- Binary serialization format — 3-10x smaller payloads than JSON, faster to serialize/deserialize
- Schema-defined (`.proto` files) with strong typing and code generation for every major language
- Backwards-compatible evolution: add fields without breaking existing clients (field numbers are stable)

```protobuf
syntax = "proto3";

service OrderService {
  rpc GetOrder (GetOrderRequest) returns (Order);
  rpc StreamOrderUpdates (GetOrderRequest) returns (stream OrderUpdate);
}

message GetOrderRequest {
  string order_id = 1;
}

message Order {
  string id = 1;
  string status = 2;
  repeated LineItem items = 3;
}
```

**HTTP/2 advantages:**

- **Multiplexing**: Multiple RPC calls share a single TCP connection without head-of-line blocking — no need for connection pooling
- **Header compression** (HPACK): Reduces overhead for repeated headers across calls
- **Streaming**: Native support for long-lived bidirectional streams over a single connection
- **Flow control**: Per-stream backpressure prevents fast producers from overwhelming slow consumers

**The four streaming modes:**

| Mode | Client sends | Server sends | Use case |
|------|-------------|-------------|----------|
| **Unary** | 1 message | 1 message | Standard request/response (like REST) |
| **Server streaming** | 1 message | N messages | Live feed, query results streamed as they load |
| **Client streaming** | N messages | 1 message | File upload, telemetry batch |
| **Bidirectional streaming** | N messages | N messages | Chat, real-time collaboration, log tailing |

**When gRPC is a poor fit:**

- **Browser clients**: Browsers can't speak gRPC natively (no HTTP/2 trailer support). gRPC-Web exists but requires a proxy and only supports unary + server streaming.
- **Human debugging**: Binary protobuf payloads aren't readable in logs, browser dev tools, or `curl`. JSON APIs are far easier to inspect and test ad-hoc.
- **Simple CRUD with few clients**: The toolchain overhead (protoc, code generation, proto management) isn't worth it for a straightforward REST API with one or two consumers.
- **Public APIs**: External developers expect REST or GraphQL. Protobuf schemas and code generation are friction for third-party integrations.
- **Ecosystems without good gRPC support**: Some languages and platforms have immature gRPC libraries.

</details>

<details>
<summary>12. When should you use Server-Sent Events vs WebSockets for real-time communication — what are the architectural differences (unidirectional vs bidirectional, HTTP/1.1 compatibility, automatic reconnection), what are the tradeoffs in scaling and infrastructure support, and when is SSE sufficient vs when do you need full WebSockets?</summary>

**Architectural differences:**

| Aspect | SSE | WebSockets |
|--------|-----|------------|
| **Direction** | Server → Client only | Bidirectional |
| **Protocol** | HTTP/1.1 (regular GET with `text/event-stream`) | Starts as HTTP, upgrades to `ws://` protocol |
| **Reconnection** | Built-in — browser auto-reconnects with `Last-Event-ID` | Manual — you implement reconnection logic yourself |
| **Data format** | Text only (UTF-8) | Text and binary (images, audio, protobuf) |
| **Multiplexing** | Each SSE connection is a separate HTTP request | Single connection, custom message framing |
| **Browser support** | Native `EventSource` API, works everywhere | Native `WebSocket` API, works everywhere |

**SSE implementation is trivial:**

```typescript
// Server
app.get('/events', (req, res) => {
  res.setHeader('Content-Type', 'text/event-stream');
  res.setHeader('Cache-Control', 'no-cache');
  res.setHeader('Connection', 'keep-alive');

  const send = (event: string, data: object) => {
    res.write(`event: ${event}\ndata: ${JSON.stringify(data)}\n\n`);
  };

  send('orderUpdate', { orderId: '123', status: 'shipped' });

  // Client reconnects with Last-Event-ID header automatically
  req.on('close', () => { /* cleanup */ });
});

// Client (browser)
const source = new EventSource('/events');
source.addEventListener('orderUpdate', (e) => {
  const data = JSON.parse(e.data);
});
```

**Scaling tradeoffs:**

- **SSE** works through any HTTP infrastructure — load balancers, CDNs, proxies, API gateways — with zero special configuration. It's just a long-lived HTTP response.
- **WebSockets** require sticky sessions or a pub/sub layer (Redis) because the connection is stateful. Many proxies and load balancers need explicit WebSocket support. Cloud providers sometimes have lower connection limits for WebSockets.
- Both hold open connections, so connection count per server is the scaling bottleneck for both. SSE is slightly lighter (no frame parsing overhead).

**When SSE is sufficient:**

- Notifications, live feeds, dashboards, progress updates — anything where only the server pushes
- Streaming LLM responses (this is why ChatGPT and most AI APIs use SSE)
- When you want maximum infrastructure compatibility with minimal setup

**When you need WebSockets:**

- Chat applications — both sides send messages
- Collaborative editing — cursor positions, text changes flow both ways
- Gaming — player inputs and game state updates are bidirectional
- When you need binary data transfer (file chunks, audio streams)

**Practical recommendation:** Default to SSE for server push. It's simpler to implement, simpler to scale, and works through all existing infrastructure. Only reach for WebSockets when you genuinely need the client to send data back over the same connection — and even then, consider whether a regular POST + SSE would be simpler.

</details>

## Cross-Cutting — Concepts

<details>
<summary>13. What problems do API gateways solve and when is the BFF (Backend for Frontend) pattern the better choice — how do gateways handle routing, authentication, rate limiting, and protocol translation, and what goes wrong when the gateway becomes a bottleneck or takes on too much business logic?</summary>

An API gateway is a single entry point that sits between clients and your backend services. It handles cross-cutting concerns so individual services don't have to.

**What the gateway handles:**

- **Routing**: Maps external URLs to internal services. `/api/orders/*` → Order Service, `/api/users/*` → User Service. Clients see one domain; internally you have many services.
- **Authentication**: Validates JWTs or API keys once at the edge, then forwards authenticated identity (user ID, scopes) to backend services via headers. Services trust the gateway and skip re-validation.
- **Rate limiting**: Enforces per-client or per-endpoint limits before requests reach backend services (as covered in question 14).
- **Protocol translation**: Accepts REST from browsers, translates to gRPC for internal services. Clients don't need to speak gRPC.
- **Other**: TLS termination, request/response transformation, CORS handling, logging, circuit breaking.

**When the gateway becomes a problem:**

- **Bottleneck**: All traffic funnels through it. If the gateway is slow or down, everything is down. Must be horizontally scalable and stateless.
- **Business logic creep**: Starts with routing, then someone adds data transformation, then aggregation, then validation rules. Now the gateway is a monolith that every team depends on and nobody owns. Rule of thumb: the gateway should handle transport/security concerns, never domain logic.
- **Deployment coupling**: If all teams deploy through one gateway config, changes require coordination. Each update to routing rules becomes a bottleneck.

**When BFF (Backend for Frontend) is better:**

A BFF is a lightweight backend owned by a frontend team that aggregates and shapes data specifically for that client.

- **Different clients need very different data shapes**: Mobile needs minimal payload, web needs rich data, partner API needs a different structure entirely. A single gateway can't efficiently serve all three without becoming complex.
- **Frontend teams want to iterate independently**: With a BFF, the web team can change their API contract without coordinating with the mobile team.
- **Aggregation is client-specific**: If the web dashboard needs to combine data from 5 services into one view, that aggregation belongs in a BFF — not in a shared gateway.

**Gateway vs BFF — they often coexist:**

```
Browser → API Gateway → Web BFF → [Order Service, User Service, ...]
Mobile  → API Gateway → Mobile BFF → [Order Service, User Service, ...]
Partner → API Gateway → Partner API → [Order Service, ...]
```

The gateway handles auth and rate limiting. The BFF handles client-specific aggregation and shaping. Each layer does what it's good at.

</details>

<details>
<summary>14. Why does rate limiting at the API layer need different strategies for different dimensions — how do per-user, per-IP, and per-endpoint rate limits serve different purposes, what algorithms work for each (token bucket, sliding window), and where should rate limiting live (gateway vs application)?</summary>

Different dimensions protect against different threats. A single rate limit strategy can't cover all of them.

**Per-IP rate limiting:**

- **Purpose**: Prevents DDoS, brute force attacks, and abuse from unauthenticated clients
- **Key**: Client IP address (be careful with proxies — use `X-Forwarded-For` but validate it)
- **Typical limit**: High (e.g., 1000 req/min) — shared IPs behind corporate NATs shouldn't be punished
- **Best algorithm**: Token bucket — allows short bursts while enforcing average rate. A corporate office sharing one IP can burst, but sustained abuse is blocked.

**Per-user rate limiting:**

- **Purpose**: Fair usage enforcement — prevent one authenticated user from consuming disproportionate resources
- **Key**: User ID or API key from the authenticated request
- **Typical limit**: Varies by plan (free: 100 req/min, paid: 1000 req/min)
- **Best algorithm**: Sliding window — gives accurate per-minute/per-hour counts without the burst allowance. Users should get consistent, predictable limits.

**Per-endpoint rate limiting:**

- **Purpose**: Protect expensive operations — a search endpoint or report generation shouldn't be called as frequently as a health check
- **Key**: HTTP method + path pattern (e.g., `POST /orders`, `GET /search`)
- **Typical limit**: Low for write/expensive operations, high for reads
- **Best algorithm**: Either works. Token bucket if you want to allow burst writes; sliding window for strict enforcement.

**Token bucket vs sliding window:**

- **Token bucket**: Bucket holds N tokens, refills at a fixed rate. Each request consumes a token. Allows bursts up to bucket size, then enforces the refill rate. Good for traffic that's naturally bursty.
- **Sliding window**: Counts requests in a rolling time window. No burst allowance — 100 req/min means 100 req in any 60-second window. More predictable, simpler to reason about.

**Where should rate limiting live?**

- **Gateway layer**: Per-IP and global rate limits. This stops abusive traffic before it reaches your services. Cheaper to reject early.
- **Application layer**: Per-user and per-endpoint limits that require knowledge of the authenticated user or the business cost of an operation. The gateway doesn't know your pricing tiers.
- **Both is normal**: Gateway handles coarse IP-based limiting; application handles fine-grained user/endpoint limits.

**Response headers** (communicate limits to clients):

```
X-RateLimit-Limit: 100        # max requests allowed
X-RateLimit-Remaining: 42     # requests left in current window
X-RateLimit-Reset: 1710345600 # UTC epoch when window resets
Retry-After: 30               # seconds to wait (on 429 responses)
```

</details>

<details>
<summary>15. How do API authentication mechanisms differ at the design level — when should you use API keys vs Bearer tokens vs OAuth2, what are the conventions for the Authorization header, and how do you decide between 401 Unauthorized and 403 Forbidden for different failure scenarios?</summary>

**API keys:**

- A long random string (`sk_live_abc123...`) that identifies the calling application
- Sent as a header (`X-API-Key`) or query parameter
- **Use when**: Server-to-server integrations, third-party developer access, simple identity without user context
- **Limitations**: No user identity — the key represents an application, not a user. No expiration unless you build rotation. If leaked, it grants full access until revoked.

**Bearer tokens (typically JWTs):**

- Short-lived tokens carrying user identity and claims, sent in the `Authorization` header
- `Authorization: Bearer eyJhbGciOiJSUzI1NiJ9...`
- **Use when**: User-facing APIs where you need to know WHO is making the request and what permissions they have
- **Advantages**: Stateless validation (signature check, no DB lookup), carries user context (ID, roles, scopes), short-lived with refresh token rotation
- **The `Authorization` header convention**: `Authorization: <scheme> <credentials>` — `Bearer` is the scheme for OAuth2 tokens, `Basic` for base64-encoded username:password

**OAuth2:**

- An authorization framework — not a single mechanism but a set of flows (Authorization Code, Client Credentials, etc.)
- **Use when**: You need delegated authorization — "let this app access my data on the user's behalf" without sharing passwords
- **Client Credentials flow**: Machine-to-machine, no user involved. Client exchanges its ID + secret for a Bearer token.
- **Authorization Code flow**: User-facing apps. User authenticates with the identity provider, app receives an authorization code, exchanges it for tokens.
- OAuth2 produces Bearer tokens — so in practice, APIs validate Bearer tokens regardless of which OAuth2 flow generated them.

**401 vs 403 — the decision tree:**

| Scenario | Status | Why |
|----------|--------|-----|
| No `Authorization` header sent | 401 | No credentials provided — "authenticate yourself" |
| Token is expired or malformed | 401 | Credentials are invalid — "try again with valid credentials" |
| Valid token but user lacks permission for this resource | 403 | Authenticated but not authorized — "I know who you are, you can't do this" |
| Valid token but resource belongs to another tenant | 403 | Same — identity is established but access is denied |
| API key is invalid | 401 | No valid identity established |
| Valid API key but rate limit exceeded | 429 | Not an auth issue — use the correct status code |

**Key distinction**: 401 means "I don't know who you are" — the client should re-authenticate. 403 means "I know who you are, and the answer is no" — re-authenticating won't help. Include a `WWW-Authenticate` header with 401 responses to tell the client what authentication scheme to use.

</details>

<details>
<summary>16. Why does request validation need to happen at a specific point in the request lifecycle — where should schema validation (using tools like Zod or Joi) run relative to authentication and routing, what should validation error responses look like, and what are the risks of missing input sanitization?</summary>

**The correct order in the request lifecycle:**

```
Request → Rate Limiting → Authentication → Authorization → Routing → Validation → Handler
```

Validation runs **after authentication/authorization** but **before the business logic handler**. Why this order matters:

- **Auth before validation**: Don't waste CPU validating a request body from an unauthenticated user. Reject unauthorized requests early and cheaply.
- **Validation before handler**: The handler should trust its inputs. If validation passes, the handler can focus on business logic without defensive checks.
- **Routing before validation**: Each endpoint has its own schema. You need to know which endpoint was hit before you know which schema to apply.

**Schema validation with Zod:**

```typescript
import { z } from 'zod';

const CreateOrderSchema = z.object({
  customerId: z.string().uuid(),
  items: z.array(z.object({
    productId: z.string().uuid(),
    quantity: z.number().int().positive().max(1000),
  })).min(1).max(100),
  note: z.string().max(500).optional(),
});

// Validation middleware
function validate<T>(schema: z.ZodSchema<T>) {
  return (req: Request, res: Response, next: NextFunction) => {
    const result = schema.safeParse(req.body);
    if (!result.success) {
      return res.status(400).json({
        error: {
          code: 'VALIDATION_ERROR',
          message: 'Request body validation failed',
          details: result.error.issues.map(issue => ({
            field: issue.path.join('.'),
            message: issue.message,
            code: issue.code,
          })),
        },
      });
    }
    req.body = result.data; // replace with parsed & typed data
    next();
  };
}

app.post('/orders', authenticate, validate(CreateOrderSchema), createOrderHandler);
```

**Validation error response format** (as covered in question 7, keep it consistent):

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Request body validation failed",
    "details": [
      { "field": "items.0.quantity", "message": "Number must be greater than 0", "code": "too_small" },
      { "field": "customerId", "message": "Invalid uuid", "code": "invalid_string" }
    ]
  }
}
```

Map each validation issue to a specific field path so clients can display errors next to the right form field.

**Risks of missing input sanitization:**

- **SQL injection**: Unsanitized strings concatenated into queries. Use parameterized queries — schema validation alone doesn't prevent this.
- **NoSQL injection**: MongoDB query operators in user input (`{"$gt": ""}`) bypassing auth checks. Validate that fields are the expected type, not just present.
- **XSS via stored input**: If user input is stored and later rendered in HTML, malicious `<script>` tags execute in other users' browsers. Sanitize HTML on output or store sanitized versions.
- **Prototype pollution**: Malicious `__proto__` or `constructor` keys in JSON bodies can modify object prototypes. Zod strips unknown keys when configured with `.strict()` or `.strip()`.
- **Denial of service**: Extremely large strings, deeply nested objects, or massive arrays. Set explicit `.max()` limits on all unbounded fields.

</details>

## Practical — REST Patterns

<details>
<summary>17. Design a RESTful API for an order management system — show the resource hierarchy (orders, line items, payments), URI structure, HTTP methods for each operation, appropriate status codes for success and failure cases, and explain why you modeled sub-resources the way you did</summary>

**Resource hierarchy:**

```
/orders
/orders/{orderId}
/orders/{orderId}/line-items
/orders/{orderId}/line-items/{lineItemId}
/orders/{orderId}/payments
/orders/{orderId}/payments/{paymentId}
```

**API design:**

| Method | URI | Purpose | Success | Key Failures |
|--------|-----|---------|---------|-------------|
| `POST` | `/orders` | Create a new order | 201 + Location header | 400 (validation), 409 (duplicate idempotency key) |
| `GET` | `/orders` | List orders (paginated) | 200 | — |
| `GET` | `/orders/{orderId}` | Get order with embedded line items | 200 | 404 |
| `PATCH` | `/orders/{orderId}` | Update order (shipping address, notes) | 200 | 404, 409 (version conflict) |
| `DELETE` | `/orders/{orderId}` | Cancel order | 204 | 404, 409 (already shipped) |
| `POST` | `/orders/{orderId}/line-items` | Add line item | 201 | 404 (order), 400 (invalid product) |
| `PATCH` | `/orders/{orderId}/line-items/{lineItemId}` | Update quantity | 200 | 404 |
| `DELETE` | `/orders/{orderId}/line-items/{lineItemId}` | Remove line item | 204 | 404, 409 (order already processing) |
| `POST` | `/orders/{orderId}/payments` | Submit payment | 201 | 400, 402 (payment failed), 409 (already paid) |
| `GET` | `/orders/{orderId}/payments` | List payments for order | 200 | 404 (order) |

**Why sub-resources?**

Line items are modeled as sub-resources of orders because they have no meaning outside the context of an order — you never query "all line items across all orders" in client-facing flows. The lifecycle is tied to the parent: when an order is deleted, its line items go with it.

Payments are also sub-resources because each payment belongs to exactly one order. However, if you later need to query payments independently (e.g., an admin dashboard for payment reconciliation), you'd promote payments to a top-level resource: `GET /payments?orderId=123&status=failed`.

**Example: Create order request/response:**

```typescript
// POST /orders
// Idempotency-Key: 550e8400-e29b...
{
  "customerId": "cust_abc123",
  "shippingAddress": {
    "line1": "123 Main St",
    "city": "Berlin",
    "postalCode": "10115",
    "country": "DE"
  },
  "items": [
    { "productId": "prod_xyz", "quantity": 2 }
  ]
}

// Response: 201 Created
// Location: /orders/ord_789
{
  "id": "ord_789",
  "status": "pending",
  "customerId": "cust_abc123",
  "items": [
    {
      "id": "li_001",
      "productId": "prod_xyz",
      "quantity": 2,
      "unitPrice": { "amount": 2999, "currency": "EUR" }
    }
  ],
  "total": { "amount": 5998, "currency": "EUR" },
  "createdAt": "2026-03-13T10:00:00Z",
  "version": 1
}
```

**Design notes:**

- Monetary amounts are integers in the smallest currency unit (cents) to avoid floating-point issues
- `version` field enables optimistic concurrency — client sends `If-Match: 1` header on PATCH, server returns 409 if the version has changed
- Order state transitions (pending → paid → shipped → delivered) are enforced server-side — the API rejects invalid transitions with 409

</details>

<details>
<summary>18. Implement idempotency for a payment creation endpoint — show the idempotency key flow (header, storage, deduplication), how to handle concurrent requests with the same key, what storage backend you'd use, and what the client retry experience looks like</summary>

This builds on the idempotency concepts from question 3, applied to a concrete payment endpoint.

**Full implementation:**

```typescript
import { Redis } from 'ioredis';

const redis = new Redis();
const IDEMPOTENCY_TTL = 86400; // 24 hours
const LOCK_TTL = 30; // 30 seconds for in-flight processing

interface StoredResponse {
  statusCode: number;
  body: unknown;
}

// Idempotency middleware
async function idempotency(req: Request, res: Response, next: NextFunction) {
  const key = req.headers['idempotency-key'] as string;

  if (!key) {
    return res.status(400).json({
      error: { code: 'MISSING_IDEMPOTENCY_KEY', message: 'Idempotency-Key header is required for this endpoint' },
    });
  }

  const storageKey = `idempotency:${req.path}:${req.user.id}:${key}`;

  // 1. Check if we already have a stored response
  const cached = await redis.get(storageKey);
  if (cached) {
    const { statusCode, body }: StoredResponse = JSON.parse(cached);
    return res.status(statusCode).json(body);
  }

  // 2. Acquire lock to prevent concurrent duplicates
  const lockKey = `idempotency-lock:${storageKey}`;
  const acquired = await redis.set(lockKey, '1', 'NX', 'EX', LOCK_TTL);

  if (!acquired) {
    // Another request with the same key is currently being processed
    return res.status(409).json({
      error: { code: 'REQUEST_IN_PROGRESS', message: 'A request with this idempotency key is already being processed. Retry shortly.' },
    });
  }

  // 3. Intercept the response to store it
  const originalJson = res.json.bind(res);
  res.json = function (body: unknown) {
    // Store response for future duplicate requests
    const toStore: StoredResponse = { statusCode: res.statusCode, body };
    redis.set(storageKey, JSON.stringify(toStore), 'EX', IDEMPOTENCY_TTL);
    redis.del(lockKey);
    return originalJson(body);
  };

  next();
}

// Payment creation handler
app.post('/orders/:orderId/payments', authenticate, idempotency, async (req, res) => {
  const { orderId } = req.params;
  const { amount, currency, method } = req.body;

  const order = await orderService.findById(orderId);
  if (!order) return res.status(404).json({ error: { code: 'ORDER_NOT_FOUND' } });
  if (order.status !== 'pending') {
    return res.status(409).json({ error: { code: 'ORDER_NOT_PAYABLE', message: `Order status is ${order.status}` } });
  }

  const payment = await paymentService.create({ orderId, amount, currency, method });
  res.status(201).json(payment);
});
```

**Key design decisions:**

- **Scope the key to user + path**: `idempotency:${path}:${userId}:${key}` prevents one user from interfering with another's idempotency keys, and the same key can be reused across different endpoints.
- **Store the full response, not just "seen"**: On retry, the client gets back the exact same response — including the payment ID, status, and all fields. This is critical for payments where the client needs the payment reference.
- **Lock for concurrent requests**: The `NX` (set-if-not-exists) flag ensures only the first request processes. The second returns 409, telling the client to retry after a short delay.
- **TTL on everything**: Lock expires in 30s (handles crashes where the lock is never released). Stored response expires in 24h (no need to keep it forever).

**Why Redis?** Low-latency reads (sub-millisecond), atomic `SET NX` for locking, built-in TTL expiration. For payment-critical systems where you can't lose idempotency state on Redis restart, write the idempotency record to PostgreSQL in the same transaction as the payment creation — then Redis serves as a fast cache in front of it.

**Client retry experience:**

```typescript
async function createPaymentWithRetry(orderId: string, payment: PaymentInput) {
  const idempotencyKey = crypto.randomUUID(); // generate once, reuse on retries

  for (let attempt = 0; attempt < 3; attempt++) {
    try {
      const res = await fetch(`/orders/${orderId}/payments`, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'Idempotency-Key': idempotencyKey, // same key every retry
        },
        body: JSON.stringify(payment),
      });

      if (res.status === 409) {
        await sleep(1000 * (attempt + 1)); // backoff and retry
        continue;
      }
      return await res.json();
    } catch (networkError) {
      await sleep(1000 * (attempt + 1)); // network failure, retry is safe
    }
  }
  throw new Error('Payment creation failed after 3 attempts');
}
```

The client generates the idempotency key once and reuses it across all retries. Whether the first request succeeded (and the response was lost) or genuinely failed, the retry is safe — the server either returns the cached response or processes it for the first time.

</details>

<details>
<summary>19. Implement cursor-based pagination for a list endpoint that returns timestamped records — show the API contract (request params, response shape with next cursor), the database query approach, and explain why the cursor is opaque, what encoding to use, and how this handles records inserted between pages</summary>

This builds on the pagination concepts from question 5, with a full implementation.

**API contract:**

```
GET /orders?limit=20&cursor=eyJ0IjoiMjAyNi0wMy0xM1QxMDowMDowMFoiLCJpIjoib3JkXzc4OSJ9
```

**Response shape:**

```json
{
  "data": [
    { "id": "ord_788", "createdAt": "2026-03-13T09:59:00Z", "status": "shipped" },
    { "id": "ord_787", "createdAt": "2026-03-13T09:58:00Z", "status": "pending" }
  ],
  "pagination": {
    "nextCursor": "eyJ0IjoiMjAyNi0wMy0xM1QwOTo1ODowMFoiLCJpIjoib3JkXzc4NyJ9",
    "hasNextPage": true
  }
}
```

**Full implementation:**

```typescript
interface CursorPayload {
  t: string; // timestamp (createdAt)
  i: string; // id (tiebreaker)
}

function encodeCursor(createdAt: string, id: string): string {
  return Buffer.from(JSON.stringify({ t: createdAt, i: id })).toString('base64url');
}

function decodeCursor(cursor: string): CursorPayload {
  return JSON.parse(Buffer.from(cursor, 'base64url').toString());
}

app.get('/orders', authenticate, async (req, res) => {
  const limit = Math.min(Number(req.query.limit) || 20, 100); // cap at 100
  const cursor = req.query.cursor as string | undefined;

  let query = `
    SELECT id, status, customer_id, total_amount, created_at
    FROM orders
    WHERE customer_id = $1
  `;
  const params: unknown[] = [req.user.id];

  if (cursor) {
    const { t, i } = decodeCursor(cursor);
    // Composite keyset condition — handles duplicate timestamps
    query += ` AND (created_at, id) < ($${params.length + 1}, $${params.length + 2})`;
    params.push(t, i);
  }

  query += ` ORDER BY created_at DESC, id DESC LIMIT $${params.length + 1}`;
  params.push(limit + 1); // fetch one extra to determine hasNextPage

  const rows = await db.query(query, params);
  const hasNextPage = rows.length > limit;
  const data = rows.slice(0, limit);

  const lastItem = data[data.length - 1];
  const nextCursor = hasNextPage && lastItem
    ? encodeCursor(lastItem.created_at, lastItem.id)
    : null;

  res.json({
    data,
    pagination: { nextCursor, hasNextPage },
  });
});
```

**Why the cursor is opaque:**

The cursor is base64url-encoded so clients treat it as an opaque string. This gives you freedom to change the underlying pagination strategy (different sort columns, different encoding) without breaking clients. If you exposed raw values like `?after_timestamp=2026-03-13T09:58:00Z&after_id=ord_787`, clients would build logic around those parameters, and you'd be locked into that contract.

**Why base64url (not base64)?** URL-safe encoding — no `+`, `/`, or `=` characters that need percent-encoding in query strings.

**Why composite key `(created_at, id)`?** Timestamps alone aren't unique — two orders created in the same second would break pagination. The ID column as a tiebreaker guarantees uniqueness. The database needs a composite index:

```sql
CREATE INDEX idx_orders_pagination ON orders (customer_id, created_at DESC, id DESC);
```

**How this handles records inserted between pages:**

New records have a `created_at` newer than your cursor position. Since you're paginating backwards in time (`< cursor`), new inserts appear before your cursor — they don't affect the next page at all. You never see duplicates or gaps. This is the core advantage over offset pagination, where an insert at position 5 shifts everything and causes page 2 to repeat a record from page 1.

</details>

<details>
<summary>20. Design a consistent error response format and implement it across multiple endpoints — show the JSON structure (error code, message, details, correlation ID), how you map domain errors to HTTP status codes, and how correlation IDs flow from client through gateway to backend for debugging</summary>

This builds on the error design principles from question 7, with a concrete implementation.

**Error response structure:**

```json
{
  "error": {
    "code": "ORDER_NOT_CANCELLABLE",
    "message": "Cannot cancel an order that has already been shipped",
    "details": [
      { "field": "orderId", "issue": "Order ord_789 has status SHIPPED" }
    ],
    "correlationId": "req-a1b2c3d4-e5f6"
  }
}
```

**Domain error classes:**

```typescript
// Base error with code-to-status mapping
abstract class AppError extends Error {
  abstract readonly code: string;
  abstract readonly statusCode: number;
  readonly details?: Array<{ field: string; issue: string }>;

  constructor(message: string, details?: Array<{ field: string; issue: string }>) {
    super(message);
    this.details = details;
  }
}

class NotFoundError extends AppError {
  readonly code = 'NOT_FOUND';
  readonly statusCode = 404;
}

class ConflictError extends AppError {
  readonly code: string;
  readonly statusCode = 409;

  constructor(code: string, message: string) {
    super(message);
    this.code = code;
  }
}

class ValidationError extends AppError {
  readonly code = 'VALIDATION_ERROR';
  readonly statusCode = 400;
}

// Domain-specific errors
class OrderNotCancellable extends ConflictError {
  constructor(orderId: string, currentStatus: string) {
    super('ORDER_NOT_CANCELLABLE', `Cannot cancel order ${orderId} with status ${currentStatus}`);
  }
}
```

**Correlation ID flow — the full journey:**

```
Client → API Gateway → Backend Service → Database
  |          |              |
  |   Generate or forward   |
  |   X-Correlation-ID      |
  |          |              |
  |   Add to request headers |
  |          |              |
  |          |    Log with correlationId
  |          |              |
  |   Return in response    |
  ←──────────────────────────
```

**Middleware implementation:**

```typescript
import { randomUUID } from 'crypto';
import { AsyncLocalStorage } from 'async_hooks';

// Store correlation ID for the request lifecycle
const requestContext = new AsyncLocalStorage<{ correlationId: string }>();

// 1. Correlation ID middleware — runs first
function correlationId(req: Request, res: Response, next: NextFunction) {
  // Use client-provided ID or generate one
  const id = (req.headers['x-correlation-id'] as string) || `req-${randomUUID()}`;
  res.setHeader('X-Correlation-ID', id);

  // Make it available to all downstream code via AsyncLocalStorage
  requestContext.run({ correlationId: id }, () => next());
}

// 2. Error handler middleware — runs last (catches all thrown errors)
function errorHandler(err: Error, req: Request, res: Response, _next: NextFunction) {
  const ctx = requestContext.getStore();
  const correlationId = ctx?.correlationId || 'unknown';

  if (err instanceof AppError) {
    // Known domain error — map to structured response
    logger.warn({ correlationId, code: err.code, message: err.message });
    return res.status(err.statusCode).json({
      error: {
        code: err.code,
        message: err.message,
        details: err.details,
        correlationId,
      },
    });
  }

  // Unknown error — don't leak internals
  logger.error({ correlationId, err });
  res.status(500).json({
    error: {
      code: 'INTERNAL_ERROR',
      message: 'An unexpected error occurred',
      correlationId,
    },
  });
}

// 3. Usage in handlers — just throw domain errors
app.delete('/orders/:id', authenticate, async (req, res) => {
  const order = await orderService.findById(req.params.id);
  if (!order) throw new NotFoundError(`Order ${req.params.id} not found`);
  if (order.status === 'shipped') throw new OrderNotCancellable(order.id, order.status);

  await orderService.cancel(order.id);
  res.status(204).end();
});

// Register middleware
app.use(correlationId);
// ... routes ...
app.use(errorHandler);
```

**Key design decisions:**

- **AsyncLocalStorage** propagates the correlation ID to any code called during the request — including service layers, database queries, and logging — without passing it through every function signature.
- **Domain errors map to HTTP status codes** in the error class itself, keeping the mapping close to the error definition. The error handler doesn't need a giant switch statement.
- **Unknown errors return generic 500** — never expose stack traces, SQL errors, or internal details to clients. Log the full error server-side with the correlation ID so you can find it.
- **Gateway forwards the correlation ID** to downstream services via the `X-Correlation-ID` header. When a client reports an error, you search all service logs by that ID to trace the full request path.

</details>

## Practical — GraphQL, gRPC & Real-Time

<details>
<summary>21. Build a GraphQL API for a blog (posts, authors, comments) using a schema-first approach — show the schema definition, implement resolvers with DataLoader to prevent N+1 queries, and demonstrate how a single query that fetches posts with their authors and comments would execute without vs with DataLoader</summary>

This builds on the GraphQL concepts from questions 8-10, with a full working implementation.

**Schema definition:**

```graphql
type Query {
  posts(first: Int = 10, after: String): PostConnection!
}

type PostConnection {
  edges: [PostEdge!]!
  pageInfo: PageInfo!
}

type PostEdge {
  node: Post!
  cursor: String!
}

type PageInfo {
  hasNextPage: Boolean!
  endCursor: String
}

type Post {
  id: ID!
  title: String!
  body: String!
  author: Author!
  comments(first: Int = 20): [Comment!]!
  createdAt: String!
}

type Author {
  id: ID!
  name: String!
  email: String!
}

type Comment {
  id: ID!
  body: String!
  author: Author!
  createdAt: String!
}
```

**The query we want to optimize:**

```graphql
query {
  posts(first: 10) {
    edges {
      node {
        title
        author { name }
        comments(first: 5) {
          body
          author { name }
        }
      }
    }
  }
}
```

**Without DataLoader — the N+1 disaster:**

```typescript
// Naive resolvers
const resolvers = {
  Query: {
    posts: (_, { first, after }) => db.posts.findMany({ take: first }),
  },
  Post: {
    author: (post) => db.authors.findById(post.authorId),       // called 10x
    comments: (post) => db.comments.findByPostId(post.id),      // called 10x
  },
  Comment: {
    author: (comment) => db.authors.findById(comment.authorId), // called up to 50x
  },
};
// Total: 1 (posts) + 10 (post authors) + 10 (comments lists) + up to 50 (comment authors) = 71 queries
```

**With DataLoader — batched to ~4 queries:**

```typescript
import DataLoader from 'dataloader';

// Create loaders per request (critical — never share across requests)
function createLoaders() {
  return {
    authorLoader: new DataLoader<string, Author>(async (ids) => {
      const authors = await db.authors.findMany({ where: { id: { in: [...ids] } } });
      const map = new Map(authors.map(a => [a.id, a]));
      return ids.map(id => map.get(id) ?? new Error(`Author ${id} not found`));
    }),

    commentsByPostLoader: new DataLoader<string, Comment[]>(async (postIds) => {
      const comments = await db.comments.findMany({
        where: { postId: { in: [...postIds] } },
        orderBy: { createdAt: 'desc' },
      });
      // Group by postId, return in same order as input
      const grouped = new Map<string, Comment[]>();
      for (const c of comments) {
        const list = grouped.get(c.postId) ?? [];
        list.push(c);
        grouped.set(c.postId, list);
      }
      return postIds.map(id => grouped.get(id) ?? []);
    }),
  };
}

// Attach loaders to context (new instance per request)
const server = new ApolloServer({
  typeDefs,
  resolvers,
  context: () => ({ loaders: createLoaders() }),
});

// Optimized resolvers
const resolvers = {
  Query: {
    posts: async (_, { first, after }) => {
      const posts = await db.posts.findMany({ take: first + 1 }); // +1 for hasNextPage
      const hasNextPage = posts.length > first;
      const edges = posts.slice(0, first);
      return {
        edges: edges.map(p => ({ node: p, cursor: encodeCursor(p.id) })),
        pageInfo: { hasNextPage, endCursor: edges.length ? encodeCursor(edges.at(-1)!.id) : null },
      };
    },
  },
  Post: {
    author: (post, _, { loaders }) => loaders.authorLoader.load(post.authorId),
    comments: (post, { first = 20 }, { loaders }) =>
      loaders.commentsByPostLoader.load(post.id).then(c => c.slice(0, first)),
  },
  Comment: {
    author: (comment, _, { loaders }) => loaders.authorLoader.load(comment.authorId),
  },
};
```

**Execution with DataLoader:**

1. `posts` resolver runs — **1 query** (fetch 10 posts)
2. Engine resolves `author` and `comments` for all 10 posts. DataLoader collects all calls in the same tick:
   - `authorLoader.load()` called 10 times — batched into **1 query** (`SELECT * FROM authors WHERE id IN (...)`)
   - `commentsByPostLoader.load()` called 10 times — batched into **1 query** (`SELECT * FROM comments WHERE post_id IN (...)`)
3. Engine resolves `author` for all comments. `authorLoader.load()` called for each comment author — many are already cached from step 2 (post authors who also commented). Remaining unique IDs batched into **1 query**.

**Total: 4 queries instead of 71.** The `authorLoader` cache also deduplicates — if the same author wrote 3 comments, their data is fetched once.

</details>

## Practical — Production & Cross-Cutting

<details>
<summary>22. Implement rate limiting for a REST API with different limits per endpoint and per authenticated user — show the middleware code using a sliding window or token bucket approach with Redis as the backing store, the response headers (X-RateLimit-Limit, X-RateLimit-Remaining, Retry-After), and what the client experience looks like when they hit the limit</summary>

This builds on the rate limiting concepts from question 14, with a sliding window implementation.

**Sliding window counter using Redis:**

The sliding window approach uses Redis sorted sets. Each request is stored as a member with a timestamp score. To check the rate, count members within the current window.

```typescript
import { Redis } from 'ioredis';

const redis = new Redis();

interface RateLimitConfig {
  windowMs: number;   // window size in milliseconds
  maxRequests: number; // max requests per window
}

// Per-endpoint limits
const endpointLimits: Record<string, RateLimitConfig> = {
  'POST:/orders':          { windowMs: 60_000, maxRequests: 20 },
  'GET:/orders':           { windowMs: 60_000, maxRequests: 100 },
  'POST:/orders/*/payments': { windowMs: 60_000, maxRequests: 10 },
  'default':               { windowMs: 60_000, maxRequests: 60 },
};

// Per-user tier overrides
const tierMultipliers: Record<string, number> = {
  free: 1,
  pro: 5,
  enterprise: 20,
};

function getEndpointKey(method: string, path: string): string {
  // Match exact or wildcard patterns
  const exact = `${method}:${path}`;
  if (endpointLimits[exact]) return exact;

  // Try wildcard match (e.g., POST:/orders/*/payments)
  for (const pattern of Object.keys(endpointLimits)) {
    const regex = new RegExp('^' + pattern.replace(/\*/g, '[^/]+') + '$');
    if (regex.test(exact)) return pattern;
  }
  return 'default';
}

async function slidingWindowCheck(
  key: string,
  config: RateLimitConfig,
): Promise<{ allowed: boolean; remaining: number; resetAt: number }> {
  const now = Date.now();
  const windowStart = now - config.windowMs;

  // Atomic pipeline: remove expired entries, count current, add new entry
  const pipeline = redis.pipeline();
  pipeline.zremrangebyscore(key, 0, windowStart);       // remove entries outside window
  pipeline.zcard(key);                                   // count entries in window
  pipeline.zadd(key, now.toString(), `${now}:${Math.random()}`); // add current request
  pipeline.pexpire(key, config.windowMs);                // set TTL on the key

  const results = await pipeline.exec();
  const currentCount = results![1][1] as number; // count BEFORE adding current request

  const allowed = currentCount < config.maxRequests;
  const remaining = Math.max(0, config.maxRequests - currentCount - 1);
  const resetAt = Math.ceil((now + config.windowMs) / 1000);

  if (!allowed) {
    // Remove the entry we just added since the request is rejected
    await redis.zremrangebyscore(key, now, now);
  }

  return { allowed, remaining, resetAt };
}

// Rate limiting middleware
function rateLimit() {
  return async (req: Request, res: Response, next: NextFunction) => {
    const endpointKey = getEndpointKey(req.method, req.path);
    const config = endpointLimits[endpointKey] || endpointLimits['default'];

    // Apply user tier multiplier
    const tier = req.user?.tier || 'free';
    const multiplier = tierMultipliers[tier] || 1;
    const effectiveLimit = config.maxRequests * multiplier;
    const effectiveConfig = { ...config, maxRequests: effectiveLimit };

    // Key scoped to user + endpoint pattern
    const userId = req.user?.id || req.ip;
    const redisKey = `ratelimit:${userId}:${endpointKey}`;

    const { allowed, remaining, resetAt } = await slidingWindowCheck(redisKey, effectiveConfig);

    // Always set rate limit headers — even on successful requests
    res.setHeader('X-RateLimit-Limit', effectiveLimit);
    res.setHeader('X-RateLimit-Remaining', remaining);
    res.setHeader('X-RateLimit-Reset', resetAt);

    if (!allowed) {
      const retryAfter = Math.ceil(config.windowMs / 1000);
      res.setHeader('Retry-After', retryAfter);
      return res.status(429).json({
        error: {
          code: 'RATE_LIMITED',
          message: `Rate limit exceeded. Maximum ${effectiveLimit} requests per ${config.windowMs / 1000}s for this endpoint.`,
          retryAfter,
        },
      });
    }

    next();
  };
}

// Register
app.use(rateLimit());
```

**Client experience when hitting the limit:**

```
HTTP/1.1 429 Too Many Requests
X-RateLimit-Limit: 20
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1710345660
Retry-After: 60
Content-Type: application/json

{
  "error": {
    "code": "RATE_LIMITED",
    "message": "Rate limit exceeded. Maximum 20 requests per 60s for this endpoint.",
    "retryAfter": 60
  }
}
```

A well-behaved client reads `Retry-After` and backs off:

```typescript
async function fetchWithRateLimit(url: string, options: RequestInit) {
  const res = await fetch(url, options);

  if (res.status === 429) {
    const retryAfter = Number(res.headers.get('Retry-After') || 60);
    await sleep(retryAfter * 1000);
    return fetch(url, options); // single retry
  }

  // Log remaining quota for observability
  const remaining = res.headers.get('X-RateLimit-Remaining');
  if (remaining && Number(remaining) < 5) {
    console.warn(`Rate limit nearly exhausted: ${remaining} remaining`);
  }

  return res;
}
```

**Why sliding window over token bucket here:** Sliding window gives predictable, exact counts per time window — there's no burst allowance. For a payment endpoint limited to 10/minute, you want strict enforcement, not "10 per minute on average with occasional bursts of 20." Token bucket would be better for endpoints where natural traffic is bursty (e.g., batch imports).

</details>

---

## Experience-Based Questions

These questions test real-world experience. Prepare by mapping them to your own projects and situations.

<details>
<summary>23. Tell me about a time you designed an API from scratch for a new product or feature — what design decisions did you make, what tradeoffs did you face, and what would you change in hindsight?</summary>

**What the interviewer is looking for:**

- Ability to think about API consumers, not just implementation
- Awareness of design tradeoffs (consistency vs convenience, flexibility vs simplicity)
- Willingness to reflect on past decisions honestly
- Understanding of how API decisions compound over time

**Suggested structure (STAR-like):**

1. **Context**: What was the feature/product? Who were the API consumers (frontend team, mobile, third party, internal services)?
2. **Key decisions**: Resource modeling, REST vs GraphQL, authentication model, versioning strategy, error format. Pick 2-3 decisions that had the most impact.
3. **Tradeoffs you navigated**: What did you optimize for and what did you sacrifice? (Speed to ship vs extensibility? Simplicity vs flexibility for multiple clients?)
4. **What worked**: Which decisions aged well? What did consumers appreciate?
5. **What you'd change**: Be specific and honest. Not "I'd use a different framework" but "I'd model X as a sub-resource instead of a top-level resource because Y."

**Example outline to personalize:**

- "I designed the API for [feature] that served [N clients/consumers]."
- "The biggest design decision was [resource modeling / choosing REST over GraphQL / pagination approach]. I chose X because [consumer needs, team constraints, timeline]."
- "The tradeoff was [flexibility vs shipping speed]. We sacrificed [extensibility for nested resources] to ship in [timeframe]."
- "In hindsight, I'd change [specific modeling decision] because we later needed [capability] and the original design made it harder — we ended up adding [workaround]."
- "The lesson I took away was [concrete principle about API design, like 'design for the next two use cases, not the next ten']."

**Key points to hit:**

- Show you thought about the consumer experience, not just the backend implementation
- Demonstrate you understand that API decisions are hard to reverse — they become contracts
- Be honest about mistakes — interviewers value self-awareness over a perfect story

</details>

<details>
<summary>24. Describe a time you dealt with API backwards compatibility issues — what was breaking, how did you manage the migration, and what did you learn about API evolution?</summary>

**What the interviewer is looking for:**

- Understanding that APIs are contracts with real downstream impact
- Experience managing a migration without breaking consumers
- Awareness of deprecation strategies and communication practices
- Lessons learned about preventing compatibility issues in the first place

**Suggested structure:**

1. **What broke (or almost broke)**: A field rename, type change, removed endpoint, changed response shape, or shifted semantics. Be specific about the breaking change.
2. **Impact assessment**: How many consumers were affected? How did you discover the issue (monitoring, client reports, proactive review)?
3. **Migration strategy**: Did you version the API? Run old and new in parallel? Use feature flags or content negotiation? Add a deprecation period?
4. **Communication**: How did you notify consumers? Sunset headers, changelog, direct outreach?
5. **Resolution and timeline**: How long did the migration take? What tools helped (API diffing, contract tests)?
6. **Lesson learned**: What process or design change did you adopt to prevent this next time?

**Example outline to personalize:**

- "We needed to [change field type / restructure response / rename endpoint] because [business reason]."
- "The challenge was that [N consumers / external partners / mobile apps with slow update cycles] depended on the old format."
- "We handled it by [running both versions in parallel / adding a new field alongside the old one / introducing a v2 endpoint with a deprecation timeline on v1]."
- "We communicated through [Sunset headers / API changelog / direct Slack/email to consumer teams] and monitored v1 usage to know when it was safe to remove."
- "The key lesson was [additive changes are almost always safe / never rename a field in place — add new, deprecate old / contract tests between producer and consumer catch this early]."

**Key points to hit:**

- Show empathy for API consumers — they can't always update immediately
- Demonstrate a systematic migration approach, not a "deploy and hope" strategy
- Reference concrete practices: Sunset headers (as covered in question 4), contract tests, monitoring usage of deprecated endpoints

</details>

<details>
<summary>25. Tell me about a time you had to choose between REST and GraphQL (or another paradigm) for a project — what factors drove the decision, and how did it work out?</summary>

**What the interviewer is looking for:**

- Ability to evaluate technology choices based on context, not hype
- Understanding of the real tradeoffs between paradigms (not just textbook comparisons)
- Evidence that you considered team capabilities, client needs, and operational complexity
- Honest reflection on whether the choice was right

**Suggested structure:**

1. **Context**: What was the project? What were the client types (browser, mobile, internal service)?
2. **Options considered**: Which paradigms did you evaluate and why?
3. **Decision factors**: What drove the final choice? (Team familiarity, client diversity, caching needs, latency requirements, time to ship)
4. **What you chose and why**: Connect the decision directly to the factors
5. **How it worked out**: Did the choice age well? Were there unexpected benefits or costs?
6. **What you'd tell someone facing the same decision**: Distill the lesson

**Example outline to personalize:**

- "For [project], we needed an API that served [web + mobile / multiple internal services / external developers]."
- "We considered [REST and GraphQL / REST and gRPC]. The key factors were [client diversity — mobile wanted minimal payloads, web wanted rich data / latency between internal services / team had no GraphQL experience]."
- "We chose [paradigm] because [specific factor was the tiebreaker]. For example, [GraphQL let frontend teams iterate on data needs without backend changes / REST's caching gave us CDN benefits we needed / gRPC's code generation eliminated contract drift between services]."
- "It worked out [well / with some pain]. The benefit we didn't expect was [specific]. The cost we underestimated was [operational complexity / learning curve / tooling gaps]."
- "The lesson: [choose based on who your consumers are and what your team can operate, not on which technology is more modern]."

**Key points to hit:**

- Ground the decision in concrete project constraints, not abstract preference
- Reference the decision framework from question 2 — show you think systematically about paradigm selection
- Acknowledge that combinations are normal (REST for public, gRPC for internal, etc.)

</details>

<details>
<summary>26. Describe a production incident caused by an API design issue (missing idempotency, poor error handling, pagination problems, or rate limiting gaps) — what went wrong, how did you fix it, and what design changes did you make to prevent it from recurring?</summary>

**What the interviewer is looking for:**

- Real experience with production consequences of API design decisions
- Structured incident response thinking (detect, mitigate, fix, prevent)
- Ability to trace a production issue back to a design root cause
- Evidence that you improved the system, not just patched the symptom

**Suggested structure:**

1. **The incident**: What happened from the user/client perspective? (Duplicate charges, missing data, service degradation, client errors)
2. **Root cause**: Which API design gap caused it? Be specific — missing idempotency key on a payment endpoint, no rate limiting on an expensive search, offset pagination causing duplicate records, error responses that gave clients no useful information for retry logic.
3. **Detection and response**: How was it detected (monitoring, client report, alert)? What was the immediate mitigation?
4. **The fix**: What code/design change resolved the immediate issue?
5. **Prevention**: What systematic design change did you make to prevent the class of problem? (Added idempotency middleware to all write endpoints, implemented rate limiting at the gateway, switched from offset to cursor pagination, standardized error response format)
6. **Lesson**: What principle did this reinforce?

**Example outline to personalize:**

- "We had a production incident where [specific symptom — duplicate orders, service overloaded by one client, users seeing repeated items in a feed]."
- "The root cause was [missing idempotency on POST /orders — client retries on timeout created duplicate orders / no per-endpoint rate limiting — one client's batch script hit our search endpoint 10k times per minute / offset pagination on a frequently-updated feed — inserts between page fetches caused duplicates]."
- "We detected it through [monitoring spike / client support ticket / anomaly in order counts]."
- "Immediate mitigation was [manual dedup / blocking the client / adding a temporary cache]."
- "The permanent fix was [implementing idempotency keys (as in question 3/18) / adding sliding window rate limiting per endpoint (as in question 14/22) / migrating to cursor-based pagination (as in question 5/19)]."
- "To prevent recurrence, we [added idempotency as a required middleware for all POST endpoints / made rate limiting part of our API design checklist / added integration tests that paginate through mutating data]."
- "The lesson: [API design gaps don't surface in testing — they surface under production load with real client behavior. Design defensively from the start.]"

**Key points to hit:**

- Connect the incident to a specific API design concept covered in this file — shows depth of understanding
- Show the progression from symptom to root cause to systemic fix
- Emphasize prevention over just fixing the immediate issue — this signals senior thinking

</details>
