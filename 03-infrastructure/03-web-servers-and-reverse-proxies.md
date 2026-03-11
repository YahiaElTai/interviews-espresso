# Web Servers & Reverse Proxies

> **35 questions** — 15 theory, 20 practical

- Forward proxy vs reverse proxy: purpose, architecture, and use cases
- Request lifecycle through Nginx to a Node.js backend (TLS, headers, buffering, upstream selection)
- Nginx architecture: worker processes, event-driven model (epoll), why it handles high concurrency efficiently
- Nginx config structure and operations: http/server/location context hierarchy, nginx -t validation, graceful reload vs restart
- Load balancing algorithms: round-robin, least connections, IP hash, generic hash — tradeoffs and failure modes
- TLS termination at the proxy: certificate management, TLS passthrough, re-encryption, compliance
- Self-managed Nginx vs cloud load balancers (GCP HTTP(S) LB, AWS ALB/NLB): when to use each, what you gain and lose, cost and operational tradeoffs
- Proxy-layer caching: cache keys, Cache-Control interaction, cache stampede, stale data risks
- Rate limiting: leaky bucket, token bucket, sliding window — proxy-layer vs application-layer
- HTTP/2 and gRPC proxying: protocol negotiation, downstream HTTP/2 with upstream HTTP/1.1, why gRPC requires HTTP/2 end-to-end
- WebSocket and SSE proxying: connection upgrade handling, proxy timeouts for long-lived connections, buffering behavior
- Connection draining, upstream keepalive tuning, and zero-downtime rolling deployments
- Request size limits and buffering: client_max_body_size, proxy buffering on/off, proxy_buffer_size — common misconfigurations with file uploads and streaming responses
- Static file serving, location block hierarchy, try_files, cache-busting, gzip/Brotli compression
- Proxy headers: X-Forwarded-For, X-Real-IP, Host — and trust-proxy security risks
- Security headers: HSTS, CSP, X-Frame-Options, X-Content-Type-Options, server version hiding
- Proxy-layer observability: structured access logs, request IDs for distributed tracing, latency and error rate metrics, stub_status
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
<summary>6. When should you use a self-managed Nginx reverse proxy vs a cloud-managed load balancer (GCP HTTP(S) LB, AWS ALB/NLB) — what capabilities does each provide, what do you gain and lose with each approach, and how does the distinction between L4 and L7 load balancing affect which one you choose?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>7. How does proxy-layer caching work in Nginx — what determines the cache key, how does the proxy interact with Cache-Control headers from the backend, what is a cache stampede (thundering herd) and how do you prevent it, and what are the risks of serving stale data from the proxy cache?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>8. What are the main rate limiting algorithms (leaky bucket, token bucket, sliding window), how do they differ in behavior and fairness — and when should rate limiting be done at the proxy layer vs the application layer, considering that both approaches have different strengths and failure modes?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>9. Why is proxying WebSocket and SSE connections different from regular HTTP — how does the connection upgrade mechanism work through a reverse proxy, what proxy timeout and buffering settings break long-lived connections if misconfigured, and how does Nginx's buffering behavior differ between SSE streams and WebSocket frames?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>10. Why does proxying HTTP/2 and gRPC traffic through Nginx require different configuration than regular HTTP/1.1 — how does protocol negotiation work between client and proxy, why is it common to terminate HTTP/2 at the proxy and use HTTP/1.1 upstream, why does gRPC require HTTP/2 end-to-end (and what breaks if any hop downgrades to HTTP/1.1), and how do you configure Nginx to handle both?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>11. Why are connection draining and upstream keepalive tuning critical for zero-downtime rolling deployments — what happens to in-flight requests when an upstream server is removed without draining, how do keepalive connections between Nginx and backends reduce latency, and what goes wrong when keepalive settings are mismatched between the proxy and the application?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>12. What do the proxy headers X-Forwarded-For, X-Real-IP, and Host convey, why are they necessary when a reverse proxy sits between clients and backends — and what security vulnerabilities arise from trusting these headers without proper configuration (trust-proxy chains, IP spoofing)?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>13. What are the key security response headers (Content-Security-Policy, X-Frame-Options, X-Content-Type-Options, Strict-Transport-Security) and why should they be set at the proxy layer — what attacks does each header prevent, and why is hiding the server version (Server header) a worthwhile security measure?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>14. Why would you serve multiple backend services from a single domain using path-based routing at the proxy layer — what are the alternatives (subdomains, separate domains), what are the tradeoffs, and how does Nginx's location block matching and priority system (exact, prefix, regex) determine which block handles a given request?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>15. Why is proxy-layer observability important even when your application already has its own logging and metrics — what do structured access logs, request ID propagation for distributed tracing, latency/error rate metrics, and Nginx's stub_status module each provide that application-level instrumentation cannot?</summary>

<!-- Answer will be added later -->

</details>

## Practical — Configuration & Routing

