# Networking & Protocols

> **18 questions** — 13 theory, 5 practical

- End-to-end request lifecycle: DNS resolution, TCP connect, TLS handshake, HTTP request/response — the full path of a network call
- TCP/IP model: application, transport, network, link layers — encapsulation, data flow, and how layering guides debugging
- TCP reliability: three-way handshake, teardown, sequence numbers, sliding window, flow control, congestion control basics (slow start)
- UDP vs TCP tradeoffs and which protocols build on each
- DNS resolution: recursive vs iterative queries, caching, TTL, failure modes
- HTTP/1.1 vs HTTP/2 vs HTTP/3: head-of-line blocking, multiplexing, binary framing, HTTP/3 over QUIC (UDP-based, connection migration)
- HTTP semantics: method idempotency (GET/PUT/DELETE vs POST), status code families, commonly misused codes (401 vs 403, 502 vs 503), retry-safe vs non-retry-safe responses
- gRPC: HTTP/2 transport, Protobuf serialization, streaming modes (unary, server, client, bidirectional), and tradeoffs vs REST
- WebSockets vs SSE vs HTTP polling: upgrade handshake, frames, persistent connection models (keep-alive vs multiplexing vs WebSocket), operational costs, when each fits
- Proxies and load balancers: L4 vs L7 differences, X-Forwarded-For/X-Real-IP headers, TLS termination points, and how proxies affect client IP visibility
- HTTP caching: Cache-Control, ETag, Vary, conditional requests, cache poisoning
- CORS: preflight mechanism, credentialed requests, common misconfigurations and wildcard pitfalls

---

## Foundational

<details>
<summary>1. What happens end-to-end when a browser makes an HTTPS request — walk through the full lifecycle from DNS resolution through TCP connection, TLS handshake, and HTTP request/response, explaining what each step accomplishes and why the sequence matters for understanding latency and failure modes?</summary>

**1. DNS Resolution (~1 RTT, often cached)**

The browser needs an IP address. It checks the browser cache, then the OS cache, then queries a recursive DNS resolver (typically your ISP or a public resolver like 8.8.8.8). The resolver walks the DNS hierarchy — root → TLD (`.com`) → authoritative nameserver — and returns the IP. This step introduces latency and is a common failure point: if DNS is slow or the record has bad TTL, every request is affected.

**2. TCP Three-Way Handshake (1 RTT)**

The browser opens a TCP connection to the resolved IP on port 443:
- Client sends **SYN** (synchronize)
- Server responds with **SYN-ACK**
- Client sends **ACK**

This establishes a reliable, ordered, bidirectional byte stream. It's 1 full round trip before any data flows.

**3. TLS Handshake (1-2 RTTs for TLS 1.2, 1 RTT for TLS 1.3)**

Over the TCP connection, the client and server negotiate encryption:
- **ClientHello**: client sends supported cipher suites, TLS version, a random value
- **ServerHello**: server picks cipher suite, sends its certificate
- **Key exchange**: client verifies the certificate chain, both sides derive session keys (using ECDHE in modern setups)
- **Finished**: both sides confirm the handshake with encrypted messages

TLS 1.3 collapses this to 1 RTT (and supports 0-RTT resumption for repeat connections). This step is where certificate errors, expired certs, and cipher mismatch failures surface.

**4. HTTP Request/Response (1+ RTTs)**

The browser sends the HTTP request (method, path, headers, optional body) over the encrypted channel. The server processes it and returns a response (status code, headers, body). With HTTP/1.1 this uses a single request-response per connection (unless keep-alive and pipelining are used). With HTTP/2, multiple requests can be multiplexed over the same connection.

**Why the sequence matters:**

- **Latency stacks**: DNS + TCP + TLS + HTTP = minimum 3-4 RTTs before the first byte of data arrives (TTFB). On a 100ms RTT connection, that's 300-400ms minimum.
- **Failure isolation**: Knowing the sequence tells you where to look. DNS failure → no connection attempt. TCP timeout → network/firewall issue. TLS error → certificate/cipher problem. HTTP error → application issue.
- **Optimization targets**: Connection reuse (keep-alive, HTTP/2 multiplexing) eliminates repeated TCP+TLS handshakes. DNS prefetching and pre-connect hints reduce cold-start latency. TLS 1.3 and 0-RTT resumption shave round trips.

</details>

<details>
<summary>2. Why is the TCP/IP model organized into layers (application, transport, network, link) instead of being a single monolithic protocol — what problem does layering solve, how does encapsulation work as data moves through the stack, and how does understanding layers help you debug networking issues faster?</summary>

**Why layering exists:**

Layering separates concerns so each layer can evolve independently. HTTP doesn't care whether it runs over TCP or QUIC. TCP doesn't care whether the link layer is Ethernet or Wi-Fi. This separation means you can swap TLS versions without changing HTTP, or switch from IPv4 to IPv6 without rewriting your application.

**The four layers and what they own:**

| Layer | Responsibility | Example protocols | Unit |
|-------|---------------|-------------------|------|
| **Application** | Application semantics (request/response, encoding) | HTTP, DNS, gRPC, SMTP | Message |
| **Transport** | Reliable (or unreliable) delivery between processes | TCP, UDP | Segment/Datagram |
| **Network** | Routing between hosts across networks | IP (v4/v6) | Packet |
| **Link** | Delivery on a single physical/logical link | Ethernet, Wi-Fi | Frame |

**Encapsulation:**

As data moves down the stack, each layer wraps the data from the layer above with its own header:

```
Application:  [HTTP headers + body]
Transport:    [TCP header | HTTP data]
Network:      [IP header | TCP segment]
Link:         [Ethernet header | IP packet | Ethernet trailer]
```

On the receiving end, each layer strips its header and passes the payload up. The receiver's TCP layer doesn't see Ethernet headers; the application doesn't see IP headers.

**How this helps debugging:**

When something fails, layers tell you where to look:

- **Can't resolve hostname** → DNS (application layer) — check `dig` or `nslookup`
- **Connection refused / timeout** → TCP (transport) or IP (network) — check `telnet host port`, firewall rules, security groups
- **TLS handshake failure** → Transport/application boundary — check `openssl s_client`, certificate validity
- **HTTP 502/503** → Application layer — the connection worked but the upstream service didn't respond correctly

Without layering awareness, every network issue looks the same: "it doesn't work." With it, you narrow the problem space immediately.

</details>

<details>
<summary>3. Why do both TCP and UDP exist instead of just one transport protocol — what are the fundamental tradeoffs between reliability and performance, which well-known protocols build on each (and why), and when would you deliberately choose UDP over TCP in a backend system?</summary>

**The fundamental tradeoff:**

TCP provides reliable, ordered delivery at the cost of latency (handshake, retransmissions, head-of-line blocking). UDP provides fast, fire-and-forget delivery with no guarantees — the application handles reliability if it needs it.

| | TCP | UDP |
|---|-----|-----|
| **Connection** | Stateful (3-way handshake) | Connectionless |
| **Ordering** | Guaranteed, in-order | No ordering |
| **Reliability** | Retransmits lost packets | No retransmission |
| **Flow/congestion control** | Built-in | None (application must manage) |
| **Overhead** | 20-byte header minimum | 8-byte header |

