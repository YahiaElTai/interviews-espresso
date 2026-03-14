# Web Servers & Reverse Proxies

> **20 questions** — 9 theory, 9 practical, 2 experience

- Forward proxy vs reverse proxy: purpose, architecture, and use cases
- Request lifecycle through Nginx to a Node.js backend (TLS, headers, buffering, upstream selection)
- Nginx architecture: worker processes, event-driven model (epoll), why it handles high concurrency efficiently
- Nginx config structure and operations: http/server/location context hierarchy, nginx -t validation, graceful reload vs restart
- Load balancing algorithms: round-robin, least connections, IP hash, generic hash — tradeoffs and failure modes
- TLS termination at the proxy: certificate management, TLS passthrough, re-encryption, compliance
- Rate limiting: leaky bucket, token bucket, sliding window — proxy-layer vs application-layer
- WebSocket and SSE proxying: connection upgrade handling, proxy timeouts for long-lived connections, buffering behavior
- Proxy headers: X-Forwarded-For, X-Real-IP, Host — and trust-proxy security risks
- Debugging: 502 Bad Gateway, 504 Gateway Timeout, DNS caching, stale upstream IPs
- Path-based routing: serving multiple services from one domain, location block priority and matching rules

---

## Foundational

<details>
<summary>1. What is the difference between a forward proxy and a reverse proxy — why do they exist as separate concepts, what architectural role does each play, and what are the real-world use cases where you'd deploy one vs the other?</summary>

The distinction comes down to **which side of the connection the proxy represents**.

**Forward proxy** sits in front of **clients** and makes requests on their behalf to the internet. The backend servers don't know the real client — they see the proxy's IP. The client explicitly knows it's using a proxy (configured in browser/OS settings or via environment variables).

**Reverse proxy** sits in front of **servers** and receives requests on their behalf from clients. The client doesn't know (or care) that a proxy exists — it looks like it's talking directly to the server.

**Forward proxy use cases:**
- Corporate networks controlling/monitoring outbound traffic
- Bypassing geo-restrictions or content filters
- Caching frequently accessed external resources to reduce bandwidth
- Anonymizing client identity from destination servers

**Reverse proxy use cases:**
- Load balancing across multiple backend instances
- TLS termination — handle encryption in one place instead of every backend
- Serving static files efficiently without hitting the application server
- Rate limiting, request filtering, and DDoS mitigation at the edge
- Path-based routing — single domain, multiple backend services

**Why they're separate concepts:** A forward proxy is a client-side concern (the client opts in). A reverse proxy is an infrastructure concern (the backend operator deploys it). They solve fundamentally different problems — client anonymity/control vs server scalability/security — even though the underlying mechanism (intercepting and forwarding HTTP requests) is similar.

</details>

<details>
<summary>2. Walk through the full lifecycle of an HTTPS request from a client browser through Nginx to a Node.js backend and back — what happens at each stage (DNS resolution, TLS handshake, header manipulation, buffering, upstream selection, response), and why does putting Nginx in front of Node.js improve the overall architecture?</summary>

**Step-by-step lifecycle:**

1. **DNS resolution** — Browser resolves `api.example.com` to the IP of the Nginx server (or a cloud load balancer in front of it). The browser may cache this result per the DNS TTL.

2. **TCP connection** — Browser opens a TCP connection to Nginx on port 443 (3-way handshake: SYN, SYN-ACK, ACK).

3. **TLS handshake** — Nginx and the browser negotiate TLS. Nginx presents its certificate, they agree on a cipher suite, and establish a symmetric session key. This is where TLS termination happens — the encrypted tunnel ends at Nginx.

4. **HTTP request parsing** — Nginx decrypts the request and parses the HTTP method, URI, headers, and body. It matches the request against `server` blocks (by `server_name` and port) and then against `location` blocks within the matched server.

5. **Header manipulation** — Nginx adds/modifies headers before forwarding: sets `X-Forwarded-For` to the client IP, `X-Real-IP`, `X-Forwarded-Proto: https`, and passes through or overrides the `Host` header.

6. **Upstream selection** — Based on the matched `location` block's `proxy_pass`, Nginx selects a backend from the `upstream` pool using the configured algorithm (round-robin by default). It skips servers marked as failed.

7. **Request buffering** — By default, Nginx buffers the entire client request body before forwarding to the upstream. This protects slow backends from holding connections open while slow clients trickle in data.

8. **Proxy to backend** — Nginx opens a connection to the selected Node.js instance (usually plain HTTP on a private network) and forwards the request. This connection may be reused via keepalive.

9. **Backend processing** — Node.js handles the request and sends the response back to Nginx.

10. **Response buffering** — Nginx buffers the upstream response (controlled by `proxy_buffering`). This frees the backend connection quickly — the backend doesn't wait for the slow client to download.

11. **Response to client** — Nginx sends the buffered response back over the existing TLS connection to the browser. Nginx may add response headers (`add_header` directives like HSTS, CORS, cache-control).

**Why Nginx in front of Node.js improves the architecture:**

- **TLS offloading** — Node.js doesn't manage certificates. One place to rotate certs, configure protocols.
- **Static file serving** — Nginx serves static assets from disk without hitting Node.js at all. Far more efficient.
- **Request buffering** — Protects Node.js from slow clients tying up the event loop's connections.
- **Load balancing** — Distribute traffic across multiple Node.js processes/containers.
- **Graceful deployments** — Drain connections from old backends while routing new traffic to new ones.
- **Security layer** — Rate limiting, IP blocking, request size limits, and header sanitization happen before traffic reaches application code.

</details>

<details>
<summary>3. How does Nginx's architecture — master process, worker processes, and event-driven I/O with epoll — allow it to handle tens of thousands of concurrent connections on a single machine, and why is this fundamentally different from a thread-per-connection model like traditional Apache?</summary>

**Nginx architecture:**

- **Master process** — Runs as root. Reads config, binds to ports (80/443), spawns worker processes, and handles signals (reload, stop). Does not serve requests.
- **Worker processes** — Typically one per CPU core. Each worker is a single-threaded process that handles thousands of connections concurrently using non-blocking I/O.
- **Event loop with epoll** — Each worker uses the OS kernel's `epoll` (Linux) or `kqueue` (macOS/BSD) to monitor thousands of file descriptors (sockets) simultaneously. When data is ready on any socket, the kernel notifies the worker, which processes it and moves on. No blocking, no waiting.

**How this enables high concurrency:**

Each connection is just a file descriptor and a small state struct in memory — a few KB. A single worker can hold 10,000+ connections because it never blocks waiting for any one of them. When a client is slow to send data, the worker just processes other ready connections and comes back when epoll says data arrived.

**Why thread-per-connection (Apache prefork/worker) is fundamentally different:**

In the traditional Apache model, each connection gets its own thread (or process). This means:
- **Memory overhead** — Each thread consumes ~1-8 MB of stack space. 10,000 connections = 10-80 GB of RAM just for stacks.
- **Context switching** — The OS must constantly switch between thousands of threads, burning CPU cycles on scheduling rather than actual work.
- **Blocking I/O** — Each thread blocks on `read()`/`write()` calls, sitting idle while waiting for slow clients.

