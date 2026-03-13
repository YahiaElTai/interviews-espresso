# Docker & Containers

> **25 questions** — 9 theory, 12 practical, 4 experience

- Linux primitives: cgroups (resource limits), namespaces (process/network/mount isolation), and how they combine to create containers
- Docker images, layers, union filesystem, and copy-on-write
- Dockerfile best practices: layer ordering, build cache, multi-stage builds, .dockerignore
- ENTRYPOINT vs CMD: exec form vs shell form, combining them, and how they affect signal handling and overrides
- Container networking modes (bridge, host, none, overlay) and container-to-container communication
- Volumes, bind mounts, and data persistence
- PID 1 problem, signal handling, and graceful shutdown (tini/dumb-init)
- Container security: root vs non-root, attack vectors, container escapes, build-time secrets (BuildKit secrets, multi-stage exclusion), runtime hardening (read-only filesystem, dropping Linux capabilities)
- Docker Compose: multi-service orchestration, depends_on with health checks, local dev with bind mounts and hot-reload, integration testing in CI
- Resource limits (CPU, memory), monitoring usage, and OOM behavior
- Container runtimes: Docker vs containerd vs CRI-O, OCI image and runtime specs, why K8s moved away from dockershim
- Image optimization: slim vs alpine vs distroless base images, image size impact
- Debugging: docker logs, inspect, exec, diagnosing startup failures, build cache misses, and layer debugging

---

## Foundational

<details>
<summary>1. What are containers actually doing at the Linux kernel level — how do namespaces and cgroups work together to create process isolation, why is this fundamentally different from virtual machines, and what does it mean that containers share the host kernel?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>2. How do Docker images work under the hood — what is the layer model, how does the union filesystem combine layers into a single view, how does copy-on-write work at runtime, and why does understanding layers matter for build performance and image size?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>3. What are Docker's networking modes (bridge, host, none, overlay), why does Docker provide multiple modes instead of just one, and how does container-to-container communication work in each — what determines when you'd pick one mode over another?</summary>

<!-- Answer will be added later -->

</details>

## Conceptual Depth

<details>
<summary>4. What is the PID 1 problem in containers — why does the first process in a container behave differently from a normal process regarding signal handling, what goes wrong with zombie processes and graceful shutdown when your application runs as PID 1 directly, and why do tools like tini and dumb-init exist?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>5. Why does Docker have both ENTRYPOINT and CMD instead of a single run command -- what is the difference between exec form and shell form, how do ENTRYPOINT and CMD combine when both are specified, why does exec form matter for signal handling (and how does this connect to the PID 1 problem), and what happens when someone runs docker run with command-line arguments against each configuration?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>6. Why are containers not a security boundary by default — what are the real attack vectors (root in container, kernel exploits, mounted sockets, excessive capabilities), how do container escapes happen, and what does running as non-root actually protect against vs what it doesn't?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>7. Why does Docker separate volumes and bind mounts as distinct mechanisms for data persistence — what problem does each solve, how do their lifecycle and ownership models differ, and what goes wrong when you rely on container-layer storage for data that needs to survive container restarts?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>8. What is the difference between Docker, containerd, and CRI-O as container runtimes — what do the OCI image and runtime specs standardize, why did Kubernetes remove dockershim, and what does this mean for how containers actually run in a Kubernetes cluster vs on a developer's laptop?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>9. Why are containers without resource limits a production risk — how do CPU limits and memory limits work differently at the kernel level, what exactly happens during an OOM kill (who decides, what gets killed, how do you detect it), and how do misconfigured limits cause problems that are worse than no limits at all?</summary>

<!-- Answer will be added later -->

</details>

## Practical — Dockerfile & Build Optimization

<details>
<summary>10. Write a Dockerfile for a Node.js/TypeScript application that demonstrates correct layer ordering for build cache efficiency — show the wrong order first, then the correct order, explain why each layer placement matters, and walk through what happens to the cache when you change application code vs dependencies.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>11. Write a multi-stage Dockerfile for a production Node.js/TypeScript application — show each stage (install dependencies, build, production), explain why separating stages matters for image size and security, and demonstrate how to avoid including devDependencies, source maps, and build tools in the final image.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>12. Compare slim, alpine, and distroless base images for a Node.js application — show the Dockerfile for each, compare the resulting image sizes, explain the tradeoffs in debugging capability, security surface area, and compatibility (musl vs glibc for alpine), and recommend when to use each.</summary>