**Protocols built on each:**

- **TCP**: HTTP/1.1, HTTP/2, SMTP, FTP, SSH, PostgreSQL wire protocol — anything where you need every byte delivered in order
- **UDP**: DNS (single query-response, retry at app level), DHCP, QUIC (HTTP/3), video/voice streaming (RTP), game networking, service discovery protocols

**Why QUIC chose UDP:**

QUIC (and therefore HTTP/3) runs over UDP because it needs to implement its own reliability and congestion control at the application layer. This lets it avoid TCP's head-of-line blocking problem — if one stream's packet is lost, other streams on the same connection aren't blocked. It also enables connection migration (when a mobile client switches from Wi-Fi to cellular, the connection survives because it's identified by a connection ID, not a TCP 4-tuple).

**When to choose UDP in a backend system:**

- **Metrics/telemetry shipping**: StatsD sends UDP datagrams — losing a metric data point is acceptable, but blocking on retransmissions isn't
- **Service discovery**: DNS-based discovery uses UDP for speed
- **Real-time audio/video**: Late data is worse than lost data — retransmitting a packet that arrives after its playback window is pointless
- **Custom protocols needing stream multiplexing**: When you need TCP-like reliability but without TCP's head-of-line blocking (this is literally why QUIC exists)

In most backend work (APIs, databases, message queues), TCP is the right default. Choose UDP only when you have a specific reason to trade reliability for latency or control.

</details>

## Conceptual Depth

<details>
<summary>4. How does TCP establish and tear down connections reliably -- explain the three-way handshake and four-way teardown, what sequence numbers and acknowledgments accomplish, and why the protocol needs both a connection setup and an explicit teardown phase rather than just sending data directly?</summary>

**Three-way handshake (connection establishment):**

```
Client              Server
  |--- SYN (seq=x) --->|      Client picks initial sequence number x
  |<-- SYN-ACK (seq=y, ack=x+1) --|  Server picks its own seq y, acknowledges x
  |--- ACK (ack=y+1) ->|      Client acknowledges server's seq
```

Three steps are the minimum needed to:
1. Both sides agree on initial sequence numbers (preventing stale packets from old connections being misinterpreted)
2. Both sides confirm they can send AND receive (two-way handshake wouldn't confirm the client received the server's sequence number)

**Sequence numbers and acknowledgments:**

Every byte sent has a sequence number. The receiver sends back an ACK with the next expected sequence number. This enables:
- **Ordering**: Receiver reassembles out-of-order packets using sequence numbers
- **Reliability**: Sender knows what was received. Unacknowledged segments are retransmitted after a timeout
- **Duplicate detection**: If a retransmitted segment arrives after the original, the receiver recognizes the duplicate sequence number and discards it

**Four-way teardown (connection termination):**

```
Client              Server
  |--- FIN --------->|    Client is done sending
  |<-- ACK ----------|    Server acknowledges
  |<-- FIN ----------|    Server is done sending (may come later — server might still have data to flush)
  |--- ACK --------->|    Client acknowledges
```

It's four steps (not two) because the connection is full-duplex — each direction closes independently. The server might still have data to send after the client signals it's done. The initiator then enters a `TIME_WAIT` state (typically 2x MSL, ~60 seconds) to handle any delayed packets still in transit.

**Why not just send data directly?**

Without setup: no agreed-upon sequence numbers, no confirmation both sides are ready, no way to distinguish new connections from stale packets left over from previous connections on the same port. Without teardown: no way to ensure the receiver got all data, no way to clean up connection state on both sides, and risk of orphaned connections consuming resources.

**Practical relevance:** `TIME_WAIT` accumulation is a real production issue. A server handling many short-lived connections can exhaust ephemeral ports if connections pile up in `TIME_WAIT`. Solutions: `SO_REUSEADDR`, connection pooling, or switching to long-lived connections (HTTP keep-alive, HTTP/2).

</details>

<details>
<summary>5. How do TCP's flow control and congestion control mechanisms prevent network collapse -- explain the sliding window mechanism, how flow control prevents the sender from overwhelming the receiver, and how congestion control (slow start, congestion avoidance) adapts to network capacity, including why these are separate mechanisms solving different problems?</summary>

**Two different problems:**