**Nginx's model** avoids all three: minimal per-connection memory, no context switching (one thread per core), and never blocks on I/O. The tradeoff is that worker code must be non-blocking — you can't do CPU-heavy computation inside a worker without stalling all connections on that worker. This is fine for a reverse proxy (the work is just shuffling bytes) but is why you wouldn't run application logic inside Nginx.

**Practical configuration:**

```nginx
worker_processes auto;          # one per CPU core
events {
    worker_connections 10240;   # max connections per worker
    use epoll;                  # explicit on Linux (auto-detected)
    multi_accept on;            # accept all pending connections at once
}
```

With 4 cores and 10,240 connections per worker, this handles ~40,000 concurrent connections.

</details>

## Conceptual Depth

<details>
<summary>4. What are the main load balancing algorithms Nginx supports (round-robin, least connections, IP hash, generic hash), what tradeoffs does each make between simplicity, fairness, and session affinity — and what failure modes can each algorithm introduce when upstream servers go down or respond slowly?</summary>

**Round-robin (default)**

Distributes requests sequentially across upstreams, respecting optional `weight` values.

- **Tradeoff:** Simplest, no state to maintain. Fair distribution assuming requests have similar cost. No session affinity.
- **Failure mode:** If one backend is slow (not down, just slow), round-robin keeps sending it requests at the same rate, creating a growing queue on that server while others sit idle. This is the "slow turtle" problem.

**Least connections** (`least_conn`)

Sends each new request to the upstream with the fewest active connections.

- **Tradeoff:** Adapts to backends with different processing speeds — slow backends naturally accumulate connections, so they get fewer new ones. Better fairness for heterogeneous workloads.
- **Failure mode:** A backend that accepts connections but processes them extremely slowly will still accumulate connections. Also, if a backend is hanging (connection established but no response), it keeps its connection count high — which is actually self-correcting. The real risk is a backend that responds quickly but with errors — it'll have low connection count and keep getting traffic.

**IP hash** (`ip_hash`)

Hashes the client IP to deterministically route to the same upstream. Provides session affinity without cookies.

- **Tradeoff:** Guarantees the same client always hits the same backend. Useful for in-memory sessions or local caches. But distribution is only as fair as the IP distribution — a corporate NAT sending thousands of users from one IP overloads one backend.
- **Failure mode:** When a backend goes down, all clients hashed to it get redistributed — AND the hash table is recalculated, potentially reshuffling many other clients too. If you're relying on affinity for session state, those sessions break.

**Generic hash** (`hash $request_uri consistent`)

Hashes an arbitrary key (URI, cookie, header). The `consistent` flag enables consistent hashing (ketama), which minimizes reshuffling when upstreams change.

- **Tradeoff:** Very flexible — hash by anything. With `consistent`, adding/removing a server only remaps ~1/N of keys. Great for cache-friendly routing where you want the same URL to always hit the same backend (maximizing local cache hit rates).
- **Failure mode:** Without `consistent`, any upstream change causes complete redistribution. With it, the failure is more subtle: if your hash key has low cardinality (e.g., only 5 unique values), traffic concentrates on a few backends regardless of the algorithm.

```nginx
upstream api {
    least_conn;
    server 10.0.0.1:3000 weight=3 max_fails=3 fail_timeout=30s;
    server 10.0.0.2:3000 weight=2;
    server 10.0.0.3:3000 backup;  # only used when others are down
}
```

**When all upstreams fail:** Nginx returns 502 Bad Gateway. If you have `backup` servers defined, those are tried first. The `fail_timeout` window eventually expires and Nginx retries the failed servers.

</details>

<details>
<summary>5. Why is TLS termination typically done at the reverse proxy rather than at each application server — what are the operational and performance benefits, when would you use TLS passthrough instead, when would you re-encrypt traffic between the proxy and backend, and how do compliance requirements (PCI-DSS, HIPAA) influence this decision?</summary>

**Why terminate TLS at the proxy:**

- **Centralized certificate management** — One place to install, renew, and rotate certificates instead of configuring every application instance. Critical at scale with dozens of services.
- **Performance** — TLS handshakes are CPU-intensive (asymmetric crypto). The proxy can use hardware acceleration (AES-NI) and maintain TLS session caches. Backends speak plain HTTP, which is cheaper.
- **Simplified backends** — Application code doesn't need TLS configuration, certificate paths, or protocol version management. Fewer moving parts = fewer misconfigurations.
- **Connection reuse** — The proxy terminates thousands of client TLS connections but uses a small pool of persistent HTTP connections to backends, reducing backend connection overhead dramatically.
- **Visibility** — The proxy can inspect request content for rate limiting, routing, header manipulation, WAF rules, and logging. With TLS passthrough, it's an opaque byte stream.

**When to use TLS passthrough:**

The proxy forwards the raw TLS stream to the backend without decrypting. Use when:
- The backend must terminate TLS itself (mutual TLS / client certificate authentication where the backend needs the client cert).
- You legally cannot decrypt traffic at the proxy (certain compliance scenarios).
- You're proxying non-HTTP protocols over TLS (e.g., database connections).

The downside: the proxy can only route based on SNI (Server Name Indication) — it can't see the URL path, headers, or body.

**When to re-encrypt (TLS termination + backend TLS):**

The proxy terminates the client TLS connection, inspects/manipulates the request, then creates a new TLS connection to the backend. Use when:
- Internal network isn't trusted (multi-tenant cloud, traffic crosses network boundaries).
- Compliance requires encryption in transit end-to-end, but you still need proxy-layer routing and inspection.

The cost is double TLS overhead and certificate management on both sides.

**Compliance influence:**

- **PCI-DSS** — Requires encryption of cardholder data in transit. If the proxy-to-backend path crosses untrusted networks, you need re-encryption. On a private VLAN within the same data center, TLS termination at the proxy is generally acceptable.
- **HIPAA** — Requires encryption of PHI in transit. Similar to PCI — the question is whether the internal network segment is considered "in transit." Most auditors accept termination at the proxy if the backend network is isolated and access-controlled.
- **Both** — Require strong protocols (TLS 1.2+) and cipher suites. Easier to enforce in one place (the proxy) than across every application.

</details>

<details>
<summary>6. What are the main rate limiting algorithms (leaky bucket, token bucket, sliding window), how do they differ in behavior and fairness — and when should rate limiting be done at the proxy layer vs the application layer, considering that both approaches have different strengths and failure modes?</summary>

**Leaky bucket**

Requests enter a queue (bucket) and are processed at a fixed rate. If the bucket is full, new requests are rejected.

- **Behavior:** Smooths traffic to a constant output rate. Bursts fill the queue and are processed gradually.
- **Fairness:** Very fair — everyone gets the same steady rate. But no ability to handle legitimate bursts.
- **This is what Nginx's `limit_req` implements** — the `rate` parameter is the drain rate, and `burst` is the bucket size.

**Token bucket**

Tokens are added to a bucket at a fixed rate (up to a max capacity). Each request consumes a token. No tokens = rejected.