<details>
<summary>16. Configure Nginx as a reverse proxy for two Node.js services running on different ports, routing requests based on URL path (/api/* to one service, /admin/* to another, and everything else to a static frontend) — show the full nginx.conf with upstream blocks, location blocks, and proxy_pass directives, and explain what happens when the location block matching order is wrong.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>17. Configure Nginx to load balance across three backend instances using least-connections with health checks — show the upstream block configuration, explain how max_fails and fail_timeout work, what happens when all upstreams are marked unhealthy, and how you'd add a new backend instance without restarting Nginx.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>18. Configure TLS termination in Nginx with certificate and key files, redirect all HTTP traffic to HTTPS, set up HSTS headers, and configure OCSP stapling — show the server blocks for both port 80 and 443, explain the ssl_protocols and ssl_ciphers choices, and what breaks if the certificate chain is incomplete.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>19. Configure Nginx to serve a React SPA's static files with proper cache-busting headers for hashed assets (long Cache-Control max-age) vs the index.html (no-cache), set up try_files for client-side routing fallback, and enable gzip and Brotli compression — show the full location blocks and explain why getting the caching strategy wrong for index.html causes users to see stale versions of the app.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>20. Configure Nginx to handle large file uploads (up to 100MB) to a Node.js backend and stream responses for a download endpoint — show the directives for client_max_body_size, proxy_buffering, proxy_buffer_size, and proxy_max_temp_file_size, explain what symptoms users see when client_max_body_size is too low (413 error), what happens when proxy buffering is enabled for a streaming response endpoint, and why the default proxy_buffer_size causes issues with backends that return large headers.</summary>

<!-- Answer will be added later -->

</details>

## Practical — Advanced Proxy Patterns

<details>
<summary>21. Configure Nginx proxy caching for an API endpoint — show the proxy_cache_path directive, cache zone configuration, cache key customization, how to set different cache durations per response code, how to bypass the cache for authenticated requests, and how to use proxy_cache_use_stale to serve stale content during backend failures.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>22. Configure rate limiting in Nginx using the limit_req module — show how to set up a shared memory zone, define the rate, configure burst and nodelay, apply different rate limits to different location blocks (e.g., stricter limits on /api/login), and explain what the client experiences when they hit the limit vs when they exceed the burst.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>23. Configure Nginx to proxy WebSocket connections and SSE streams to a Node.js backend — show the configuration for the connection upgrade mechanism (Upgrade and Connection headers), explain what proxy_read_timeout and proxy_send_timeout values you'd set for long-lived connections, why proxy_buffering must be off for SSE, and what symptoms appear when these settings are wrong.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>24. Configure Nginx for zero-downtime rolling deployments — show the upstream keepalive settings (keepalive, keepalive_timeout, keepalive_requests), explain how to use the proxy_next_upstream directive to retry failed requests on a different backend, and demonstrate how connection draining works when removing a backend from the upstream block during a deploy.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>25. Configure the proxy headers (X-Forwarded-For, X-Real-IP, X-Forwarded-Proto, Host) in Nginx for a setup where Nginx sits behind a cloud load balancer — show the proxy_set_header directives, explain how set_header vs add_header behaves differently in nested location blocks, and demonstrate the configuration that prevents IP spoofing through X-Forwarded-For header injection.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>26. Configure security response headers in Nginx (Content-Security-Policy, X-Frame-Options, X-Content-Type-Options, Strict-Transport-Security) and hide the server version — show the add_header directives, explain why these are typically set at the proxy layer rather than in application code, and what happens when add_header in a nested location block silently drops headers set in a parent block.</summary>

<!-- Answer will be added later -->

</details>

## Practical — Observability & Debugging

<details>
<summary>27. Configure Nginx structured access logs in JSON format that include request duration, upstream response time, status code, request ID, and client IP — show the log_format directive, explain how to generate and propagate a unique request ID ($request_id) through to the backend for distributed tracing, and set up the stub_status endpoint for monitoring tools to scrape basic connection metrics.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>28. Your application is intermittently returning 502 Bad Gateway through Nginx — walk through the exact steps to diagnose the root cause: what to check in Nginx error logs, how to determine whether the upstream is crashing vs refusing connections vs timing out during response, what the difference between "connection refused" and "no live upstreams" means, and how to fix each case.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>29. Users report that certain API requests hang for exactly 60 seconds and then return 504 Gateway Timeout — walk through the debugging process: how to identify which timeout is being hit (proxy_read_timeout, proxy_connect_timeout, or upstream application timeout), how to check whether the backend is actually slow vs Nginx is misconfigured, and what the correct timeout tuning strategy looks like for different types of endpoints.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>30. After a deployment, some requests are failing because Nginx is still sending traffic to old backend IPs — explain why this happens (DNS caching in Nginx with upstream blocks), how Nginx resolves DNS for upstream servers at startup vs at runtime, and show the configuration changes (resolver directive, variables in proxy_pass) that fix stale upstream IP issues in dynamic environments like Kubernetes or auto-scaling groups.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>31. You're adding a new /api/v2 path to your Nginx config but requests are unexpectedly hitting a different location block — walk through how Nginx evaluates location blocks (exact match with =, preferential prefix with ^~, regex with ~, and plain prefix), show examples of conflicting location blocks that produce surprising results, and explain the systematic approach to debugging which location block is being selected for a given request.</summary>

<!-- Answer will be added later -->

</details>

---

## Experience-Based Questions

These questions test real-world experience. Prepare by mapping them to your own projects and situations.

<details>
<summary>32. Tell me about a time you configured or optimized an Nginx reverse proxy setup in production — what was the architecture, what problems were you solving, and what configuration decisions had the biggest impact on performance or reliability?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>33. Describe a time you debugged a proxy-related issue in production (502s, 504s, TLS errors, routing problems) — what were the symptoms, how did you narrow down whether the issue was in the proxy layer vs the backend, and what was the root cause?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>34. Tell me about a time you implemented or improved a zero-downtime deployment strategy that involved a reverse proxy or load balancer — what challenges did you face with connection draining, health checks, or rolling updates, and how did you verify zero downtime?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>35. Describe a time you had to decide between managing your own reverse proxy (Nginx, HAProxy) and using a cloud-managed load balancer — what factors drove the decision, what tradeoffs did you accept, and would you make the same choice again?</summary>

<!-- Answer framework will be added later -->

</details>