- **Flow control** prevents the sender from overwhelming the **receiver** (receiver's buffer is full)
- **Congestion control** prevents the sender from overwhelming the **network** (routers between sender and receiver are overloaded)

They're separate because the receiver might be able to handle 10 MB/s, but the network path between them can only carry 1 MB/s. You need both limits enforced.

**Flow control — the sliding window:**

The receiver advertises a **receive window** (rwnd) in every ACK — the amount of buffer space it has available. The sender cannot have more than rwnd bytes of unacknowledged data in flight. As the receiver processes data and frees buffer space, it advertises a larger window; if the buffer fills up, it advertises a smaller window (or zero, which pauses the sender).

```
Sender's view:
[already ACKed] [sent, awaiting ACK] [can send now] [can't send yet]
                |<--- send window --->|
                (min of rwnd and cwnd)
```

The send window slides forward as ACKs arrive — hence "sliding window."

**Congestion control — probing network capacity:**

The sender maintains a **congestion window** (cwnd) that limits how much data is in flight, independent of the receiver's window. The actual send window is `min(rwnd, cwnd)`.

Key phases:

1. **Slow start**: cwnd starts small (typically 10 segments / ~14KB) and doubles every RTT (exponential growth). Despite the name, it ramps up quickly.

2. **Congestion avoidance**: Once cwnd hits a threshold (ssthresh), growth slows to linear — increase by 1 segment per RTT. This probes for capacity carefully.

3. **Packet loss reaction**: When loss is detected:
   - **Triple duplicate ACKs** (fast retransmit): Halve cwnd, enter congestion avoidance. The network is congested but not collapsed.
   - **Timeout**: Reset cwnd to initial value, start slow start over. This is severe — something is very wrong.

**Why this matters in practice:**

- **New connections are slow**: cwnd starts small, so the first few RTTs of a new TCP connection underutilize the link. This is why connection reuse (HTTP keep-alive, HTTP/2) matters — warm connections have already ramped up cwnd.
- **High-latency links suffer more**: Slow start takes more wall-clock time to ramp up on a 200ms RTT link than a 10ms one. This is one reason QUIC's 0-RTT matters for mobile.
- **Bandwidth-delay product**: On high-bandwidth, high-latency links, you need large windows to fill the pipe. If the receiver advertises a small rwnd or the initial cwnd is too small, throughput suffers regardless of link capacity.

</details>

<details>
<summary>6. How does DNS resolution actually work — what is the difference between recursive and iterative queries, how does caching at each level (browser, OS, resolver, authoritative) interact with TTL values, and what are the common DNS failure modes that cause outages in production (stale caches, propagation delays, resolver failures)?</summary>

**Recursive vs iterative queries:**

- **Recursive**: The client asks a resolver, "Give me the answer." The resolver does all the work — querying root, TLD, and authoritative nameservers — and returns the final answer. This is what your application does when it calls `dns.resolve()`. The recursive resolver (e.g., 8.8.8.8, your cloud VPC resolver) is the one doing the heavy lifting.

- **Iterative**: The queried server responds with "I don't know, but ask this server next." The root server says "ask the .com TLD," the TLD says "ask ns1.example.com." The recursive resolver walks this chain iteratively until it gets the answer.

In practice, your app makes a recursive query to its resolver, and the resolver makes iterative queries to the DNS hierarchy.

**Caching layers and TTL:**

Each level caches results based on the TTL (time-to-live) set on the DNS record:

1. **Browser cache**: Chrome caches DNS for ~60 seconds regardless of TTL
2. **OS resolver cache**: Respects TTL from the response (e.g., `systemd-resolved`, macOS resolver)
3. **Recursive resolver cache**: The main cache — large, shared across many clients, respects TTL
4. **Authoritative nameserver**: Doesn't cache — it IS the source of truth

When you change a DNS record with a 300s TTL, it takes up to 300s for caches to expire. But some resolvers (especially ISP resolvers) don't strictly respect TTL, which is why "DNS propagation" can feel unpredictable.

**Common DNS failure modes in production:**

- **Stale cache after migration**: You move a service to a new IP, but clients with cached old records keep hitting the dead endpoint. Mitigation: lower TTL well before the migration, wait for old TTL to expire, then migrate.

- **Resolver failure**: If your VPC's DNS resolver goes down (or gets rate-limited), all DNS lookups fail and every HTTP call breaks — even though the services are fine. This is why redundant resolvers matter and why Kubernetes runs `coredns` as a cluster service.

- **TTL too low**: Very short TTLs (5-10s) mean constant DNS lookups, adding latency to every request and creating dependency on resolver availability. TTL too high means slow failover.

- **NXDOMAIN caching**: Some resolvers aggressively cache negative responses. If a record briefly doesn't exist (during a deployment misconfiguration), clients may cache the "not found" response.

- **DNS amplification / DDoS**: Authoritative nameservers can be overwhelmed, making your domain unresolvable. This is why services like Route53 and Cloudflare DNS exist — they absorb DDoS at the DNS layer.

</details>

<details>
<summary>7. Why did the web need HTTP/2 and HTTP/3 when HTTP/1.1 worked for decades — explain head-of-line blocking at the HTTP level (HTTP/1.1) vs at the TCP level (HTTP/2), how HTTP/2's binary framing and multiplexing solve the first problem, and why HTTP/3 had to move to QUIC over UDP to solve the second, including what connection migration enables that TCP cannot?</summary>

**The problem with HTTP/1.1 — HTTP-level head-of-line blocking:**

HTTP/1.1 is text-based and allows only one request-response in flight per TCP connection at a time. If you need 20 resources, they're serialized. Browsers worked around this by opening 6 parallel TCP connections per origin — wasteful (each needs its own handshake and slow-start ramp-up) and still limited.

**HTTP/2's solution — binary framing and multiplexing:**

HTTP/2 replaces the text protocol with a binary framing layer. Each request/response is a **stream**, and streams are split into **frames** that can be interleaved on a single TCP connection. This means 100 requests can be in flight simultaneously on one connection — no more 6-connection hack.

Other wins: **header compression** (HPACK) eliminates repeated headers across requests, and **server push** allows proactive resource delivery (though this feature is rarely used in practice and is being deprecated).

**The remaining problem — TCP-level head-of-line blocking:**

HTTP/2 multiplexes at the HTTP layer, but TCP still sees a single ordered byte stream. If one TCP packet is lost, TCP blocks ALL streams until that packet is retransmitted and acknowledged — even if the lost packet belongs to stream 3 and streams 1, 2, 4-10 have all their data ready. On lossy networks (mobile), this makes HTTP/2 sometimes slower than HTTP/1.1 (which used separate TCP connections, so one loss only blocked one stream).

**HTTP/3's solution — QUIC over UDP:**

HTTP/3 moves the transport to QUIC, which runs over UDP and implements its own reliability per stream. If a packet for stream 3 is lost, only stream 3 stalls — other streams continue unblocked. QUIC also handles TLS 1.3 as part of its handshake, achieving 1-RTT connection setup (vs TCP's 1 RTT + TLS's 1 RTT) and 0-RTT for repeat connections.

**Connection migration:**

TCP connections are identified by a 4-tuple: (source IP, source port, dest IP, dest port). When a mobile client switches from Wi-Fi to cellular, its source IP changes and the TCP connection dies — requiring a new handshake.

QUIC connections are identified by a **connection ID** in the QUIC header, not the IP tuple. When the network changes, the client sends packets from the new IP with the same connection ID. The server recognizes it as the same connection. No handshake, no cwnd reset, no interrupted streams. This is a significant win for mobile-first applications.

**Summary of the evolution:**

| Version | Transport | HOL Blocking | Connection Setup |
|---------|-----------|-------------|-----------------|
| HTTP/1.1 | TCP | Per-connection (HTTP level) | 2-3 RTTs (TCP + TLS) |
| HTTP/2 | TCP | Per-connection (TCP level) | 2-3 RTTs |
| HTTP/3 | QUIC/UDP | Per-stream only | 1 RTT (0-RTT resumption) |

</details>

<details>
<summary>8. Why does HTTP distinguish between safe, idempotent, and non-idempotent methods — explain how GET, PUT, DELETE, and POST differ in terms of idempotency and safety, why this matters for retry logic and caching, what the status code families (1xx-5xx) signal, and why certain codes are commonly misused (401 vs 403, 502 vs 503, 200 for errors)?</summary>

**Safe vs idempotent vs neither:**

- **Safe**: The request doesn't modify server state. Clients and intermediaries can freely repeat it, prefetch it, cache it.
- **Idempotent**: Making the same request N times has the same effect as making it once. Safe to retry on failure.
- **Neither**: Each request may cause a different side effect.

| Method | Safe | Idempotent | Typical use |
|--------|------|-----------|-------------|
| GET | Yes | Yes | Read a resource |
| HEAD | Yes | Yes | Read headers only |
| PUT | No | Yes | Replace a resource entirely |
| DELETE | No | Yes | Remove a resource |
| POST | No | No | Create a resource, trigger action |
| PATCH | No | No* | Partial update |

*PATCH can be idempotent depending on implementation (e.g., "set field X to Y" is idempotent; "increment counter" is not).

**Why this matters for retry logic and caching:**

- **Retries**: A load balancer or client can safely retry a GET or PUT on timeout — the result is the same. Retrying a POST might create duplicate orders, charge a credit card twice, or send duplicate emails. This is why POST-based mutations need application-level idempotency keys.
- **Caching**: Only safe methods (GET, HEAD) should be cached by intermediaries. A proxy that caches POST responses would serve stale data. Cache-Control headers are only meaningful on cacheable responses.

**Status code families:**

| Family | Meaning | Key codes |
|--------|---------|-----------|
| 1xx | Informational | 101 Switching Protocols (WebSocket upgrade) |
| 2xx | Success | 200 OK, 201 Created, 204 No Content |
| 3xx | Redirection | 301 Permanent, 302 Found, 304 Not Modified |
| 4xx | Client error | 400 Bad Request, 401, 403, 404, 409 Conflict, 429 Too Many Requests |
| 5xx | Server error | 500 Internal, 502 Bad Gateway, 503 Service Unavailable, 504 Gateway Timeout |

**Commonly misused codes:**

- **401 vs 403**: 401 means "not authenticated" — the server doesn't know who you are (missing/invalid token). 403 means "authenticated but not authorized" — the server knows who you are but you don't have permission. Many APIs return 403 for both, which makes debugging harder.

- **502 vs 503**: 502 means the gateway/proxy got an invalid response from the upstream server (upstream crashed, returned garbage). 503 means the service is temporarily unavailable (overloaded, in maintenance). The distinction matters for retry strategy: 503 often comes with a Retry-After header; 502 means something is broken upstream.

- **200 for errors**: Some APIs return 200 with `{"error": "something broke"}` in the body. This breaks HTTP-aware infrastructure — load balancers, caches, monitoring, and retry logic all rely on status codes to distinguish success from failure.

</details>

<details>
<summary>9. Why would you choose gRPC over REST for service-to-service communication — explain how gRPC uses HTTP/2 as its transport, why Protobuf serialization matters for performance and type safety, what the four streaming modes (unary, server streaming, client streaming, bidirectional) enable, and what the real-world tradeoffs are (tooling, debugging, browser support, learning curve)?</summary>

**How gRPC works:**

gRPC uses HTTP/2 as its transport and Protocol Buffers (Protobuf) as its default serialization format. Each RPC method maps to an HTTP/2 stream, and because HTTP/2 multiplexes streams over a single connection, many concurrent RPCs share one TCP connection efficiently.

The `.proto` file defines the service contract:

```protobuf
service OrderService {
  rpc GetOrder (GetOrderRequest) returns (Order);              // unary
  rpc StreamOrders (OrderFilter) returns (stream Order);        // server streaming
  rpc BatchCreate (stream CreateOrderReq) returns (BatchResult); // client streaming
  rpc LiveUpdates (stream Sub) returns (stream OrderEvent);     // bidirectional
}
```

Code generation from this file produces typed client stubs and server interfaces in any supported language — the contract is language-agnostic and version-controlled.

**Why Protobuf matters:**

- **Performance**: Binary encoding is 3-10x smaller than JSON and faster to serialize/deserialize. For high-throughput internal services (thousands of RPCs/second), this adds up.
- **Type safety**: The schema is explicit — field types, required vs optional, enums. No more "is this field a string or a number?" runtime surprises. Schema evolution rules (field numbers, backward compatibility) are built into the format.

**The four streaming modes:**

| Mode | Use case |
|------|----------|
| **Unary** | Standard request-response, like REST. Most common. |
| **Server streaming** | Client sends one request, server streams back many messages. Useful for: query results, real-time feeds, progress updates. |
| **Client streaming** | Client streams many messages, server responds once. Useful for: bulk uploads, aggregation. |
| **Bidirectional** | Both sides stream independently. Useful for: chat, collaborative editing, long-lived subscriptions. |

Streaming modes are a first-class feature — no WebSocket hack or SSE workaround needed.

**Real-world tradeoffs — gRPC vs REST:**

| | gRPC | REST (JSON over HTTP) |
|---|------|------|
| **Performance** | Binary, compact, fast | Text-based, larger payloads |
| **Type safety** | Schema-first, codegen | Schema optional (OpenAPI) |
| **Streaming** | Native 4 modes | Requires WebSocket/SSE |
| **Debugging** | Hard to inspect (binary on wire) | `curl`, browser, any HTTP tool |
| **Browser support** | Requires gRPC-Web proxy | Native |
| **Tooling maturity** | Growing but less ecosystem | Massive ecosystem |
| **Learning curve** | Protobuf + codegen + gRPC concepts | Widely understood |

**When to choose gRPC:**

- Internal service-to-service communication with high throughput requirements
- Polyglot microservices needing a shared contract
- When you need streaming natively
- When payload size and serialization speed matter (e.g., ML model serving, telemetry pipelines)

**When REST wins:**

- Public-facing APIs (browser-accessible, curl-debuggable)
- Simple CRUD services where the overhead of Protobuf tooling isn't justified
- Teams unfamiliar with gRPC where the learning curve slows delivery

</details>

<details>
<summary>10. When would you choose WebSockets vs Server-Sent Events vs HTTP long polling for a real-time feature -- explain how each approach works at a high level (WebSocket upgrade and full-duplex frames, SSE over standard HTTP, polling's simplicity), what the operational costs are for each (connection limits, load balancer stickiness, reconnection handling), and what factors drive the decision in practice?</summary>

**How each works:**

**WebSockets**: Client sends an HTTP/1.1 Upgrade request. If the server accepts, the connection switches from HTTP to a persistent, full-duplex WebSocket connection. Both sides can send frames (text or binary) at any time, independently. The connection stays open until explicitly closed.

**Server-Sent Events (SSE)**: Client opens a standard HTTP GET request with `Accept: text/event-stream`. The server holds the connection open and pushes events as they occur, each formatted as `data: ...\n\n`. It's one-directional (server → client) over plain HTTP. The browser's `EventSource` API handles automatic reconnection with `Last-Event-ID`.

**HTTP Long Polling**: Client sends a normal HTTP request. The server holds it open until there's new data (or a timeout), responds, and the client immediately sends another request. It simulates real-time by cycling through request-response pairs.

**Comparison:**

| | WebSocket | SSE | Long Polling |
|---|-----------|-----|-------------|
| **Direction** | Bidirectional | Server → client only | Server → client (client sends via separate requests) |
| **Protocol** | WS/WSS (not HTTP after upgrade) | Standard HTTP | Standard HTTP |
| **Reconnection** | Manual (application handles) | Built-in (`EventSource` auto-reconnects) | Built-in (next request = reconnect) |
| **Binary data** | Yes (binary frames) | No (text only, base64 for binary) | Yes (in response body) |
| **HTTP/2 compatible** | No (requires HTTP/1.1 upgrade) | Yes (multiplexed with other requests) | Yes |
| **Load balancer** | Needs sticky sessions or WS-aware LB | Standard HTTP — works with any LB | Standard HTTP |
| **Max connections** | No fixed per-origin cap — limited by browser memory and OS file descriptors | Browser limit ~6 per origin (HTTP/1.1) | Same as SSE |

**Operational costs:**

- **WebSockets**: Highest operational complexity. Load balancers must support WS upgrades. Connection state lives on a specific server, so you need sticky sessions or a pub/sub backend (Redis) to broadcast to the right connections. Health checks and graceful shutdown are more complex. Each connection consumes a file descriptor.

- **SSE**: Lower operational cost — it's just HTTP. Works behind standard load balancers and CDNs. Auto-reconnection is built into the browser API. The 6-connection limit per origin in HTTP/1.1 is the main constraint (solved with HTTP/2 multiplexing).

- **Long Polling**: Simplest to implement and deploy — no special protocol support needed. But it's the least efficient: each "event" requires a full HTTP request-response cycle, adding latency and overhead. Scales poorly for high-frequency updates.

**Decision factors:**

- **Need bidirectional real-time** (chat, collaborative editing, gaming)? → **WebSocket**
- **Server-to-client notifications only** (live scores, dashboards, deployment status)? → **SSE** — simpler to operate, auto-reconnect, works with HTTP/2
- **Low-frequency updates, simplicity over performance** (email inbox check, build status)? → **Long polling** or just regular polling with short intervals
- **Binary data streaming** (audio, video frames)? → **WebSocket**
- **Need to work through restrictive corporate proxies**? → **SSE** or **long polling** (some proxies block WebSocket upgrades)

In most backend notification scenarios (order updates, live dashboards, alerts), SSE is the right default — it's simpler to build, deploy, and operate than WebSockets, and the unidirectional constraint is rarely a problem since the client can use regular HTTP requests for the upstream direction.

</details>

<details>
<summary>11. What is the difference between an L4 and L7 load balancer/proxy, why does the layer matter — explain what each layer can inspect and route on, where TLS termination typically happens (and the tradeoffs of terminating at the proxy vs the backend), how X-Forwarded-For and X-Real-IP headers preserve client IP visibility through proxy chains, and what breaks when these headers are missing or misconfigured?</summary>

**L4 vs L7 — what each can see:**

| | L4 (Transport) | L7 (Application) |
|---|----------------|-------------------|
| **Sees** | IP addresses, ports, TCP/UDP headers | Full HTTP: URL path, headers, cookies, body |
| **Routes on** | IP:port tuples | URL path, Host header, cookie values, query params |
| **TLS** | Passes through (or terminates, but can't inspect HTTP) | Terminates TLS, inspects HTTP content |
| **Protocol awareness** | None — treats payload as opaque bytes | Understands HTTP, can rewrite headers, inject responses |
| **Performance** | Faster — less processing per packet | Slower — must parse application protocol |
| **Examples** | AWS NLB, HAProxy (TCP mode), iptables | AWS ALB, Nginx, Envoy, HAProxy (HTTP mode) |

**When L4 is enough**: Simple TCP load balancing, gRPC (where HTTP/2 multiplexing means one connection per backend anyway), non-HTTP protocols (database connections, custom TCP protocols).

**When you need L7**: Path-based routing (`/api` → service A, `/static` → CDN), header-based routing (A/B testing via cookie), rate limiting per API key, request/response transformation, WebSocket upgrade handling.

**TLS termination — where and why:**

- **Terminate at the proxy (most common)**: The proxy decrypts TLS, inspects HTTP, and forwards plain HTTP (or re-encrypts) to backends. This enables L7 routing, header injection, logging of request details, and centralized certificate management. The tradeoff: traffic between proxy and backend is unencrypted unless you add internal TLS ("mTLS" or re-encryption), and the proxy becomes a high-value attack target.

- **Terminate at the backend (passthrough)**: The L4 proxy forwards encrypted TCP bytes directly to the backend. The backend handles TLS. This gives end-to-end encryption but means the proxy can't inspect or route on HTTP content. Used when regulatory requirements demand end-to-end encryption or when the proxy is untrusted.

- **Re-encryption**: Proxy terminates external TLS, inspects HTTP, then opens a new TLS connection to the backend. Best of both worlds but adds latency from double TLS handshake. Common in zero-trust architectures.

**X-Forwarded-For and X-Real-IP:**

When a proxy terminates a connection, the backend sees the proxy's IP as the source, not the client's. These headers preserve client identity:

- **X-Forwarded-For**: A comma-separated list of IPs the request has passed through: `X-Forwarded-For: client, proxy1, proxy2`. Each proxy appends its predecessor's IP. The leftmost IP is the original client.

- **X-Real-IP**: Set by the first proxy to the original client IP. Simpler than XFF but only carries one address.

**What breaks when misconfigured:**

- **Missing headers**: Rate limiting, geo-blocking, and audit logs all use client IP. Without XFF, every request appears to come from the proxy's IP — rate limits apply globally instead of per-client, geo-blocking fails, and logs are useless for forensics.

- **Trusting untrusted XFF**: A client can spoof `X-Forwarded-For: 1.2.3.4` in their request. If your backend blindly trusts the leftmost IP, attackers can bypass IP-based rate limiting or access controls. The fix: configure your backend to only trust XFF entries added by known proxies (Nginx's `set_real_ip_from`, Express's `trust proxy` setting).

- **Missing X-Forwarded-Proto**: Without this header, the backend doesn't know the original request was HTTPS. It might generate HTTP redirect URLs or set cookies without the `Secure` flag.

</details>

<details>
<summary>12. How does HTTP caching work end-to-end — explain the Cache-Control directives (max-age, no-cache, no-store, public/private, s-maxage), how ETag and Last-Modified enable conditional requests (304 Not Modified), what the Vary header does and why it matters for CDNs, and how cache poisoning attacks exploit caching infrastructure?</summary>

**Cache-Control directives:**

| Directive | Meaning |
|-----------|---------|
| `max-age=N` | Response is fresh for N seconds from the time of the request. Browser/proxy can serve from cache without revalidation. |
| `s-maxage=N` | Like `max-age` but only applies to shared caches (CDNs, reverse proxies). Overrides `max-age` for shared caches. |
| `no-cache` | **Does NOT mean "don't cache."** It means "cache it, but always revalidate with the server before using it." The cache stores the response but must check freshness via conditional request every time. |
| `no-store` | Actually means don't cache — don't store the response anywhere. For sensitive data (bank balances, PII). |
| `public` | Any cache (browser, CDN, proxy) can store this. Required for caching responses to authenticated requests in shared caches. |
| `private` | Only the browser cache can store this. Shared caches (CDN) must not. Use for user-specific data. |
| `must-revalidate` | Once stale, the cache must revalidate — it cannot serve stale content even if the server is unreachable. |
| `immutable` | Content will never change. Browser won't even make conditional requests on reload. Perfect for content-hashed static assets. |

**Conditional requests (ETag and Last-Modified):**

When a cached response expires (or uses `no-cache`), the client sends a conditional request to check if the resource changed:

1. Server sends initial response with `ETag: "abc123"` (a hash/fingerprint of the content) or `Last-Modified: Wed, 01 Jan 2025 00:00:00 GMT`
2. On revalidation, client sends `If-None-Match: "abc123"` or `If-Modified-Since: Wed, 01 Jan 2025 00:00:00 GMT`
3. If unchanged, server responds **304 Not Modified** (no body) — the cache serves its stored copy
4. If changed, server responds **200** with the new content

This saves bandwidth — a 304 is tiny compared to re-downloading the full resource. ETag is more precise than Last-Modified (which has 1-second granularity).

**The Vary header:**

`Vary` tells caches that the response differs based on specific request headers. For example:

- `Vary: Accept-Encoding` — the response body differs for `gzip` vs `br` vs no compression. The cache stores separate entries for each.
- `Vary: Accept-Language` — different language versions cached separately.
- `Vary: Cookie` — effectively makes the response uncacheable by shared caches (every user has different cookies).

**Why it matters for CDNs:** Without proper `Vary`, a CDN might serve a gzipped response to a client that doesn't support gzip, or serve English content to a French user. With `Vary: Accept-Encoding`, the CDN maintains separate cached versions per encoding.

**Cache poisoning attacks:**

Cache poisoning exploits the gap between what a cache uses as its key and what the server uses to generate the response:

1. **Unkeyed headers**: The cache keys on URL + Host, but the server also uses a header like `X-Forwarded-Host` to generate links in the response. Attacker sends a request with `X-Forwarded-Host: evil.com`. The server generates a page with links pointing to `evil.com`. The CDN caches this response. All subsequent users requesting that URL get the poisoned response.

2. **Unkeyed query parameters**: Some frameworks ignore certain query parameters, but the cache includes them in the key — or vice versa. Attackers find parameters the cache ignores but the server processes.

**Defenses**: Strip unexpected headers at the proxy/CDN level, use `Vary` correctly, define explicit cache keys that include all headers the server uses, and audit what the server reflects into responses.

</details>

<details>
<summary>13. Why does CORS exist and how does the preflight mechanism actually work — explain what the browser enforces vs what the server controls, how OPTIONS preflight requests work, why credentialed requests have stricter rules (no wildcard origins), and what the common CORS misconfigurations are that either break legitimate requests or create security vulnerabilities?</summary>

**Why CORS exists:**

Browsers enforce the **Same-Origin Policy**: JavaScript on `app.example.com` cannot make requests to `api.other.com` by default. This prevents malicious sites from making authenticated requests to your bank's API using your cookies. CORS (Cross-Origin Resource Sharing) is the controlled escape hatch — the server explicitly declares which origins are allowed to access its resources.

**What the browser enforces vs what the server controls:**

- **Browser**: Blocks the JavaScript code from reading the response if CORS headers are missing or don't match. The request may still be sent (for simple requests) — CORS doesn't prevent the request, it prevents the response from reaching the JS.
- **Server**: Responds with `Access-Control-Allow-Origin` and related headers to declare what's permitted. The server doesn't enforce CORS — it just provides the policy. The browser enforces it.

**How preflight works:**

For "non-simple" requests (custom headers, PUT/DELETE methods, `Content-Type: application/json`), the browser sends an **OPTIONS** request before the actual request:

```
OPTIONS /api/users HTTP/1.1
Host: api.example.com
Origin: https://app.example.com
Access-Control-Request-Method: PUT
Access-Control-Request-Headers: Content-Type, Authorization
```

The server responds with what's allowed:

```
HTTP/1.1 204 No Content
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Methods: GET, PUT, DELETE
Access-Control-Allow-Headers: Content-Type, Authorization
Access-Control-Max-Age: 86400
```

If the preflight passes, the browser sends the actual PUT request. `Access-Control-Max-Age` caches the preflight result so subsequent requests skip the OPTIONS call.

**Simple requests** (GET/POST with standard headers and form content types) skip preflight — the browser sends the request directly and checks the response headers. This preserves backward compatibility with how forms already worked pre-CORS.

**Credentialed requests — stricter rules:**

When requests include cookies or Authorization headers (`credentials: 'include'` in fetch):

- `Access-Control-Allow-Origin` **cannot be `*`** — it must be an explicit origin. This prevents a malicious site from using your cookies to access any API.
- `Access-Control-Allow-Credentials: true` must be set.
- `Access-Control-Expose-Headers` cannot use `*` either.

**Common misconfigurations:**

- **`Access-Control-Allow-Origin: *` with credentials**: Doesn't work — the browser rejects it. Developers then "fix" it by reflecting the Origin header back without validation, which is worse (see below).

- **Reflecting Origin without validation**: The server sets `Access-Control-Allow-Origin` to whatever the `Origin` header says. This effectively disables CORS — any site can make credentialed requests. This is a security vulnerability equivalent to having no same-origin policy.

- **Missing `Vary: Origin`**: If the server responds with different CORS headers depending on the Origin, caches must know to vary on it. Without `Vary: Origin`, a CDN might cache the CORS headers for one origin and serve them to another, breaking or allowing unintended access.

- **Forgetting OPTIONS handler**: The server returns 404 or 405 for OPTIONS requests, breaking all preflighted requests. Many frameworks handle this automatically, but custom routing or API gateways may not.

- **Allowing `null` origin**: Some developers allow `Origin: null` (sent by sandboxed iframes, local files). This lets attackers craft requests from sandboxed contexts that pass CORS checks.

</details>

## Practical — Implementation & Configuration

<details>
<summary>14. Configure CORS middleware in a Node.js/Express API that handles both simple and preflighted requests — show the middleware setup for allowing specific origins, handling credentialed requests, exposing custom headers, and explain what breaks when you use a wildcard origin with credentials and how you'd support multiple allowed origins dynamically?</summary>

**Using the `cors` package (recommended approach):**

```typescript
import express from "express";
import cors from "cors";

const app = express();

// Static list of allowed origins
const allowedOrigins = [
  "https://app.example.com",
  "https://staging.example.com",
];

app.use(
  cors({
    // Dynamic origin check — called for every request
    origin: (origin, callback) => {
      // Allow requests with no origin (server-to-server, curl)
      if (!origin || allowedOrigins.includes(origin)) {
        callback(null, origin); // reflect the specific origin back
      } else {
        callback(new Error("Not allowed by CORS"));
      }
    },
    credentials: true, // Allow cookies/auth headers
    methods: ["GET", "POST", "PUT", "DELETE", "PATCH"],
    allowedHeaders: ["Content-Type", "Authorization", "X-Request-ID"],
    exposedHeaders: ["X-Total-Count", "X-Request-ID"], // Headers the browser JS can read
    maxAge: 86400, // Cache preflight for 24 hours
  })
);
```

**What breaks with wildcard + credentials:**

If you set `origin: '*'` with `credentials: true`, the browser rejects the response. The CORS spec explicitly forbids `Access-Control-Allow-Origin: *` on credentialed requests because it would mean any website could make authenticated requests using the user's cookies. The fix is always reflecting the specific, validated origin as shown above.

**Dynamic origin support for multiple allowed origins:**

The `origin` callback pattern above is the correct approach. The key detail: the `cors` middleware sets `Vary: Origin` automatically when using a function, which tells CDNs and caches to store separate responses per origin. If you're writing CORS manually, forgetting `Vary: Origin` causes cache poisoning — a CDN caches the response with origin A's CORS headers and serves it to origin B.

**Manual implementation (without the cors package):**

```typescript
app.use((req, res, next) => {
  const origin = req.headers.origin;

  if (origin && allowedOrigins.includes(origin)) {
    res.setHeader("Access-Control-Allow-Origin", origin);
    res.setHeader("Access-Control-Allow-Credentials", "true");
    res.setHeader("Vary", "Origin"); // Critical for caching correctness
  }

  // Handle preflight
  if (req.method === "OPTIONS") {
    res.setHeader("Access-Control-Allow-Methods", "GET,POST,PUT,DELETE,PATCH");
    res.setHeader(
      "Access-Control-Allow-Headers",
      "Content-Type,Authorization,X-Request-ID"
    );
    res.setHeader("Access-Control-Max-Age", "86400");
    return res.status(204).end();
  }

  next();
});
```

**Gotchas:**

- CORS middleware must run before route handlers — if a route responds before CORS headers are set, the browser blocks it.
- `exposedHeaders` is often forgotten. By default, the browser only exposes a few "safe" response headers to JS (Cache-Control, Content-Language, Content-Type, etc.). Custom headers like `X-Total-Count` for pagination are invisible to frontend code unless explicitly exposed.
- The `OPTIONS` response must return `204` (or `200`) with no body. Some frameworks accidentally route `OPTIONS` to the actual handler, which may require auth and fail.

</details>

<details>
<summary>15. Implement a Server-Sent Events endpoint in Node.js and a basic WebSocket server for the same use case (e.g., live notifications) — show both implementations side by side, explain the key differences in setup complexity, connection handling, and how the client reconnects after disconnection in each approach?</summary>

**SSE Implementation:**

```typescript
import express from "express";

const app = express();

// Track connected clients
const sseClients = new Set<express.Response>();

app.get("/notifications", (req, res) => {
  // SSE requires these specific headers
  res.setHeader("Content-Type", "text/event-stream");
  res.setHeader("Cache-Control", "no-cache");
  res.setHeader("Connection", "keep-alive");

  // Send initial connection event
  res.write("data: connected\n\n");

  sseClients.add(res);

  // Client disconnects
  req.on("close", () => {
    sseClients.delete(res);
  });
});

// Broadcast to all connected SSE clients
function broadcastSSE(event: string, data: unknown) {
  const message = `event: ${event}\ndata: ${JSON.stringify(data)}\nid: ${Date.now()}\n\n`;
  for (const client of sseClients) {
    client.write(message);
  }
}

app.listen(3000);
```

**SSE Client (browser):**

```typescript
const source = new EventSource("/notifications");

source.addEventListener("notification", (e) => {
  const data = JSON.parse(e.data);
  console.log("Notification:", data);
});

// Auto-reconnects on disconnect, sends Last-Event-ID header
source.onerror = () => console.log("SSE reconnecting...");
```

**WebSocket Implementation:**

```typescript
import { createServer } from "http";
import { WebSocketServer, WebSocket } from "ws";
import express from "express";

const app = express();
const server = createServer(app);
const wss = new WebSocketServer({ server });

const wsClients = new Set<WebSocket>();

wss.on("connection", (ws) => {
  wsClients.add(ws);
  ws.on("error", console.error);

  ws.send(JSON.stringify({ type: "connected" }));

  // WebSocket is bidirectional — can receive messages from client
  ws.on("message", (raw) => {
    const msg = JSON.parse(raw.toString());
    console.log("Client says:", msg);
  });

  ws.on("close", () => {
    wsClients.delete(ws);
  });

  // Heartbeat to detect dead connections
  // Custom property — ws types don't include isAlive
  ws.on("pong", () => {
    (ws as any).isAlive = true;
  });
  (ws as any).isAlive = true;
});

// Ping clients every 30s to detect broken connections
const interval = setInterval(() => {
  for (const ws of wsClients) {
    if (!(ws as any).isAlive) {
      ws.terminate();
      wsClients.delete(ws);
      return;
    }
    (ws as any).isAlive = false;
    ws.ping();
  }
}, 30_000);

wss.on("close", () => clearInterval(interval));

function broadcastWS(event: string, data: unknown) {
  const message = JSON.stringify({ event, data, id: Date.now() });
  for (const ws of wsClients) {
    if (ws.readyState === WebSocket.OPEN) {
      ws.send(message);
    }
  }
}

server.listen(3000);
```

**WebSocket Client (browser):**

```typescript
function connect() {
  const ws = new WebSocket("wss://api.example.com/notifications");

  ws.onmessage = (e) => {
    const { event, data } = JSON.parse(e.data);
    console.log("Notification:", event, data);
  };

  // Manual reconnection — no built-in auto-reconnect
  ws.onclose = () => {
    console.log("WS closed, reconnecting in 3s...");
    setTimeout(connect, 3000);
  };
}
connect();
```

**Key differences** (for a detailed feature comparison, see Q10):

| | SSE | WebSocket |
|---|-----|-----------|
| **Setup complexity** | Just Express route + response headers | Separate WebSocketServer, HTTP upgrade handling |
| **Protocol** | Standard HTTP — works with existing middleware, auth, CORS | Separate WS protocol — auth must happen during upgrade or first message |
| **Direction** | Server → client only | Bidirectional |
| **Reconnection** | Automatic via `EventSource` API, resumes with `Last-Event-ID` | Manual — you write the reconnect logic with backoff |
| **Dead connection detection** | HTTP keep-alive / TCP handles it | Must implement ping/pong heartbeat yourself |
| **Load balancer** | Standard HTTP — no special config | Must support WS upgrade; may need sticky sessions |
| **Data format** | Text only (UTF-8) | Text or binary frames |

**Reconnection handling is the biggest practical difference.** SSE's `EventSource` automatically reconnects and sends the `Last-Event-ID` header, letting the server replay missed events. With WebSockets, you build all of this yourself: reconnect with exponential backoff, track what the client last received, and replay from that point.

For the notification use case, SSE is clearly the better fit — it's simpler, reconnection is built in, and notifications are inherently server-to-client.

</details>

<details>
<summary>16. Set appropriate Cache-Control headers in a Node.js API for three scenarios: static assets (JS/CSS with content hashes), public API responses that change infrequently, and user-specific data that must not be cached by shared caches — show the middleware configuration and explain why each directive combination is correct for its scenario?</summary>

```typescript
import express from "express";

const app = express();

// ──────────────────────────────────────────────
// Scenario 1: Static assets with content hashes
// Files like /assets/app.a1b2c3.js — the hash changes when content changes
// ──────────────────────────────────────────────
app.use(
  "/assets",
  express.static("dist/assets", {
    setHeaders: (res) => {
      res.setHeader(
        "Cache-Control",
        "public, max-age=31536000, immutable"
      );
    },
  })
);
// Why: Content-hashed files NEVER change at the same URL — if content changes,
// the hash changes and it's a different URL. So we cache aggressively:
// - public: CDNs and browsers can cache
// - max-age=31536000: 1 year (effectively forever)
// - immutable: browser won't even revalidate on page reload

// ──────────────────────────────────────────────
// Scenario 2: Public API data that changes infrequently
// e.g., product catalog, country list, configuration
// ──────────────────────────────────────────────
app.get("/api/products", (req, res) => {
  const products = getProducts();
  const etag = computeETag(products); // hash of the response body

  res.setHeader("Cache-Control", "public, max-age=60, s-maxage=300, stale-while-revalidate=60");
  res.setHeader("ETag", etag);

  // Handle conditional request
  if (req.headers["if-none-match"] === etag) {
    return res.status(304).end();
  }

  res.json(products);
});
// Why:
// - public: safe for CDN caching (no user-specific data)
// - max-age=60: browser uses cache for 60s before revalidating
// - s-maxage=300: CDN caches for 5 min (longer than browser — CDN serves many users)
// - stale-while-revalidate=60: CDN can serve stale content for 60s while fetching
//   fresh content in the background — users never wait for revalidation
// - ETag: enables 304 responses when content hasn't changed, saving bandwidth

// ──────────────────────────────────────────────
// Scenario 3: User-specific data
// e.g., user profile, account balance, personal settings
// ──────────────────────────────────────────────
app.get("/api/me", authMiddleware, (req, res) => {
  const profile = getUserProfile(req.user.id);

  res.setHeader("Cache-Control", "private, no-cache, max-age=0");
  res.json(profile);
});
// Why:
// - private: only the user's browser can cache — CDN/shared caches must NOT store
// - no-cache: browser must revalidate with server before every use
// - max-age=0: response is immediately stale
// Together: the browser can store it (avoiding re-download if unchanged with ETag)
// but must check with the server every time. CDNs never see it.
//
// For truly sensitive data (bank balance, passwords), use:
// Cache-Control: no-store
// This prevents the browser from storing the response at all.
```

**Common mistakes:**

- Using `no-cache` when you mean `no-store`. `no-cache` still caches, it just always revalidates. `no-store` truly prevents caching.
- Forgetting `s-maxage` for CDN-cached API responses — without it, the CDN uses `max-age`, which might be too short or too long for a shared cache.
- Not setting `private` on authenticated responses — without it, a CDN might cache user A's profile and serve it to user B.
- Setting aggressive cache headers on HTML pages — if the HTML is cached, users can't get the new hashed asset URLs. HTML should use `no-cache` or short `max-age`.

</details>

---

## Experience-Based Questions

These questions test real-world experience. Prepare by mapping them to your own projects and situations.

<details>
<summary>17. Tell me about a time you debugged a networking issue in production — what were the symptoms, what tools did you use to isolate the problem, and what turned out to be the root cause?</summary>

**What the interviewer is looking for:**

- Systematic debugging approach (not guessing)
- Knowledge of networking concepts applied to a real situation
- Familiarity with debugging tools (not just "I looked at logs")
- Clear thinking under pressure — how you narrowed down the problem space

**Suggested structure (STAR-like):**

1. **Situation**: What was the system, what was the symptom? (Timeouts? Connection refused? Intermittent failures? Elevated latency?)
2. **Layered investigation**: How did you isolate which layer was failing? (DNS? TCP? TLS? Application?)
3. **Tools used**: Name specific tools and what each told you
4. **Root cause**: What was actually wrong?
5. **Resolution and prevention**: How did you fix it and prevent recurrence?

**Example outline (personalize with your own experience):**

> "We saw intermittent 502 errors on our API gateway. About 5% of requests were failing, and it wasn't tied to load.
>
> First, I checked application logs — the gateway was reporting 'connection reset by peer' from the upstream service. That told me TCP connections were being closed unexpectedly, so it wasn't an application-level bug.
>
> I checked the upstream service's health — it was running fine, no crashes or restarts. Then I looked at the load balancer (AWS ALB) metrics and noticed the 502s correlated with connections that had been idle for exactly 60 seconds.
>
> That pointed to a keepalive mismatch. The ALB's idle timeout was 60 seconds, but our Node.js server's `server.keepAliveTimeout` was set to the default 5 seconds. The ALB thought the connection was alive, but the backend had already closed it. The ALB sent the next request on the dead connection and got a reset.
>
> Fix: Set `server.keepAliveTimeout` to 65 seconds (higher than the ALB's 60s). 502s dropped to zero.
>
> Prevention: Added a monitoring alert on 502 rates and documented the keepalive configuration requirement in our service template."

**Tools to mention (pick what's relevant to your story):**

- `curl -v` / `wget`: See the full request-response including TLS handshake
- `dig` / `nslookup`: DNS resolution debugging
- `tcpdump` / Wireshark: Packet-level inspection
- `openssl s_client`: TLS handshake debugging
- `netstat` / `ss`: Connection state inspection (TIME_WAIT accumulation, etc.)
- Cloud provider metrics: ALB/NLB connection counts, error rates, latency percentiles
- Application logs with correlation IDs

</details>

<details>
<summary>18. Tell me about a time you had to choose between WebSockets, SSE, and HTTP polling for a real-time feature — what were the requirements, what tradeoffs did you evaluate, and would you make the same choice again?</summary>

**What the interviewer is looking for:**

- Structured decision-making — not just "we picked WebSockets because they're cool"
- Understanding of the tradeoffs covered in question 10 applied to a real situation
- Awareness of operational cost, not just developer experience
- Ability to reflect on whether the choice held up over time

**Suggested structure:**

1. **Context**: What was the feature? What did "real-time" mean — sub-second? Every few seconds? How many concurrent users?
2. **Requirements analysis**: Unidirectional or bidirectional? Binary data? How critical is message delivery? What infrastructure exists?
3. **Options evaluated**: Which approaches did you consider and what tradeoffs stood out for your specific case?
4. **Decision and rationale**: What did you pick and why? What was the deciding factor?
5. **Outcome and reflection**: Did it work well? What would you change?

**Example outline (personalize with your own experience):**

> "We needed live status updates for long-running data import jobs — users would upload a CSV, and we wanted to show real-time progress (percentage, row count, errors) in the dashboard.
>
> Requirements: Server-to-client only (the client never sends data after the initial request). A few hundred concurrent users max. Updates every 1-2 seconds during an import. Standard infrastructure — Nginx reverse proxy, Kubernetes pods.
>
> I evaluated all three:
> - **Polling**: Simplest, but importing could take 30 minutes. Polling every 2 seconds for 30 minutes is thousands of wasted requests.
> - **WebSockets**: Full-duplex, but we didn't need bidirectional communication. It would've required sticky sessions or a Redis pub/sub layer since our pods scale horizontally, plus we'd need to handle reconnection logic ourselves.
> - **SSE**: Unidirectional (perfect for our use case), works over standard HTTP so no Nginx upgrade config needed, `EventSource` API handles reconnection automatically with `Last-Event-ID`.
>
> We went with SSE. The deciding factor was operational simplicity — it worked with our existing infrastructure with zero config changes, and the auto-reconnection meant we didn't have to build retry logic.
>
> It worked well. The one thing I'd reconsider: if we ever need to add client-to-server messaging (like canceling an import mid-stream), we'd either add a separate REST endpoint for that or consider switching to WebSockets. But for unidirectional updates, SSE was the right call."

**Key points to hit regardless of your specific story:**

- Show you considered at least two options seriously
- Name the specific tradeoff that tipped the decision (operational cost, bidirectional need, infrastructure constraints, team familiarity)
- Be honest about limitations of the choice — interviewers value engineers who acknowledge tradeoffs rather than claiming their choice was perfect

</details>