- **Behavior:** Allows bursts up to the bucket capacity, then settles to the token refill rate. More flexible than leaky bucket.
- **Fairness:** A client that was idle accumulates tokens and can burst, which may be desirable (letting a returning user load a page quickly) or a problem (a bot saving up tokens).
- **Common in:** API gateways, AWS API Gateway, many application-level rate limiters.

**Sliding window**

Counts requests in a rolling time window. Can be exact (sliding window log — store every timestamp) or approximate (sliding window counter — interpolate between current and previous fixed windows).

- **Behavior:** Limits total requests in any given window period. No smoothing — a client can use all their quota in a burst at the start of the window.
- **Fairness:** Most intuitive for API rate limits ("100 requests per minute"). The counter variant is memory-efficient and widely used.
- **Common in:** Application-level rate limiters, Redis-based implementations.

**Proxy layer vs application layer:**

| Aspect | Proxy layer (Nginx) | Application layer |
|---|---|---|
| **Granularity** | Per-IP, per-URI pattern | Per-user, per-API-key, per-tenant, per-endpoint |
| **Speed** | Extremely fast — rejects before touching backend | Adds latency; request must reach the app |
| **State** | Shared memory on one machine; no cluster awareness without sync | Can use centralized store (Redis) shared across instances |
| **Use case** | DDoS mitigation, brute force prevention, coarse protection | Business-level quotas (free tier: 1000 req/day), per-tenant limits |
| **Failure mode** | If Nginx restarts, counters reset. Multiple Nginx instances = each has its own counters | Redis failure = rate limits stop working (decide: fail open or closed) |

**Practical recommendation:** Use both. Proxy-layer rate limiting as a first line of defense (cheap, fast, protects the backend from abuse). Application-layer rate limiting for business logic (per-user quotas, tiered pricing, tenant-level fairness). The proxy catches the firehose; the application enforces the rules.

</details>

<details>
<summary>7. Why is proxying WebSocket and SSE connections different from regular HTTP — how does the connection upgrade mechanism work through a reverse proxy, what proxy timeout and buffering settings break long-lived connections if misconfigured, and how does Nginx's buffering behavior differ between SSE streams and WebSocket frames?</summary>

**Why they're different from regular HTTP:**

Standard HTTP is request-response: client sends request, server sends response, done. WebSocket and SSE are **long-lived connections** that stay open for minutes, hours, or indefinitely. A reverse proxy configured for normal HTTP will aggressively time out, buffer, or close these connections.

**WebSocket upgrade mechanism through a proxy:**

1. Client sends an HTTP/1.1 request with `Upgrade: websocket` and `Connection: Upgrade` headers.
2. Nginx must **forward these hop-by-hop headers** to the backend (normally, `Connection` and `Upgrade` are stripped by proxies).
3. Backend responds with `101 Switching Protocols`.
4. Nginx switches the connection to a bidirectional TCP tunnel — no more HTTP parsing, just raw frame forwarding in both directions.

The critical Nginx config:

```nginx
location /ws/ {
    proxy_pass http://backend;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_read_timeout 3600s;   # keep connection alive for 1 hour
    proxy_send_timeout 3600s;
}
```

`proxy_http_version 1.1` is required because HTTP/1.0 doesn't support the `Upgrade` mechanism. Without it, the WebSocket handshake fails silently.

**SSE (Server-Sent Events):**

SSE is a standard HTTP response with `Content-Type: text/event-stream` that the server keeps open, sending data chunks over time. No protocol upgrade — it's just a long-lived HTTP response.

The problem: **Nginx buffers responses by default.** It collects the upstream response into memory/disk before sending to the client. For SSE, this means events are held in the buffer and delivered in batches instead of real-time.

```nginx
location /events/ {
    proxy_pass http://backend;
    proxy_buffering off;          # send events immediately as they arrive
    proxy_cache off;              # don't cache the stream
    proxy_read_timeout 3600s;    # don't kill the stream after 60s (default)
    proxy_set_header Connection '';  # prevent keepalive issues
}
```

**Key differences in buffering behavior:**

- **WebSocket:** After the 101 upgrade, Nginx treats it as a raw TCP tunnel. Buffering doesn't apply — frames pass through immediately in both directions.
- **SSE:** Remains an HTTP response, so Nginx's response buffering IS active by default. You must explicitly set `proxy_buffering off` or the client sees events delayed in chunks.

**What breaks when misconfigured:**

- `proxy_read_timeout` left at default 60s: WebSocket/SSE connections close after 60 seconds of no data. The fix is either increasing the timeout or implementing application-level pings.
- Missing `proxy_http_version 1.1`: WebSocket upgrade fails — client gets a 400 or the connection drops.
- Missing `Upgrade`/`Connection` header forwarding: Backend never sees the upgrade request — responds with normal HTTP.
- `proxy_buffering on` for SSE: Events arrive to the client in delayed bursts instead of real-time.

</details>

<details>
<summary>8. What do the proxy headers X-Forwarded-For, X-Real-IP, and Host convey, why are they necessary when a reverse proxy sits between clients and backends — and what security vulnerabilities arise from trusting these headers without proper configuration (trust-proxy chains, IP spoofing)?</summary>

**What each header conveys:**

- **`X-Forwarded-For` (XFF):** A comma-separated list of IPs the request has passed through. Each proxy appends the IP it received the request from. Example: `X-Forwarded-For: 203.0.113.50, 10.0.0.1` means the client was 203.0.113.50, and it passed through a proxy at 10.0.0.1.

- **`X-Real-IP`:** The single, original client IP address. Typically set by the first trusted proxy (Nginx) and not appended to — just overwritten. Simpler than XFF when you only need the client IP.

- **`Host`:** The original hostname the client requested (e.g., `api.example.com`). Without forwarding this, the backend sees the proxy's internal hostname/IP, which breaks virtual hosting, cookie domains, and URL generation.

**Why they're necessary:**

Without a reverse proxy, the backend reads the client IP from the TCP socket. With a proxy in between, the TCP source IP is the proxy's IP — the real client IP is lost unless explicitly passed in headers. Same for the protocol (HTTP vs HTTPS) — the backend sees plain HTTP from the proxy even if the client used HTTPS. Hence `X-Forwarded-Proto`.

**Security vulnerabilities:**

The core problem: **these headers come from the HTTP request, which the client controls.** A malicious client can send:

```
X-Forwarded-For: 10.0.0.1
X-Real-IP: 10.0.0.1
```

If the backend naively trusts these headers, the attacker can:
- **Spoof their IP** to bypass IP-based rate limiting, geo-restrictions, or allow-lists.
- **Inject internal IPs** to bypass "admin only from internal network" checks.
- **Manipulate audit logs** to hide their real IP.

**The trust-proxy chain problem:**

The safe approach is to only trust XFF entries added by **known, trusted proxies.** In Nginx:

```nginx
# Only trust the proxy at 10.0.0.1 (e.g., cloud load balancer)
set_real_ip_from 10.0.0.0/8;
real_ip_header X-Forwarded-For;
real_ip_recursive on;
```

`real_ip_recursive on` walks the XFF chain from right to left, skipping trusted IPs, and stops at the first untrusted IP — that's the real client.

In Node.js (Express), this maps to `app.set('trust proxy', ...)`:

```typescript
// Trust only the first proxy (Nginx)
app.set('trust proxy', 1);

// Or trust specific subnets
app.set('trust proxy', 'loopback, 10.0.0.0/8');
```

**The rule:** Never trust `X-Forwarded-For` at face value. Configure the exact number of trusted proxies or their IP ranges. If your app is directly exposed (no proxy), disable trust-proxy entirely — otherwise clients can forge their IP trivially.

</details>

<details>
<summary>9. Why would you serve multiple backend services from a single domain using path-based routing at the proxy layer — what are the alternatives (subdomains, separate domains), what are the tradeoffs, and how does Nginx's location block matching and priority system (exact, prefix, regex) determine which block handles a given request?</summary>

**Why path-based routing from a single domain:**

- **Simpler CORS** — All services share one origin, so cross-service API calls aren't cross-origin. No CORS headers needed between your own services.
- **Simpler TLS** — One certificate for one domain instead of a wildcard or multiple certs for subdomains.
- **Cookie sharing** — Cookies set on the domain are available to all paths. Useful for auth tokens.
- **Client simplicity** — Frontend only talks to one base URL. No need to manage multiple API hosts.

**Alternatives and tradeoffs:**

| Approach | Pros | Cons |
|---|---|---|
| **Path-based** (`api.com/users`, `api.com/orders`) | Simple TLS, no CORS, one DNS entry | Path collisions, coupling services to URL structure, harder to move services independently |
| **Subdomain-based** (`users.api.com`, `orders.api.com`) | Clean separation, independent routing, services can move independently | Wildcard cert or per-subdomain certs, CORS needed between subdomains, more DNS records |
| **Separate domains** (`users-api.com`, `orders-api.com`) | Complete isolation, independent scaling and deployment | Multiple TLS certs, CORS everywhere, more infrastructure, confusing for consumers |

**Path-based is the common choice for microservices behind an API gateway** because the simplicity wins outweigh the downsides in most cases. Subdomain-based is better when services are truly independent products.

**Nginx location block matching priority:**

Nginx evaluates location blocks in a specific priority order, NOT the order they appear in the config file:

1. **Exact match** `= /path` — Checked first. If matched, stops immediately. Fastest.
2. **Preferential prefix** `^~ /path` — Longest matching prefix with `^~` wins. If matched, skips regex evaluation.
3. **Regex** `~ /pattern` (case-sensitive) or `~* /pattern` (case-insensitive) — Evaluated in config file order. First match wins.
4. **Standard prefix** `/path` — Longest matching prefix. Used only if no regex matched.

```nginx
location = /api {           # 1st priority: exact match for /api only
    return 200 "exact";
}
location ^~ /api/internal { # 2nd priority: prefix, skip regex
    proxy_pass http://internal;
}
location ~ ^/api/v[0-9]+ {  # 3rd priority: regex for /api/v1, /api/v2, etc.
    proxy_pass http://versioned_api;
}
location /api/ {             # 4th priority: standard prefix
    proxy_pass http://default_api;
}
```

**Common gotcha:** A regex match can override a longer prefix match. If you have `location /api/admin/` (prefix) and `location ~ ^/api/` (regex), the regex wins even though the prefix is more specific. Use `^~` on the prefix to prevent this.

</details>

## Practical — Configuration & Routing

