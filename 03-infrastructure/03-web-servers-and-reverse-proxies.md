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

<!-- Answer will be added later -->

</details>

<details>
<summary>2. Walk through the full lifecycle of an HTTPS request from a client browser through Nginx to a Node.js backend and back — what happens at each stage (DNS resolution, TLS handshake, header manipulation, buffering, upstream selection, response), and why does putting Nginx in front of Node.js improve the overall architecture?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>3. How does Nginx's architecture — master process, worker processes, and event-driven I/O with epoll — allow it to handle tens of thousands of concurrent connections on a single machine, and why is this fundamentally different from a thread-per-connection model like traditional Apache?</summary>

<!-- Answer will be added later -->

</details>

## Conceptual Depth

<details>
<summary>4. What are the main load balancing algorithms Nginx supports (round-robin, least connections, IP hash, generic hash), what tradeoffs does each make between simplicity, fairness, and session affinity — and what failure modes can each algorithm introduce when upstream servers go down or respond slowly?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>5. Why is TLS termination typically done at the reverse proxy rather than at each application server — what are the operational and performance benefits, when would you use TLS passthrough instead, when would you re-encrypt traffic between the proxy and backend, and how do compliance requirements (PCI-DSS, HIPAA) influence this decision?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>6. What are the main rate limiting algorithms (leaky bucket, token bucket, sliding window), how do they differ in behavior and fairness — and when should rate limiting be done at the proxy layer vs the application layer, considering that both approaches have different strengths and failure modes?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>7. Why is proxying WebSocket and SSE connections different from regular HTTP — how does the connection upgrade mechanism work through a reverse proxy, what proxy timeout and buffering settings break long-lived connections if misconfigured, and how does Nginx's buffering behavior differ between SSE streams and WebSocket frames?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>8. What do the proxy headers X-Forwarded-For, X-Real-IP, and Host convey, why are they necessary when a reverse proxy sits between clients and backends — and what security vulnerabilities arise from trusting these headers without proper configuration (trust-proxy chains, IP spoofing)?</summary>

<!-- Answer will be added later -->

</details><details>
<summary>9. Why would you serve multiple backend services from a single domain using path-based routing at the proxy layer — what are the alternatives (subdomains, separate domains), what are the tradeoffs, and how does Nginx's location block matching and priority system (exact, prefix, regex) determine which block handles a given request?</summary>

<!-- Answer will be added later -->

</details>

## Practical — Configuration & Routing

<details>
<summary>10. Configure Nginx as a reverse proxy for two Node.js services running on different ports, routing requests based on URL path (/api/* to one service, /admin/* to another, and everything else to a static frontend) — show the full nginx.conf with upstream blocks, location blocks, and proxy_pass directives, and explain what happens when the location block matching order is wrong.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>11. Configure Nginx to load balance across three backend instances using least-connections with health checks — show the upstream block configuration, explain how max_fails and fail_timeout work, what happens when all upstreams are marked unhealthy, and how you'd add a new backend instance without restarting Nginx.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>12. Configure TLS termination in Nginx with certificate and key files, redirect all HTTP traffic to HTTPS, set up HSTS headers, and configure OCSP stapling — show the server blocks for both port 80 and 443, explain the ssl_protocols and ssl_ciphers choices, and what breaks if the certificate chain is incomplete.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>13. Configure rate limiting in Nginx using the limit_req module — show how to set up a shared memory zone, define the rate, configure burst and nodelay, apply different rate limits to different location blocks (e.g., stricter limits on /api/login), and explain what the client experiences when they hit the limit vs when they exceed the burst.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>14. Configure Nginx to proxy WebSocket connections and SSE streams to a Node.js backend — show the configuration for the connection upgrade mechanism (Upgrade and Connection headers), explain what proxy_read_timeout and proxy_send_timeout values you'd set for long-lived connections, why proxy_buffering must be off for SSE, and what symptoms appear when these settings are wrong.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>15. Configure the proxy headers (X-Forwarded-For, X-Real-IP, X-Forwarded-Proto, Host) in Nginx for a setup where Nginx sits behind a cloud load balancer — show the proxy_set_header directives, explain how set_header vs add_header behaves differently in nested location blocks, and demonstrate the configuration that prevents IP spoofing through X-Forwarded-For header injection.</summary>

<!-- Answer will be added later -->

</details>## Practical — Observability & Debugging

<details>
<summary>16. Your application is intermittently returning 502 Bad Gateway through Nginx — walk through the exact steps to diagnose the root cause: what to check in Nginx error logs, how to determine whether the upstream is crashing vs refusing connections vs timing out during response, what the difference between "connection refused" and "no live upstreams" means, and how to fix each case.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>17. Users report that certain API requests hang for exactly 60 seconds and then return 504 Gateway Timeout — walk through the debugging process: how to identify which timeout is being hit (proxy_read_timeout, proxy_connect_timeout, or upstream application timeout), how to check whether the backend is actually slow vs Nginx is misconfigured, and what the correct timeout tuning strategy looks like for different types of endpoints.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>18. After a deployment, some requests are failing because Nginx is still sending traffic to old backend IPs — explain why this happens (DNS caching in Nginx with upstream blocks), how Nginx resolves DNS for upstream servers at startup vs at runtime, and show the configuration changes (resolver directive, variables in proxy_pass) that fix stale upstream IP issues in dynamic environments like Kubernetes or auto-scaling groups.</summary>

<!-- Answer will be added later -->

</details>---

## Experience-Based Questions

These questions test real-world experience. Prepare by mapping them to your own projects and situations.

<details>
<summary>19. Tell me about a time you configured or optimized an Nginx reverse proxy setup in production — what was the architecture, what problems were you solving, and what configuration decisions had the biggest impact on performance or reliability?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>20. Describe a time you debugged a proxy-related issue in production (502s, 504s, TLS errors, routing problems) — what were the symptoms, how did you narrow down whether the issue was in the proxy layer vs the backend, and what was the root cause?</summary>

<!-- Answer framework will be added later -->

</details>