<!-- Answer will be added later -->

</details>## Practical — Runtime & Compose<details>
<summary>13. Configure proper PID 1 signal handling in a Dockerfile using tini or dumb-init — show the Dockerfile changes, demonstrate the difference in graceful shutdown behavior with and without an init process, and show how to verify that SIGTERM is being properly forwarded to your Node.js application.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>14. Set CPU and memory resource limits on a container — show the docker run flags and Docker Compose equivalents, demonstrate how to monitor actual resource usage with docker stats, explain what happens when a container hits its memory limit (OOM kill) vs its CPU limit (throttling), and show how to detect OOM kills from container inspect output.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>15. Write a Docker Compose file for local development of a Node.js application with a PostgreSQL database and Redis — configure bind mounts for hot-reload (so code changes reflect without rebuilding), show how to handle the node_modules conflict between host and container, and set up environment variables for connecting services together.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>16. Add health checks to a Docker Compose setup — show the health check configuration for both an application container (HTTP endpoint) and a database container (pg_isready), configure service dependencies using depends_on with condition: service_healthy, and explain what Docker does when a health check fails vs what it doesn't do.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>17. Set up a Docker Compose configuration for running integration tests in CI — show how to start the application with its database and run tests against it, handle waiting for services to be ready, ensure clean state between test runs, and demonstrate the CI pipeline commands to bring up, test, and tear down the stack.</summary>

<!-- Answer will be added later -->

</details>

## Practical — Security

<details>
<summary>18. Harden a container's runtime security posture — show how to run as a non-root user (USER, file ownership, directory permissions), enable a read-only filesystem with tmpfs for directories that need writes, and drop all Linux capabilities then add back only what's needed. Explain what commonly breaks with each hardening step (port binding below 1024, file permissions, read-only conflicts with logging/temp files, missing capabilities for network binding) and how to fix each issue.</summary>

<!-- Answer will be added later -->

</details>

## Practical — Debugging & Troubleshooting

<details>
<summary>19. A container exits immediately after starting — walk through the exact debugging steps using docker logs, docker inspect (check exit code and state), and docker run with an interactive shell to reproduce the issue. What are the most common causes of immediate exit (missing entrypoint, failed health check, missing env vars, permission errors) and how do you fix each?</summary>

<!-- Answer will be added later -->

</details><details>
<summary>20. A container cannot reach another container or an external service — walk through the systematic debugging process: check which network the containers are on, verify DNS resolution, test connectivity with curl/wget from inside the container, inspect port mappings, and identify whether the issue is network mode, firewall rules, or application-level binding (0.0.0.0 vs 127.0.0.1). Show the exact commands at each step.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>21. You need to diagnose a running container that is behaving unexpectedly — show how to use docker exec to get a shell, inspect running processes, check file system state, view environment variables, test network connectivity, and read application logs. What do you do when the container image has no shell or debugging tools installed (distroless/scratch)?</summary>

<!-- Answer will be added later -->

</details>

---

## Experience-Based Questions

These questions test real-world experience. Prepare by mapping them to your own projects and situations.

<details>
<summary>22. Tell me about a time you significantly reduced Docker image size or build time — what was the starting point, what specific changes did you make (multi-stage builds, base image switch, layer reordering, .dockerignore), and what was the measurable impact on CI pipeline speed or deployment?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>23. Describe a time you debugged a container issue in production or staging — what were the symptoms, how did you diagnose the root cause (logs, exec, inspect, resource limits), and what did you change to prevent it from recurring?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>24. Tell me about a time you set up or improved a containerized local development environment for your team — what problems existed before (slow rebuilds, environment drift, flaky service dependencies), what did you implement, and how did it affect developer productivity?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>25. Describe a time you dealt with a container security concern — what was the vulnerability or misconfiguration (running as root, exposed secrets, unscanned images, excessive capabilities), how did you discover it, and what remediation steps did you take?</summary>

<!-- Answer framework will be added later -->

</details>