<details>
<summary>10. Configure Nginx as a reverse proxy for two Node.js services running on different ports, routing requests based on URL path (/api/* to one service, /admin/* to another, and everything else to a static frontend) — show the full nginx.conf with upstream blocks, location blocks, and proxy_pass directives, and explain what happens when the location block matching order is wrong.</summary>

```nginx
worker_processes auto;

events {
    worker_connections 1024;
}

http {
    include       mime.types;
    sendfile      on;

    # Upstream pools
    upstream api_service {
        server 127.0.0.1:3000;
        server 127.0.0.1:3001;  # second instance
    }

    upstream admin_service {
        server 127.0.0.1:4000;
    }

    server {
        listen 80;
        server_name example.com;

        # API service — all /api/* requests
        location /api/ {
            proxy_pass http://api_service;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }

        # Admin service — all /admin/* requests
        location /admin/ {
            proxy_pass http://admin_service;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }

        # Static frontend — everything else
        location / {
            root /var/www/frontend;
            index index.html;
            try_files $uri $uri/ /index.html;  # SPA fallback
        }
    }
}
```

**How this works:** Nginx uses longest prefix matching for standard `location` blocks. A request to `/api/users` matches both `/` and `/api/` — but `/api/` is the longer prefix, so it wins. The order in the config file doesn't matter for prefix locations.

**What happens when matching goes wrong:**

Scenario 1: **Trailing slash mismatch.** If you write `location /api` (no trailing slash) with `proxy_pass http://api_service`, a request to `/api-docs` also matches because `/api` is a prefix of `/api-docs`. Always include the trailing slash: `location /api/`.

Scenario 2: **Regex overriding prefix.** If you add a broad regex like `location ~ \.php$` and a request to `/api/endpoint.php` comes in, the regex matches and overrides the `/api/` prefix. The request goes to the PHP handler instead of the API service. Fix: use `^~` on the prefix to block regex evaluation: `location ^~ /api/`.

Scenario 3: **proxy_pass with and without trailing slash.** This is subtle:

```nginx
# Request: /api/users
location /api/ {
    proxy_pass http://api_service;    # backend sees: /api/users (path preserved)
}

location /api/ {
    proxy_pass http://api_service/;   # backend sees: /users (path stripped)
}
```

The trailing slash on `proxy_pass` tells Nginx to replace the matched location prefix with the URI in `proxy_pass`. This is a frequent source of unexpected 404s — the backend gets a different path than expected.

</details>

<details>
<summary>11. Configure Nginx to load balance across three backend instances using least-connections with health checks — show the upstream block configuration, explain how max_fails and fail_timeout work, what happens when all upstreams are marked unhealthy, and how you'd add a new backend instance without restarting Nginx.</summary>

```nginx
upstream backend {
    least_conn;

    server 10.0.0.1:3000 max_fails=3 fail_timeout=30s;
    server 10.0.0.2:3000 max_fails=3 fail_timeout=30s;
    server 10.0.0.3:3000 max_fails=3 fail_timeout=30s;
}

server {
    listen 80;
    server_name api.example.com;

    location / {
        proxy_pass http://backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_next_upstream error timeout http_502 http_503;
        proxy_next_upstream_tries 2;  # try at most 2 other upstreams
    }
}
```

**How `max_fails` and `fail_timeout` work together:**

These two parameters form Nginx's passive health check system:

- `max_fails=3` — After 3 consecutive failed requests (connection error, timeout, or status codes matched by `proxy_next_upstream`), Nginx marks the server as unavailable.
- `fail_timeout=30s` — Serves two purposes: (1) the window in which `max_fails` are counted, and (2) how long the server stays marked unavailable before Nginx tries it again.

So with `max_fails=3 fail_timeout=30s`: if 3 requests fail within any 30-second window, the server is taken out of rotation for 30 seconds. After that, Nginx sends it a single request to check if it's back — if it succeeds, the server rejoins the pool.

**When all upstreams are marked unhealthy:**

Nginx has no healthy servers to send traffic to. Behavior depends on the situation:
- When all upstreams are marked failed within their `fail_timeout` windows, Nginx returns **502 Bad Gateway** for incoming requests — there is no immediate "reset all to live" mechanism.
- Once a server's `fail_timeout` expires, Nginx sends it a single request to check recovery. If it succeeds, the server rejoins the pool.
- If `backup` servers are defined, those are tried first before all primary servers are exhausted.

**Adding a new backend without restart:**

1. Edit the Nginx config to add the new server to the `upstream` block.
2. Validate: `nginx -t`
3. Graceful reload: `nginx -s reload`

Reload sends a SIGHUP to the master process. The master spawns new workers with the updated config while old workers finish their in-flight requests and then exit. **Zero downtime** — no connections are dropped.

**Note on active health checks:** The passive system described above only detects failures when real traffic is sent. Nginx Plus (commercial) and the open-source `nginx_upstream_check_module` provide active health checks that periodically probe backends independently of traffic. For open-source Nginx, the passive check is all you get natively.

</details>

<details>
<summary>12. Configure TLS termination in Nginx with certificate and key files, redirect all HTTP traffic to HTTPS, set up HSTS headers, and configure OCSP stapling — show the server blocks for both port 80 and 443, explain the ssl_protocols and ssl_ciphers choices, and what breaks if the certificate chain is incomplete.</summary>

```nginx
# Redirect all HTTP to HTTPS
server {
    listen 80;
    server_name example.com www.example.com;
    return 301 https://$host$request_uri;
}

# HTTPS server
server {
    listen 443 ssl http2;
    server_name example.com www.example.com;

    # Certificate files — fullchain includes intermediate certs
    ssl_certificate     /etc/nginx/ssl/fullchain.pem;
    ssl_certificate_key /etc/nginx/ssl/privkey.pem;

    # Protocol versions — TLS 1.2 and 1.3 only
    ssl_protocols TLSv1.2 TLSv1.3;

    # Cipher suites — strong ciphers, forward secrecy preferred
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers off;  # let client choose (modern best practice for TLS 1.3)

    # Session caching — avoid repeated TLS handshakes
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 1d;
    ssl_session_tickets off;  # disable for forward secrecy

    # OCSP stapling — Nginx fetches OCSP response and sends it during handshake
    ssl_stapling on;
    ssl_stapling_verify on;
    ssl_trusted_certificate /etc/nginx/ssl/fullchain.pem;
    resolver 8.8.8.8 8.8.4.4 valid=300s;
    resolver_timeout 5s;

    # HSTS — tell browsers to always use HTTPS for this domain
    add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;

    location / {
        proxy_pass http://backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

**Protocol and cipher choices explained:**

- **TLS 1.0 and 1.1 are disabled** — Both have known vulnerabilities (BEAST, POODLE variants) and are deprecated by all major browsers since 2020. PCI-DSS also requires TLS 1.2+.
- **TLS 1.3** — Faster handshake (1-RTT vs 2-RTT), no legacy cipher support, mandatory forward secrecy. Enable it whenever possible.
- **ECDHE ciphers** — Provide forward secrecy: even if the server's private key is compromised later, past sessions can't be decrypted. The `ECDHE` prefix means ephemeral Diffie-Hellman key exchange.
- **`ssl_prefer_server_ciphers off`** — With TLS 1.3, the client's cipher preference is typically better because modern browsers have good defaults. For TLS 1.2-only setups, `on` gives you more control.

**What breaks with an incomplete certificate chain:**

The `ssl_certificate` file must contain the full chain: your server cert first, then intermediate CA certs, in order. If intermediate certs are missing:

- **Desktop browsers** may still work — they often cache intermediates or fetch them from the Authority Information Access (AIA) URL in the cert. This creates a false sense of "it works."
- **Mobile browsers, API clients, and curl** will fail with errors like `SSL: certificate verify failed` or `unable to get local issuer certificate`. They don't fetch missing intermediates.
- **OCSP stapling breaks** — Nginx can't verify the OCSP response without the full chain, and `ssl_stapling_verify on` causes it to fail silently (stapling just doesn't happen).

**How to verify:** `openssl s_client -connect example.com:443 -showcerts` shows the chain sent by the server. You should see 2-3 certificates (server + intermediates). If you only see one, the chain is incomplete.

</details>

<details>
<summary>13. Configure rate limiting in Nginx using the limit_req module — show how to set up a shared memory zone, define the rate, configure burst and nodelay, apply different rate limits to different location blocks (e.g., stricter limits on /api/login), and explain what the client experiences when they hit the limit vs when they exceed the burst.</summary>

```nginx
http {
    # Define shared memory zones for rate limiting
    # Zone name : size : rate (requests per second per key)
    limit_req_zone $binary_remote_addr zone=general:10m rate=10r/s;
    limit_req_zone $binary_remote_addr zone=login:10m rate=1r/s;

    # Custom error page for rate-limited requests
    limit_req_status 429;

    server {
        listen 80;
        server_name api.example.com;

        # General API — 10 req/s per IP, burst of 20
        location /api/ {
            limit_req zone=general burst=20 nodelay;
            proxy_pass http://backend;
        }

        # Login endpoint — strict: 1 req/s per IP, burst of 5
        location /api/login {
            limit_req zone=login burst=5 nodelay;
            proxy_pass http://backend;
        }

        # Static assets — no rate limiting
        location /static/ {
            root /var/www;
        }
    }
}
```

**How the pieces work:**

`limit_req_zone` defines the zone:
- `$binary_remote_addr` — The key to rate limit by (client IP in binary form, 16 bytes for IPv6). You could also use `$server_name`, `$uri`, or a combination.
- `zone=general:10m` — Name and size. 10 MB of shared memory holds ~160,000 IP addresses.
- `rate=10r/s` — The sustained rate. Internally, this is tracked as "1 request per 100ms" — Nginx uses a leaky bucket algorithm (as covered in question 6).

`limit_req` applies the zone to a location:
- `burst=20` — The bucket can hold 20 excess requests beyond the rate.
- `nodelay` — Process burst requests immediately instead of queuing them.

**What the client experiences at different traffic levels:**

Scenario: `rate=10r/s burst=20 nodelay`

1. **Under 10 req/s** — All requests pass through normally. No delay, no rejection.

2. **Burst to 30 req/s** — The first 20 excess requests (the burst) are processed immediately (thanks to `nodelay`). The client sees no delay. But those 20 "slots" in the burst buffer are now used and refill at 10/s. If more requests come before slots refill, they're rejected.

3. **Sustained above 10 req/s** — Once the burst buffer is exhausted, every request above 10/s gets a 429 (or 503 without `limit_req_status`). The client sees immediate rejection.

**Without `nodelay`:**

Burst requests are queued and released at the base rate. The client experiences increasing latency — a request that's 15th in the burst queue waits 1.5 seconds. This smooths traffic but creates unpredictable response times. Most APIs prefer `nodelay` for better client experience.

**Without `burst`:**

Any request that arrives faster than the rate is immediately rejected. Even two requests in quick succession would cause the second to be rejected if they arrive within the same rate window. This is too aggressive for most use cases — always set a reasonable `burst`.

**Multiple zones on one location:**

You can stack rate limits:

```nginx
location /api/ {
    limit_req zone=per_ip burst=20 nodelay;
    limit_req zone=per_server burst=1000 nodelay;  # global limit
    proxy_pass http://backend;
}
```

Both limits must pass for the request to proceed.

</details>

<details>
<summary>14. Configure Nginx to proxy WebSocket connections and SSE streams to a Node.js backend — show the configuration for the connection upgrade mechanism (Upgrade and Connection headers), explain what proxy_read_timeout and proxy_send_timeout values you'd set for long-lived connections, why proxy_buffering must be off for SSE, and what symptoms appear when these settings are wrong.</summary>

```nginx
# Map to handle connection upgrade — sets $connection_upgrade
# to "upgrade" when client sends Upgrade header, empty string otherwise
map $http_upgrade $connection_upgrade {
    default upgrade;
    ''      close;
}

upstream ws_backend {
    server 127.0.0.1:3000;
}

upstream sse_backend {
    server 127.0.0.1:3001;
}

server {
    listen 80;
    server_name app.example.com;

    # WebSocket endpoint
    location /ws/ {
        proxy_pass http://ws_backend;
        proxy_http_version 1.1;

        # Connection upgrade headers — critical for WebSocket
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;

        # Long timeouts for persistent connections
        proxy_read_timeout 3600s;
        proxy_send_timeout 3600s;

        # Standard proxy headers
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    # SSE endpoint
    location /events/ {
        proxy_pass http://sse_backend;
        proxy_http_version 1.1;

        # Disable buffering — events must stream immediately
        proxy_buffering off;
        proxy_cache off;

        # Long timeout for the open stream
        proxy_read_timeout 86400s;  # 24 hours

        # Chunked transfer for streaming
        proxy_set_header Connection '';
        chunked_transfer_encoding on;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

**Timeout values and reasoning:**

- **`proxy_read_timeout`** — How long Nginx waits for the backend to send data. For WebSocket/SSE, this is the maximum silence period. Set it high (1-24 hours) or rely on application-level pings. If your app sends WebSocket pings every 30 seconds, `proxy_read_timeout 60s` is safe — but 3600s gives more margin.
- **`proxy_send_timeout`** — How long Nginx waits for the client to accept data. Relevant for WebSocket when the server pushes data. Less critical for SSE (server-to-client only). Match it to `proxy_read_timeout` for WebSocket.
- **`proxy_connect_timeout`** — How long to establish the initial connection to the backend. Keep this short (5-10s) — if the backend isn't reachable, fail fast.

**Why `proxy_buffering off` for SSE:**

As covered in Q7, `proxy_buffering off` is required so events stream to the client immediately rather than accumulating in Nginx's buffers. WebSocket doesn't need this because after the 101 upgrade, Nginx operates as a TCP tunnel.

**Symptoms of misconfiguration:**

| Misconfiguration | Symptom |
|---|---|
| Missing `proxy_http_version 1.1` | WebSocket handshake fails — client gets 400 Bad Request or connection drops |
| Missing `Upgrade`/`Connection` headers | Backend receives normal HTTP request, responds with 200 instead of 101, WebSocket never establishes |
| `proxy_read_timeout 60s` (default) | Connections drop after 60 seconds of silence — users see "WebSocket disconnected" or SSE stream ends |
| `proxy_buffering on` for SSE | Events arrive in delayed bursts instead of real-time; client appears to "freeze" then suddenly updates |
| Missing `proxy_set_header Connection ''` for SSE | Nginx may try to use keepalive behavior that interferes with the persistent response stream |

</details>

<details>
<summary>15. Configure the proxy headers (X-Forwarded-For, X-Real-IP, X-Forwarded-Proto, Host) in Nginx for a setup where Nginx sits behind a cloud load balancer — show the proxy_set_header directives, explain how set_header vs add_header behaves differently in nested location blocks, and demonstrate the configuration that prevents IP spoofing through X-Forwarded-For header injection.</summary>

**Architecture:** Client -> Cloud LB (e.g., AWS ALB) -> Nginx -> Node.js backend

The cloud LB already sets `X-Forwarded-For` with the client IP. Nginx must preserve that and correctly forward it to the backend.

```nginx
http {
    # Trust the cloud load balancer's IP range to extract real client IP
    set_real_ip_from 10.0.0.0/8;        # VPC range where ALB lives
    set_real_ip_from 172.16.0.0/12;     # or your specific LB subnet
    real_ip_header X-Forwarded-For;
    real_ip_recursive on;                # walk the chain, skip trusted IPs

    server {
        listen 80;
        server_name api.example.com;

        location / {
            proxy_pass http://backend;

            # Forward the original Host header (what the client requested)
            proxy_set_header Host $host;

            # Real client IP (resolved by real_ip module from XFF chain)
            proxy_set_header X-Real-IP $remote_addr;

            # Append Nginx's view of the remote addr to the existing XFF chain
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

            # Protocol the client used (HTTPS terminated at LB or Nginx)
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }
}
```

**How `real_ip_recursive on` prevents IP spoofing:**

As covered in Q8, clients can inject fake IPs into XFF headers. With `real_ip_recursive on` and `set_real_ip_from` scoped to your infrastructure IPs only, Nginx walks the chain from right to left, skipping trusted proxies. The first untrusted IP becomes `$remote_addr`.

The key difference in a cloud LB setup: **`set_real_ip_from` must include the LB's IP range** (e.g., VPC CIDR) so Nginx skips past the LB to find the real client IP. If the LB's IP is not in the trusted range, Nginx stops there and misidentifies the LB as the client.

**`proxy_set_header` vs `add_header` in nested locations:**

These are fundamentally different directives:

- **`proxy_set_header`** — Sets a header on the **request sent to the upstream**. If you define ANY `proxy_set_header` in a `location` block, it **completely replaces** all `proxy_set_header` directives inherited from the parent `server` or `http` context. This is the most common gotcha.

```nginx
server {
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

    location /api/ {
        # This REPLACES all three headers above — only Connection is sent
        proxy_set_header Connection "upgrade";
        proxy_pass http://backend;
    }
}
```

Fix: repeat all headers in the location block, or use `include` with a shared snippet:

```nginx
# /etc/nginx/proxy-headers.conf
proxy_set_header Host $host;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;

# Then in each location:
location /api/ {
    include proxy-headers.conf;
    proxy_set_header Connection "upgrade";
    proxy_pass http://backend;
}
```

- **`add_header`** — Adds a header to the **response sent to the client**. Same inheritance behavior: defining any `add_header` in a child context replaces all inherited `add_header` directives. The `always` flag makes it apply to error responses too (not just 2xx/3xx).

</details>

## Practical — Observability & Debugging

<details>
<summary>16. Your application is intermittently returning 502 Bad Gateway through Nginx — walk through the exact steps to diagnose the root cause: what to check in Nginx error logs, how to determine whether the upstream is crashing vs refusing connections vs timing out during response, what the difference between "connection refused" and "no live upstreams" means, and how to fix each case.</summary>

**Step 1: Check the Nginx error log**

```bash
tail -f /var/log/nginx/error.log | grep 502
# or filter for upstream errors
grep "upstream" /var/log/nginx/error.log | tail -50
```

The error message tells you exactly what happened. Here are the key messages and what they mean:

**"connect() failed (111: Connection refused)"**

The backend is not listening on the expected port. Causes:
- The Node.js process crashed or hasn't started yet.
- The process is running but bound to a different port or interface (e.g., listening on `127.0.0.1` but Nginx connects to the container's IP).
- In Docker/K8s, the container is not ready yet (no readiness probe or probe is too lenient).

**Fix:** Check if the process is running (`ss -tlnp | grep 3000`), verify the port matches the upstream config, ensure the backend binds to `0.0.0.0` (not `127.0.0.1`) when running in containers.

**"no live upstreams"**

All servers in the upstream block have been marked as failed (exceeded `max_fails` within `fail_timeout`). Nginx has no backend to send traffic to.

**Fix:** This is a symptom — the root cause is why all backends failed. Check backend logs, resource exhaustion (CPU, memory, file descriptors), or network issues. As covered in question 11, Nginx will eventually retry failed servers.

**"upstream prematurely closed connection while reading response header"**

Nginx connected to the backend and sent the request, but the backend closed the connection before sending a complete response. Causes:
- Backend crashed mid-request (unhandled exception, OOM kill).
- Backend has a shorter timeout than Nginx and gave up.
- Backend's connection pool or worker limit is exhausted and it's dropping connections.

**Fix:** Check backend application logs for errors. Look for OOM kills in system logs (`dmesg | grep -i oom`). Increase backend timeout or connection limits.

**Step 2: Correlate with access logs**

```bash
# Check timing in access log — $upstream_response_time shows backend latency
grep " 502 " /var/log/nginx/access.log | tail -20
```

If `$upstream_response_time` is `0.000`, the connection was refused immediately. If it's close to `proxy_connect_timeout`, it was a connection timeout (backend overloaded, not refusing but not accepting).

**Step 3: Test the backend directly**

```bash
# Bypass Nginx — hit the backend directly
curl -v http://127.0.0.1:3000/health
```

If this succeeds, the issue is in Nginx's config or network path. If it fails, the issue is the backend.

**Step 4: Check system resources**

```bash
# File descriptor limits (common cause of "connection refused" under load)
cat /proc/$(pgrep -f "node")/limits | grep "open files"

# Check if backend is OOM
dmesg | grep -i "oom\|killed"

# Check socket state
ss -s  # summary of socket states
```

**Intermittent 502s specifically** often indicate:
- Backend restarts during deployments (fix: rolling deploys with health checks).
- Memory pressure causing periodic OOM kills (fix: right-size memory, add swap alerts).
- Connection limit exhaustion under traffic spikes (fix: increase `ulimit -n`, tune connection pooling).

</details>

<details>
<summary>17. Users report that certain API requests hang for exactly 60 seconds and then return 504 Gateway Timeout — walk through the debugging process: how to identify which timeout is being hit (proxy_read_timeout, proxy_connect_timeout, or upstream application timeout), how to check whether the backend is actually slow vs Nginx is misconfigured, and what the correct timeout tuning strategy looks like for different types of endpoints.</summary>

**The 60-second clue:**

60 seconds is the default for both `proxy_read_timeout` and `proxy_connect_timeout` in Nginx. The exact timing tells you which one:

- **Exactly 60s with no response at all** — Likely `proxy_connect_timeout`. Nginx can't even establish a TCP connection to the backend. The backend is unreachable or its listen backlog is full.
- **Exactly 60s, request was sent but no response** — Likely `proxy_read_timeout`. Nginx connected, sent the request, but the backend hasn't sent any response data within 60s.

**Step 1: Identify which timeout is firing**

Check the Nginx error log — it tells you explicitly:

```
upstream timed out (110: Connection timed out) while connecting to upstream  → proxy_connect_timeout
upstream timed out (110: Connection timed out) while reading response header → proxy_read_timeout
```

**Step 2: Is the backend actually slow?**

```bash
# Hit the backend directly with timing
time curl -v http://127.0.0.1:3000/api/slow-endpoint

# Check the specific endpoint's processing time in application logs
# In Node.js, look at your request logging middleware output
```

If the direct call also takes 60+ seconds, the backend is genuinely slow — the problem isn't Nginx.

If the direct call is fast but Nginx times out, check:
- Is Nginx connecting to the right host/port?
- Is there a network issue between Nginx and the backend (firewall, security group)?
- Is the backend rate-limiting or queuing connections from Nginx's IP?

**Step 3: Check if it's only certain endpoints**

504 on "certain API requests" suggests those specific endpoints are slow. Common causes:
- Missing database indexes causing full table scans.
- Downstream service calls that hang (the backend is waiting for another service).
- Large payload processing (file uploads, report generation).
- Deadlocks or lock contention in the database.

**Step 4: Correct timeout tuning strategy**

Don't just increase all timeouts globally — that masks problems and ties up connections:

```nginx
# Global defaults — keep tight
proxy_connect_timeout 5s;    # if backend isn't reachable in 5s, fail fast
proxy_read_timeout 30s;      # most API calls should respond in 30s

# Specific overrides for known slow endpoints
location /api/reports/ {
    proxy_read_timeout 300s;  # reports can take up to 5 minutes
    proxy_pass http://backend;
}

location /api/uploads/ {
    proxy_read_timeout 120s;
    client_max_body_size 100m;
    proxy_pass http://backend;
}

# Health check — fast timeout
location /health {
    proxy_read_timeout 5s;
    proxy_pass http://backend;
}
```

**Timeout tuning principles:**

- **`proxy_connect_timeout`** should always be short (3-10s). If you can't connect in 10 seconds, the backend is down or overloaded — waiting longer doesn't help.
- **`proxy_read_timeout`** varies by endpoint. Fast APIs: 10-30s. Report generation: 60-300s. WebSocket/SSE: 3600s+ (as covered in question 14).
- **Set timeouts at the location level**, not globally. This keeps fast endpoints fast and only gives extra time where it's actually needed.
- **The backend should always timeout before Nginx does.** If your application has a 30s request timeout, set `proxy_read_timeout` to 35s. This way the backend returns a proper error response instead of Nginx generating a generic 504.

</details>

<details>
<summary>18. After a deployment, some requests are failing because Nginx is still sending traffic to old backend IPs — explain why this happens (DNS caching in Nginx with upstream blocks), how Nginx resolves DNS for upstream servers at startup vs at runtime, and show the configuration changes (resolver directive, variables in proxy_pass) that fix stale upstream IP issues in dynamic environments like Kubernetes or auto-scaling groups.</summary>

**Why this happens:**

When you define servers in an `upstream` block by hostname, Nginx resolves DNS **once at startup** (or reload) and caches the resulting IPs permanently:

```nginx
upstream backend {
    server my-service.default.svc.cluster.local:3000;  # resolved at startup only
}
```

In Kubernetes, when a deployment rolls out, pods get new IPs. The old pods terminate, but Nginx still has their IPs cached. Requests to those IPs fail with 502 (connection refused) or hang until timeout.

The same problem occurs with auto-scaling groups behind a DNS record — new instances get new IPs, but Nginx doesn't re-resolve.

**How Nginx resolves DNS — startup vs runtime:**

- **`upstream` block with hostnames** — DNS resolved at config load time (`nginx -s reload` or startup). Results are cached forever. No TTL is respected. This is the default and the cause of stale IP issues.
- **Variable in `proxy_pass`** — DNS resolved per-request (or per the `resolver` directive's `valid` TTL). This is the fix.

**The fix — use variables to force runtime DNS resolution:**

```nginx
http {
    # Configure a DNS resolver (kube-dns in K8s, or your cloud DNS)
    resolver 10.96.0.10 valid=10s ipv6=off;  # kube-dns IP, re-resolve every 10s
    resolver_timeout 5s;

    server {
        listen 80;

        location /api/ {
            # Store the hostname in a variable — this triggers runtime resolution
            set $backend "http://my-service.default.svc.cluster.local:3000";
            proxy_pass $backend;

            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }
    }
}
```

**Key detail:** When `proxy_pass` contains a **variable**, Nginx resolves the hostname at request time using the configured `resolver`. Without a variable (hardcoded hostname in `proxy_pass` or `upstream` block), DNS is resolved once at startup.

**Tradeoffs of variable-based proxy_pass:**

| Aspect | `upstream` block | Variable `proxy_pass` |
|---|---|---|
| DNS resolution | At startup/reload only | Per request (or per `valid` TTL) |
| Load balancing | Full algorithm support (least_conn, ip_hash, etc.) | Round-robin only (across DNS A records) |
| Health checks | max_fails/fail_timeout passive checks | None — Nginx doesn't track individual IPs |
| Keepalive connections | Supported via `keepalive` directive | Not directly supported |
| Dynamic environments | Breaks on IP changes | Works correctly |

**For Kubernetes specifically**, the best approach depends on your setup:

1. **Headless Service + variable proxy_pass** — DNS returns individual pod IPs, Nginx re-resolves. Simple but no advanced load balancing.
2. **ClusterIP Service + upstream block + periodic reload** — Let Kubernetes handle load balancing at the Service level. Nginx points to the stable ClusterIP. The IP doesn't change, so stale DNS isn't an issue.
3. **Ingress controller** — Use Nginx Ingress Controller (or similar) instead of managing Nginx config yourself. It watches Kubernetes API for endpoint changes and updates its config automatically.

Option 2 is the most common in practice — it avoids the stale DNS problem entirely by pointing Nginx at a stable Kubernetes Service IP.

</details>

---

## Experience-Based Questions

These questions test real-world experience. Prepare by mapping them to your own projects and situations.

<details>
<summary>19. Tell me about a time you configured or optimized an Nginx reverse proxy setup in production — what was the architecture, what problems were you solving, and what configuration decisions had the biggest impact on performance or reliability?</summary>

**What the interviewer is looking for:**

- You understand reverse proxy architecture beyond just "copy-paste a config."
- You can explain WHY you made specific configuration choices, not just WHAT you configured.
- You've dealt with real operational concerns: performance under load, reliability during deployments, TLS management, debugging proxy issues.

**Key points to hit:**

- The architecture context: what services were behind the proxy, traffic volume, deployment environment (K8s, VMs, cloud).
- The specific problems that drove the work (not a greenfield "we needed a proxy" story).
- Configuration decisions with measurable impact.
- Something that went wrong or was surprising.

**Suggested structure (STAR-like):**

1. **Context** (2-3 sentences) — What was the system? What traffic patterns? What was the existing setup?
2. **Problem** — What was broken, slow, or unreliable? Be specific with symptoms.
3. **What you did** — Which Nginx configuration changes? Why those specifically? What alternatives did you consider?
4. **Impact** — Measurable results: latency reduction, error rate drop, successful zero-downtime deployments, etc.
5. **Lesson** — What would you do differently? What did this teach you?

**Example outline to personalize:**

"We had three Node.js services behind a single Nginx instance on EC2, handling about 5,000 req/s. After scaling to a second Nginx instance behind an ALB, we started seeing intermittent 502s during deployments. The root cause was twofold: (1) Nginx's upstream health checks were too slow to detect drained instances, and (2) we were missing connection draining on the backend side. I tuned `max_fails` and `fail_timeout`, added `proxy_next_upstream` to retry on 502s, and coordinated with the deployment script to wait for Nginx to mark the old instance as down before terminating it. Error rate during deployments dropped from ~2% to effectively zero."

**Pitfalls to avoid:**

- Don't describe a trivial setup with no real engineering decisions.
- Don't focus only on the "what" — interviewers want the reasoning behind your choices.
- Have specific numbers ready (traffic volume, error rates, latency) even if approximate.

</details>

<details>
<summary>20. Describe a time you debugged a proxy-related issue in production (502s, 504s, TLS errors, routing problems) — what were the symptoms, how did you narrow down whether the issue was in the proxy layer vs the backend, and what was the root cause?</summary>

**What the interviewer is looking for:**

- Systematic debugging methodology — not guessing.
- Ability to isolate which layer is causing the problem (client, proxy, backend, network).
- Understanding of what Nginx error messages mean and what logs to check.
- Composure under production pressure.

**Key points to hit:**

- Clear symptoms with specifics (which status codes, how often, which endpoints, when it started).
- The diagnostic steps IN ORDER — show your methodology.
- How you isolated the proxy layer from the backend (direct backend calls, log correlation, timing analysis).
- The root cause and the fix.
- What you did to prevent recurrence.

**Suggested structure:**

1. **Symptoms** (be specific) — "We saw a spike to 15% 502 error rate starting at 14:30, affecting all endpoints, but only for clients coming through the EU Nginx instance."
2. **Initial triage** — What did you check first? (Nginx error logs, backend health, recent deployments, traffic patterns)
3. **Isolation** — How did you determine it was the proxy layer? (Direct backend calls succeeded? Error messages pointed to connection issues? Timing matched Nginx timeouts?)
4. **Root cause** — The specific technical cause. The more specific, the better.
5. **Fix and prevention** — Immediate fix + what you changed to prevent it.

**Example outline to personalize:**

"After a routine deployment, about 10% of requests started returning 504 with exactly 60-second delays. I checked Nginx error logs and saw `upstream timed out while reading response header`. Direct curl to the backend was fast, so Nginx could connect but wasn't getting responses. Turned out the deployment changed the backend's listen address from `0.0.0.0` to `127.0.0.1` in a config refactor — Nginx was connecting to the container IP, which was no longer being served. The connection established (TCP handshake succeeded against the kernel) but the application never received it. Fix was reverting the bind address. Prevention: added a health check endpoint that Nginx probed, and a CI test validating the listen configuration."

**Pitfalls to avoid:**

- Don't describe an issue where you just "restarted the server and it fixed itself." Show real diagnosis.
- Avoid vague descriptions — "it was a network issue" without explaining how you identified that.
- Make sure you can explain the technical WHY behind the root cause, not just the symptom.

</details>
