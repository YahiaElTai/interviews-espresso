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

Containers are not a single kernel feature — they are a combination of two Linux primitives working together:

**Namespaces** provide isolation — they control what a process can *see*:
- **PID namespace**: Process sees itself as PID 1, can't see host processes
- **Network namespace**: Own network stack, interfaces, IP addresses, port space
- **Mount namespace**: Own filesystem view, independent mount points
- **UTS namespace**: Own hostname
- **User namespace**: Map container root to unprivileged host user
- **IPC namespace**: Isolated shared memory and message queues

**Cgroups** (control groups) provide resource limits — they control what a process can *use*:
- CPU time (shares, quota/period for hard limits)
- Memory (hard limit, soft limit, swap)
- I/O bandwidth
- Number of processes (pids)

Together: namespaces make the process *think* it's alone on the machine, cgroups *prevent* it from consuming all the machine's resources.

**VMs vs containers**: A VM runs a full guest OS kernel on virtualized hardware (hypervisor). Each VM boots its own kernel, has its own memory management, scheduler, and device drivers. A container is just a regular Linux process with namespace isolation and cgroup limits — it makes system calls directly to the host kernel.

**What "sharing the host kernel" means in practice**:
- Containers start in milliseconds (no kernel boot), VMs take seconds to minutes
- Containers use far less memory (no duplicate OS)
- A kernel vulnerability in the host affects all containers — there's no hypervisor boundary. This is why containers are not a security boundary equivalent to VMs
- All containers must be Linux containers on a Linux host (Docker Desktop on Mac/Windows runs a Linux VM underneath)
- A kernel bug or panic in one container can crash the entire host

</details>

<details>
<summary>2. How do Docker images work under the hood — what is the layer model, how does the union filesystem combine layers into a single view, how does copy-on-write work at runtime, and why does understanding layers matter for build performance and image size?</summary>

**Layer model**: Each instruction in a Dockerfile (`FROM`, `RUN`, `COPY`, `ADD`) creates a new read-only layer. A layer is a diff — it only contains files that changed from the previous layer. Layers are content-addressed (SHA256 hash), so identical layers are stored once and shared across images.

**Union filesystem** (overlay2 on modern Linux): Stacks all the read-only layers on top of each other and presents a single unified view. If the same file exists in multiple layers, the topmost version wins. The container sees a normal filesystem — the layering is invisible.

**Copy-on-write at runtime**: When a container starts, Docker adds a thin writable layer on top of the read-only image layers. Reads pass through to the image layers. When a container modifies a file, that file is copied from the image layer into the writable layer, and the modification happens there. The original image layer is never changed — this is why multiple containers can share the same image without conflicts.

**Why this matters for build performance**:

Docker caches each layer. When rebuilding, if an instruction and its inputs haven't changed, Docker reuses the cached layer. But cache invalidation is sequential — once one layer is invalidated, all subsequent layers rebuild too.

```dockerfile
# Bad: any code change invalidates npm install cache
COPY . .
RUN npm install
RUN npm run build

# Good: dependencies cached separately from code
COPY package.json package-lock.json ./
RUN npm install
COPY . .
RUN npm run build
```

**Why this matters for image size**:
- Each layer adds to the total size, even if a later layer deletes files (the delete is recorded as a "whiteout" — the original bytes still exist in the earlier layer)
- `RUN apt-get install && apt-get clean` in a single `RUN` instruction keeps the layer small; splitting them across two `RUN` instructions means the installed files persist in the first layer regardless of the cleanup in the second
- Multi-stage builds solve this completely by copying only specific artifacts into a fresh image

</details>

<details>
<summary>3. What are Docker's networking modes (bridge, host, none, overlay), why does Docker provide multiple modes instead of just one, and how does container-to-container communication work in each — what determines when you'd pick one mode over another?</summary>

**Bridge (default)**: Docker creates a virtual network bridge (`docker0`). Each container gets its own network namespace with a virtual ethernet interface connected to the bridge. Containers on the same bridge can communicate via IP. On user-defined bridge networks, Docker provides DNS resolution by container name — the default bridge does not.

```bash
# Create a user-defined bridge (recommended over default)
docker network create app-net
docker run --network app-net --name api my-api
docker run --network app-net --name db postgres
# "api" can reach "db" by hostname: postgres://db:5432
```

**Host**: Container shares the host's network namespace entirely. No network isolation — the container's processes bind directly to host ports. No NAT, no port mapping needed. Maximum network performance (no bridge overhead).

**When to use**: Performance-sensitive workloads where the ~1-2% bridge overhead matters, or monitoring tools that need to see host network traffic. Trade: you lose port isolation — two containers can't both bind port 3000.

**None**: Container gets a network namespace with only a loopback interface. Completely network-isolated. Used for batch processing, security-sensitive computations, or containers that only need filesystem interaction.

**Overlay**: Multi-host networking using VXLAN encapsulation. Containers on different Docker hosts can communicate as if they're on the same network. Used in Docker Swarm and can be used with Kubernetes.

**When to use which**:

| Mode | Isolation | DNS | Multi-host | Use case |
|------|-----------|-----|------------|----------|
| Bridge (user-defined) | Yes | Yes | No | Default for most workloads, local dev with Compose |
| Host | None | N/A | No | Performance-critical, network monitoring |
| None | Total | No | No | Security-sensitive batch jobs |
| Overlay | Yes | Yes | Yes | Swarm/multi-host clusters |

Docker Compose automatically creates a user-defined bridge network for each project, which is why services can reference each other by name out of the box.

</details>

## Conceptual Depth

<details>
<summary>4. What is the PID 1 problem in containers — why does the first process in a container behave differently from a normal process regarding signal handling, what goes wrong with zombie processes and graceful shutdown when your application runs as PID 1 directly, and why do tools like tini and dumb-init exist?</summary>

On Linux, PID 1 (the init process) has special kernel behavior that differs from every other process:

**Signal handling difference**: The kernel does NOT deliver signals to PID 1 unless PID 1 has explicitly registered a handler for that signal. For a normal process, `SIGTERM` has a default action (terminate). For PID 1, `SIGTERM` with no registered handler is silently ignored. When you run `docker stop`, Docker sends `SIGTERM`, waits 10 seconds, then sends `SIGKILL`. If your app is PID 1 and doesn't handle `SIGTERM`, it never shuts down gracefully — it always gets force-killed after the timeout.

**Zombie reaping**: PID 1 is responsible for reaping orphaned child processes (calling `wait()` on them). When a child process's parent dies, the child is re-parented to PID 1. If PID 1 doesn't reap these orphans, they accumulate as zombie processes (`<defunct>`). Most application runtimes (Node.js included) don't implement zombie reaping because they're not designed to be init systems.

**What goes wrong in practice**:
- `docker stop` takes the full 10-second timeout and then force-kills, meaning in-flight requests are dropped, database connections aren't closed cleanly, and graceful shutdown hooks don't run
- Zombie processes accumulate if the app spawns child processes (e.g., `child_process.exec`)
- Health checks may report healthy while the process is actually stuck

**Why tini/dumb-init exist**: They run as PID 1 instead of your application, handling the init responsibilities:
1. Register signal handlers and forward signals to child processes (so `SIGTERM` reaches your app)
2. Reap zombie processes
3. Exit with the child's exit code

```dockerfile
# Using tini (Docker's --init flag does this automatically)
RUN apk add --no-cache tini
ENTRYPOINT ["/sbin/tini", "--"]
CMD ["node", "dist/main.js"]
```

Or simply: `docker run --init my-app` — Docker bundles tini and injects it as PID 1.

Node.js can handle `SIGTERM` directly if you register the handler, but it still won't reap zombies. Using tini is the safer default.

</details>

<details>
<summary>5. Why does Docker have both ENTRYPOINT and CMD instead of a single run command -- what is the difference between exec form and shell form, how do ENTRYPOINT and CMD combine when both are specified, why does exec form matter for signal handling (and how does this connect to the PID 1 problem), and what happens when someone runs docker run with command-line arguments against each configuration?</summary>

**Exec form vs shell form**:

```dockerfile
# Exec form — runs the binary directly as PID 1
ENTRYPOINT ["node", "dist/main.js"]

# Shell form — wraps in /bin/sh -c, shell becomes PID 1
ENTRYPOINT node dist/main.js
# Actually runs: /bin/sh -c "node dist/main.js"
```

Shell form means `/bin/sh` is PID 1, not your application. Signals (`SIGTERM`) go to the shell, which does not forward them to the child. Your app never receives the shutdown signal — connecting directly to the PID 1 problem (covered in question 4).

**How ENTRYPOINT and CMD combine**:

ENTRYPOINT defines the executable. CMD provides default arguments. When both are specified in exec form, CMD values are appended to ENTRYPOINT:

```dockerfile
ENTRYPOINT ["node"]
CMD ["dist/main.js"]
# Runs: node dist/main.js
```

**What `docker run` arguments do**:

| Dockerfile config | `docker run myapp` | `docker run myapp --debug` |
|---|---|---|
| `CMD ["node", "main.js"]` | `node main.js` | `--debug` (CMD entirely replaced) |
| `ENTRYPOINT ["node", "main.js"]` | `node main.js` | `node main.js --debug` (appended) |
| `ENTRYPOINT ["node"]` + `CMD ["main.js"]` | `node main.js` | `node --debug` (CMD replaced, ENTRYPOINT kept) |

This is why Docker has both — ENTRYPOINT defines the fixed part of the command, CMD defines the part users can easily override. For a typical service where you always want `node dist/main.js`:

```dockerfile
# Recommended: exec form, app is PID 1, signals work correctly
ENTRYPOINT ["node", "dist/main.js"]
```

Or with tini:
```dockerfile
ENTRYPOINT ["/sbin/tini", "--"]
CMD ["node", "dist/main.js"]
```

**Key rule**: Always use exec form for ENTRYPOINT and CMD in production images. Shell form breaks signal handling.

</details>

<details>
<summary>6. Why are containers not a security boundary by default — what are the real attack vectors (root in container, kernel exploits, mounted sockets, excessive capabilities), how do container escapes happen, and what does running as non-root actually protect against vs what it doesn't?</summary>

Containers share the host kernel. Namespaces provide isolation, but they are not a security boundary in the way a hypervisor is. There is no hardware-level separation — a kernel exploit from inside a container affects the host directly.

**Real attack vectors**:

**Root in container = root on host (by default)**: Without user namespace remapping, UID 0 inside the container IS UID 0 on the host. If a container process can escape the namespace (via a kernel vulnerability, a misconfigured volume mount, or a capability exploit), it has root access to the host.

**Mounted Docker socket**: `-v /var/run/docker.sock:/var/run/docker.sock` gives the container full control over the Docker daemon — it can create privileged containers, mount the host filesystem, and effectively has root on the host. This is the most common container escape in practice.

**Kernel exploits**: Containers make syscalls directly to the host kernel. A kernel vulnerability (e.g., Dirty COW, container escape CVEs) exploitable from inside a container compromises the host. VMs don't have this risk because they have their own kernel.

**Excessive Linux capabilities**: By default, Docker grants a set of capabilities (e.g., `CAP_NET_RAW`, `CAP_SYS_CHOWN`). The `--privileged` flag grants ALL capabilities plus access to host devices — effectively no isolation at all.

**How container escapes happen**:
1. Exploit a kernel vulnerability from within the container
2. Abuse a mounted host path (e.g., `/`, `/etc`, Docker socket)
3. Use `--privileged` to access host devices and filesystems
4. Exploit writable cgroup filesystem to execute commands on the host

**What non-root protects against**:
- Prevents binding to privileged ports (<1024) inside the container
- Limits damage from application-level exploits (RCE gives attacker unprivileged access)
- Prevents modifying system files inside the container
- Combined with user namespace remapping, maps container "root" to an unprivileged host UID

**What non-root does NOT protect against**:
- Kernel exploits (these work from any UID)
- Mounted Docker socket abuse (socket permissions, not UID, control access)
- Information leakage from shared kernel resources (`/proc`, `/sys`)

Non-root is necessary but not sufficient. Defense in depth means: non-root + drop capabilities + read-only filesystem + no mounted socket + image scanning + seccomp profiles.

</details>

<details>
<summary>7. Why does Docker separate volumes and bind mounts as distinct mechanisms for data persistence — what problem does each solve, how do their lifecycle and ownership models differ, and what goes wrong when you rely on container-layer storage for data that needs to survive container restarts?</summary>

**The core problem**: Container-layer storage (the writable layer) is ephemeral. When a container is removed, its writable layer is deleted. Even `docker stop` + `docker start` preserves it, but `docker rm` destroys it. Additionally, the writable layer uses copy-on-write, which is slower than direct filesystem access for write-heavy workloads like databases.

**Volumes** — Docker-managed storage:
- Stored in Docker's data directory (`/var/lib/docker/volumes/`)
- Docker manages the lifecycle — created with `docker volume create`, persists independently of containers
- Can be named or anonymous. Named volumes persist until explicitly removed. Anonymous volumes are removed with the container if `--rm` is used
- Portable across hosts when using volume drivers (e.g., NFS, cloud block storage)
- Docker can set correct ownership when the volume is first populated from a container

```bash
docker volume create pgdata
docker run -v pgdata:/var/lib/postgresql/data postgres
# Volume survives container removal
```

**Bind mounts** — host directory mapped into the container:
- Directly mounts a host path into the container
- Docker does NOT manage the lifecycle — the directory exists on the host independently
- File ownership follows the host UID/GID, which can cause permission issues when the container runs as a different user
- Changes are bidirectional and immediate — host changes appear in the container and vice versa

```bash
docker run -v $(pwd)/src:/app/src my-app
# Used for local development — edit on host, hot-reload in container
```

**When to use which**:
- **Volumes**: Production data (databases, file uploads), anything that needs to survive container lifecycle, anything shared between containers
- **Bind mounts**: Local development (source code with hot-reload), configuration files that live in the repo
- **Neither (container layer)**: Scratch data, temp files, caches that can be regenerated

**Common pitfall**: Running a database with data in the container layer. When the container is recreated (common in orchestrators), all data is lost. This is the number one "it works in dev" disaster.

</details>

<details>
<summary>8. What is the difference between Docker, containerd, and CRI-O as container runtimes — what do the OCI image and runtime specs standardize, why did Kubernetes remove dockershim, and what does this mean for how containers actually run in a Kubernetes cluster vs on a developer's laptop?</summary>

**The runtime stack (top to bottom)**:

**Docker** is a full platform: CLI + Docker daemon (dockerd) + image building + networking + volumes + containerd + runc. It's the developer experience layer. When you run `docker run`, dockerd handles image pulling, network setup, and volume management, then delegates the actual container creation to containerd.

**containerd** is a high-level container runtime (CRI-compliant). It manages the complete container lifecycle: image pull/push, storage, container creation/deletion, and supervision. It calls a low-level runtime (runc) to actually create the container using Linux namespaces and cgroups. containerd is what Docker uses internally and what most Kubernetes clusters use directly.

**CRI-O** is a lightweight CRI-compliant runtime built specifically for Kubernetes. It does the minimum needed to run containers for K8s — no image building, no CLI. Smaller attack surface and footprint than containerd.

**runc** is the low-level OCI runtime that all of the above eventually call. It takes a filesystem bundle and an OCI runtime config, creates the namespaces and cgroups, and starts the process.

**OCI specs standardize two things**:
- **Image spec**: How images are packaged (layers, manifests, config). An image built with Docker works with containerd, CRI-O, or any OCI-compliant runtime.
- **Runtime spec**: How a container is configured and started (process, mounts, namespaces, cgroups). This is the JSON config that runc consumes.

**Why Kubernetes removed dockershim**: Kubernetes needed a CRI (Container Runtime Interface) to talk to runtimes. Docker predated CRI and didn't implement it. So Kubernetes maintained a shim (`dockershim`) that translated CRI calls to Docker API calls. The call chain was: kubelet -> dockershim -> dockerd -> containerd -> runc. Removing dockershim simplified this to: kubelet -> containerd -> runc. Docker was just unnecessary middleware in the K8s context.

**What this means in practice**:
- **Developer laptop**: Docker remains the standard tool. You use `docker build`, `docker run`, `docker compose`. Nothing changed.
- **Kubernetes cluster**: Nodes run containerd or CRI-O directly. No Docker daemon. Images built with Docker still work because of OCI compatibility.
- Your Dockerfiles, images, and registries are unaffected. The OCI specs ensure interoperability.

</details>

<details>
<summary>9. Why are containers without resource limits a production risk — how do CPU limits and memory limits work differently at the kernel level, what exactly happens during an OOM kill (who decides, what gets killed, how do you detect it), and how do misconfigured limits cause problems that are worse than no limits at all?</summary>

Without resource limits, a single container can consume all host CPU and memory, starving other containers and the host OS itself. In a Kubernetes cluster, the scheduler can't make intelligent placement decisions without resource declarations.

**CPU limits vs memory limits — fundamentally different enforcement**:

**CPU limits** use CFS (Completely Fair Scheduler) bandwidth control. You specify a quota and period (e.g., `--cpus=0.5` means 50ms of CPU time per 100ms period). When a container exhausts its quota, the kernel *throttles* it — the process is paused until the next period. CPU throttling degrades performance but never kills the process. The container just runs slower.

**Memory limits** use cgroup memory controller with a hard ceiling. When a container tries to allocate beyond its limit, the kernel's OOM killer terminates the process. There's no "slow down" — it's a cliff. The container is killed.

**What happens during an OOM kill**:
1. Container process requests memory beyond cgroup limit
2. Kernel's OOM killer selects a process to kill (highest `oom_score_adj` + memory usage)
3. Process receives `SIGKILL` (not `SIGTERM` — cannot be caught or handled)
4. Container exits with code 137 (128 + 9, where 9 = SIGKILL)
5. Docker records `OOMKilled: true` in container state

```bash
# Detect OOM kill
docker inspect <container> --format='{{.State.OOMKilled}}'
# true

# Check exit code
docker inspect <container> --format='{{.State.ExitCode}}'
# 137
```

**How misconfigured limits are worse than no limits**:

- **Memory limit too low**: Application OOM-killed under normal load. Looks like a crash, not a resource issue. Teams spend hours debugging "random crashes" that are actually OOM kills. Node.js is especially vulnerable because V8's garbage collector needs headroom — setting memory limit exactly at peak usage means GC pressure triggers OOM.
- **CPU limit too low**: CFS throttling causes latency spikes. A Node.js server that needs 200ms of CPU per request but is limited to 100ms/100ms period will see 2x response times and timeout cascades. Throttling is invisible unless you specifically monitor `cpu.stat` throttled periods.
- **Memory limit without CPU limit**: Container uses all available CPU for GC when near memory limit, starving neighbors.

**Practical guidance**: Set memory limits with ~25% headroom above measured peak usage. Set CPU limits based on measured P99 usage, not average. Always set both or set neither — asymmetric limits create surprising failure modes.

</details>

## Practical — Dockerfile & Build Optimization

<details>
<summary>10. Write a Dockerfile for a Node.js/TypeScript application that demonstrates correct layer ordering for build cache efficiency — show the wrong order first, then the correct order, explain why each layer placement matters, and walk through what happens to the cache when you change application code vs dependencies.</summary>

**Wrong order — every code change reinstalls dependencies**:

```dockerfile
FROM node:20-slim

WORKDIR /app

# BAD: copies everything, including source code
COPY . .

# This layer is invalidated every time ANY file changes
RUN npm ci

RUN npm run build

CMD ["node", "dist/main.js"]
```

The problem: `COPY . .` includes source files. When you change `src/app.ts`, the `COPY` layer is invalidated. Docker's cache invalidation is sequential — every layer after an invalidated layer also rebuilds. So `npm ci` runs again (2-3 minutes) even though `package.json` didn't change.

**Correct order — dependencies cached separately from source code**:

```dockerfile
FROM node:20-slim

WORKDIR /app

# Layer 1: copy only dependency manifests (changes rarely)
COPY package.json package-lock.json ./

# Layer 2: install dependencies (cached unless package*.json changed)
RUN npm ci

# Layer 3: copy source code (changes frequently)
COPY . .

# Layer 4: build TypeScript (rebuilds when source changes)
RUN npm run build

CMD ["node", "dist/main.js"]
```

**Cache walkthrough — when you change source code** (`src/app.ts`):
- Layer 1 (`COPY package.json`): Cache HIT — these files didn't change
- Layer 2 (`RUN npm ci`): Cache HIT — inputs unchanged
- Layer 3 (`COPY . .`): Cache MISS — source files changed
- Layer 4 (`RUN npm run build`): Rebuilds (after invalidated layer)
- Result: Skip the expensive `npm ci`, only rebuild TypeScript (~seconds)

**Cache walkthrough — when you add a dependency** (`package.json` changes):
- Layer 1 (`COPY package.json`): Cache MISS — file changed
- All subsequent layers rebuild
- Result: Full rebuild, but this happens infrequently

**Additional optimization — add `.dockerignore`**:

```
node_modules
dist
.git
*.md
.env*
```

Without `.dockerignore`, `COPY . .` includes `node_modules` (hundreds of MB), `.git` history, and build artifacts — bloating the build context sent to the Docker daemon and invalidating cache unnecessarily.

**The principle**: Order layers from least-frequently-changed to most-frequently-changed. Expensive operations (dependency install) should be as early as possible, behind a narrow cache key (just the lockfile).

</details>

<details>
<summary>11. Write a multi-stage Dockerfile for a production Node.js/TypeScript application — show each stage (install dependencies, build, production), explain why separating stages matters for image size and security, and demonstrate how to avoid including devDependencies, source maps, and build tools in the final image.</summary>

```dockerfile
# Stage 1: Install ALL dependencies (including devDependencies for build)
FROM node:20-slim AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci

# Stage 2: Build TypeScript
FROM deps AS build
COPY tsconfig.json ./
COPY src/ ./src/
RUN npm run build
# Remove source maps if generated
RUN rm -f dist/**/*.map

# Stage 3: Production image — only runtime deps + compiled output
FROM node:20-slim AS production
WORKDIR /app

# Install only production dependencies
COPY package.json package-lock.json ./
RUN npm ci --omit=dev && npm cache clean --force

# Copy compiled JavaScript from build stage
COPY --from=build /app/dist ./dist

# Security hardening (covered in detail in question 18)
USER node

EXPOSE 3000
CMD ["node", "dist/main.js"]
```

**Why each stage matters**:

**Stage 1 (deps)**: Installs all dependencies including `devDependencies` (TypeScript compiler, test frameworks, linters). These are needed to build but should never ship to production.

**Stage 2 (build)**: Compiles TypeScript to JavaScript. This stage has access to `tsc`, build tools, and source code. The compiled output in `dist/` is the only artifact that matters.

**Stage 3 (production)**: Starts from a clean `node:20-slim` image. Only `npm ci --omit=dev` runs here, so `typescript`, `jest`, `eslint`, and every other dev tool is excluded. The compiled JS is copied from the build stage.

**What gets excluded from the final image**:
- TypeScript source files (`src/`)
- `devDependencies` (TypeScript compiler, test frameworks, linters)
- Source maps (`.map` files — exposing these in production leaks source code)
- Build tools and intermediate artifacts
- npm cache

**Size impact**: A typical Node.js app with TypeScript, Jest, and ESLint might have a `deps` stage at ~800MB. The production stage is often 150-200MB — the `devDependencies` alone can be 3-5x the size of production dependencies.

**Security impact**: Fewer packages = smaller attack surface. No compiler, no test framework, no build tools that could be exploited if an attacker gets RCE.

</details>

<details>
<summary>12. Compare slim, alpine, and distroless base images for a Node.js application — show the Dockerfile for each, compare the resulting image sizes, explain the tradeoffs in debugging capability, security surface area, and compatibility (musl vs glibc for alpine), and recommend when to use each.</summary>

**Slim** (Debian-based, stripped down):
```dockerfile
FROM node:20-slim
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --omit=dev
COPY dist/ ./dist/
CMD ["node", "dist/main.js"]
```

**Alpine** (musl libc, BusyBox):
```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --omit=dev
COPY dist/ ./dist/
CMD ["node", "dist/main.js"]
```

**Distroless** (Google's minimal images — no shell, no package manager):
```dockerfile
FROM node:20-slim AS build
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --omit=dev
COPY dist/ ./dist/

FROM gcr.io/distroless/nodejs20-debian12
WORKDIR /app
COPY --from=build /app ./
CMD ["dist/main.js"]
# Note: ENTRYPOINT is already set to ["nodejs"] in distroless/nodejs,
# so CMD provides the argument — exec form CMD is appended to the image's built-in ENTRYPOINT
```

**Comparison**:

| Base image | Approx. size | Shell | Package manager | C library | Debugging |
|---|---|---|---|---|---|
| `node:20` (full) | ~1 GB | Yes | apt | glibc | Full tooling |
| `node:20-slim` | ~200 MB | Yes | apt (minimal) | glibc | Basic tools available |
| `node:20-alpine` | ~130 MB | Yes (sh) | apk | musl | BusyBox utilities |
| `distroless/nodejs20` | ~120 MB | No | No | glibc | None (must use debug variant or ephemeral containers) |

**Tradeoffs**:

**Alpine — musl vs glibc**: Alpine uses musl libc instead of glibc. Most pure JavaScript packages work fine. But native addons compiled against glibc (e.g., `bcrypt`, `sharp`, some database drivers) may fail or need to be recompiled. `npm ci` may trigger native builds that fail silently or produce segfaults. This is the most common "works on slim, breaks on alpine" issue.

**Distroless — no shell**: You cannot `docker exec -it <container> sh` into a distroless container. No debugging tools, no file exploration at runtime. Google provides a `:debug` variant with a BusyBox shell for troubleshooting. In Kubernetes, you can use ephemeral debug containers instead.

**Security surface area**: Fewer installed packages = fewer CVEs. Distroless has the smallest surface. Alpine is small but includes BusyBox (which has had its own CVEs). Slim includes more Debian packages but uses the well-audited glibc.

**Recommendations**:
- **`node:20-slim`**: Best default. Good balance of size, compatibility, and debuggability. Use this unless you have a specific reason not to.
- **`node:20-alpine`**: When image size is a priority and you've verified your dependencies work with musl. Good for simple apps with no native addons.
- **Distroless**: Maximum security posture. Best for production services where you've committed to debugging via logs and metrics rather than shell access. Requires Kubernetes ephemeral containers or the `:debug` variant for troubleshooting.

</details>

---

## Practical — Runtime & Compose

<details>
<summary>13. Configure proper PID 1 signal handling in a Dockerfile using tini or dumb-init — show the Dockerfile changes, demonstrate the difference in graceful shutdown behavior with and without an init process, and show how to verify that SIGTERM is being properly forwarded to your Node.js application.</summary>

**Option 1: tini in Dockerfile** (most common):

```dockerfile
FROM node:20-slim

RUN apt-get update && apt-get install -y --no-install-recommends tini \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --omit=dev
COPY dist/ ./dist/

# tini is PID 1, forwards signals to node
ENTRYPOINT ["/usr/bin/tini", "--"]
CMD ["node", "dist/main.js"]
```

**Option 2: Docker's built-in `--init` flag** (no Dockerfile change):

```bash
docker run --init my-app
# Docker injects its bundled tini as PID 1
```

In Docker Compose:
```yaml
services:
  api:
    build: .
    init: true  # equivalent to --init
```

**Option 3: dumb-init** (alternative to tini):

```dockerfile
RUN apt-get update && apt-get install -y --no-install-recommends dumb-init \
    && rm -rf /var/lib/apt/lists/*
ENTRYPOINT ["/usr/bin/dumb-init", "--"]
CMD ["node", "dist/main.js"]
```

**Behavior without init process** (node as PID 1):

```bash
$ docker stop my-app
# Docker sends SIGTERM to PID 1 (node)
# If node has no SIGTERM handler → signal ignored (PID 1 kernel behavior, see question 4)
# 10 seconds pass...
# Docker sends SIGKILL → process force-killed
# Exit code: 137
# In-flight requests dropped, DB connections not closed
```

**Behavior with tini**:

```bash
$ docker stop my-app
# Docker sends SIGTERM to PID 1 (tini)
# tini forwards SIGTERM to child (node)
# Node's SIGTERM handler runs → graceful shutdown
# Exit code: 0 (or whatever node exits with)
```

**Node.js graceful shutdown handler** (still needed even with tini):

```typescript
process.on('SIGTERM', async () => {
  console.log('SIGTERM received, shutting down gracefully');
  await server.close();       // stop accepting new connections
  await database.disconnect(); // close DB pool
  process.exit(0);
});
```

**Verifying signals are forwarded correctly**:

```bash
# Start container
docker run -d --name test my-app

# Check PID 1 inside the container
docker exec test ps aux
# With tini: PID 1 is /usr/bin/tini, node is PID ~7
# Without:  PID 1 is node

# Send SIGTERM and watch logs
docker stop test && docker logs test
# Should see "SIGTERM received, shutting down gracefully"

# Check exit code — 0 means graceful, 137 means force-killed
docker inspect test --format='{{.State.ExitCode}}'
```

**Alpine note**: On alpine, tini is available via `apk add --no-cache tini` and installs to `/sbin/tini`.

</details>

<details>
<summary>14. Set CPU and memory resource limits on a container — show the docker run flags and Docker Compose equivalents, demonstrate how to monitor actual resource usage with docker stats, explain what happens when a container hits its memory limit (OOM kill) vs its CPU limit (throttling), and show how to detect OOM kills from container inspect output.</summary>

**Docker run flags**:

```bash
docker run \
  --memory=512m \          # hard memory limit (OOM kill above this)
  --memory-reservation=256m \ # soft limit (hint for scheduler)
  --cpus=0.5 \             # 50% of one CPU core
  --cpu-shares=512 \       # relative weight (default 1024) — only matters under contention
  my-app
```

**Docker Compose equivalent**:

```yaml
services:
  api:
    build: .
    deploy:
      resources:
        limits:
          memory: 512M
          cpus: '0.5'
        reservations:
          memory: 256M
          cpus: '0.25'
```

Note: In Compose v2, `deploy.resources` works with `docker compose up` directly. The older `mem_limit`/`cpus` top-level keys are deprecated.

**Monitoring with docker stats**:

```bash
$ docker stats --no-stream
CONTAINER   CPU %   MEM USAGE / LIMIT   MEM %   NET I/O       BLOCK I/O
api         12.5%   234MiB / 512MiB     45.7%   1.2MB/500kB   0B/0B
db          3.2%    180MiB / 1GiB       17.6%   800kB/1.5MB   5MB/20MB
```

Key columns: `MEM USAGE / LIMIT` shows current usage against the hard cap. `CPU %` shows percentage of allocated CPU (not host CPU).

**Memory limit hit — OOM kill** (covered conceptually in question 9, here's the practical detection):

```bash
# Container exits with code 137
docker inspect api --format='{{.State.ExitCode}}'
# 137

# Confirm OOM
docker inspect api --format='{{.State.OOMKilled}}'
# true

# Check in logs — the kernel log may also show:
# "Memory cgroup out of memory: Killed process <pid>"
dmesg | grep -i oom
```

**CPU limit hit — throttling**: The container keeps running but is paused when it exceeds its quota. No exit, no kill — just slower. Throttling is invisible in `docker stats`. To detect it:

```bash
# Find the cgroup path
docker inspect api --format='{{.HostConfig.CgroupParent}}'

# Check throttling stats (on host)
# cgroup v1 (older distros):
cat /sys/fs/cgroup/cpu/docker/<container-id>/cpu.stat
# cgroup v2 (Ubuntu 22.04+, Debian 12+, Fedora 31+ — now the default):
cat /sys/fs/cgroup/system.slice/docker-<container-id>.scope/cpu.stat

# nr_periods 1000
# nr_throttled 250    ← 25% of periods were throttled
# throttled_time 5000000000  ← nanoseconds spent throttled
```

**Key difference**: Memory limits are a cliff (process dies). CPU limits are a slope (process slows). This is why memory limit misconfiguration is more dangerous — there's no warning before the kill.

</details>

<details>
<summary>15. Write a Docker Compose file for local development of a Node.js application with a PostgreSQL database and Redis — configure bind mounts for hot-reload (so code changes reflect without rebuilding), show how to handle the node_modules conflict between host and container, and set up environment variables for connecting services together.</summary>

```yaml
services:
  api:
    build:
      context: .
      dockerfile: Dockerfile.dev
    ports:
      - "3000:3000"
    volumes:
      - .:/app                    # bind mount — source code synced for hot-reload
      - /app/node_modules         # anonymous volume — prevents host node_modules from overwriting container's
    environment:
      NODE_ENV: development
      DATABASE_URL: postgres://postgres:password@db:5432/myapp
      REDIS_URL: redis://redis:6379
    depends_on:
      - db
      - redis

  db:
    image: postgres:16
    ports:
      - "5432:5432"
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password
      POSTGRES_DB: myapp
    volumes:
      - pgdata:/var/lib/postgresql/data  # named volume — data persists across restarts

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

volumes:
  pgdata:
```

**The `Dockerfile.dev`** (development-only, with hot-reload):

```dockerfile
FROM node:20-slim
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
# Don't COPY source — bind mount handles it
CMD ["npx", "tsx", "watch", "src/main.ts"]
```

**The node_modules conflict explained**:

The bind mount `.:/app` maps your entire project directory into the container. If your host has `node_modules/` (e.g., from running `npm install` on macOS), it overwrites the container's Linux-compiled `node_modules`. Native addons compiled for macOS won't work inside the Linux container.

The fix: `/app/node_modules` as an anonymous volume. Docker creates an empty volume at `/app/node_modules` that takes precedence over the bind mount at that path. The container uses its own `node_modules` (installed during `docker build`), while the bind mount provides the source code.

**How services connect**: Docker Compose creates a user-defined bridge network automatically (as covered in question 3). Services reference each other by service name: `db` resolves to the PostgreSQL container's IP, `redis` resolves to Redis. The `DATABASE_URL` and `REDIS_URL` use these hostnames directly.

**Hot-reload flow**: You edit `src/main.ts` on your host. The bind mount syncs the change instantly into the container. `tsx watch` detects the file change and restarts the server. No `docker build` or `docker compose restart` needed.

</details>

<details>
<summary>16. Add health checks to a Docker Compose setup — show the health check configuration for both an application container (HTTP endpoint) and a database container (pg_isready), configure service dependencies using depends_on with condition: service_healthy, and explain what Docker does when a health check fails vs what it doesn't do.</summary>

```yaml
services:
  api:
    build: .
    ports:
      - "3000:3000"
    environment:
      DATABASE_URL: postgres://postgres:password@db:5432/myapp
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 10s      # how often to check
      timeout: 5s        # max time for a single check
      retries: 3         # failures before "unhealthy"
      start_period: 15s  # grace period for startup (failures don't count)
    depends_on:
      db:
        condition: service_healthy  # api won't start until db is healthy

  db:
    image: postgres:16
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password
      POSTGRES_DB: myapp
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 3s
      retries: 5
      start_period: 10s

  redis:
    image: redis:7-alpine
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 3
```

**Health check states**: A container transitions through `starting` -> `healthy` or `unhealthy`. During `start_period`, failures don't count toward retries.

**What Docker does when a health check fails**:
- Sets container status to `unhealthy` (visible in `docker ps` and `docker inspect`)
- Emits a `health_status` event
- **That's it.** Docker itself does NOT restart the container, stop it, or take any action.

**What Docker does NOT do**:
- Does not restart the container (unless you add `restart: always` or `restart: on-failure`)
- Does not remove it from a load balancer (that's Kubernetes or Swarm's job)
- Does not affect running processes inside the container

**`depends_on` with `condition: service_healthy`** only affects startup ordering. It makes Docker Compose wait until the dependency's health check passes before starting the dependent service. Once started, `depends_on` has no effect — if the database goes unhealthy later, the API container keeps running.

**Gotcha with curl**: The `node:20-slim` image may not have `curl` installed. Alternatives:

```yaml
# Option 1: Use wget (often available)
test: ["CMD", "wget", "--spider", "-q", "http://localhost:3000/health"]

# Option 2: Use Node.js itself (always available)
test: ["CMD", "node", "-e", "fetch('http://localhost:3000/health').then(r => process.exit(r.ok ? 0 : 1)).catch(() => process.exit(1))"]
```

</details>

<details>
<summary>17. Set up a Docker Compose configuration for running integration tests in CI — show how to start the application with its database and run tests against it, handle waiting for services to be ready, ensure clean state between test runs, and demonstrate the CI pipeline commands to bring up, test, and tear down the stack.</summary>

**`docker-compose.test.yml`**:

```yaml
services:
  api:
    build: .
    environment:
      NODE_ENV: test
      DATABASE_URL: postgres://postgres:password@db:5432/testdb
      REDIS_URL: redis://redis:6379
    healthcheck:
      test: ["CMD", "node", "-e", "fetch('http://localhost:3000/health').then(r => process.exit(r.ok ? 0 : 1)).catch(() => process.exit(1))"]
      interval: 5s
      timeout: 3s
      retries: 10
      start_period: 20s
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy

  db:
    image: postgres:16
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password
      POSTGRES_DB: testdb
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 3s
      timeout: 3s
      retries: 10
    # No volume — data is ephemeral, wiped on teardown

  redis:
    image: redis:7-alpine
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 3s
      timeout: 3s
      retries: 5

  test-runner:
    build: .
    command: ["npm", "run", "test:integration"]
    environment:
      API_URL: http://api:3000
      DATABASE_URL: postgres://postgres:password@db:5432/testdb
    depends_on:
      api:
        condition: service_healthy
```

**CI pipeline commands** (GitHub Actions example):

```yaml
jobs:
  integration-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Run integration tests
        run: |
          docker compose -f docker-compose.test.yml up \
            --build \
            --abort-on-container-exit \
            --exit-code-from test-runner

      - name: Teardown
        if: always()
        run: |
          docker compose -f docker-compose.test.yml down --volumes --remove-orphans
```

**Key flags explained**:
- `--build`: Rebuild images (don't use stale cache in CI)
- `--abort-on-container-exit`: When any container exits (the test-runner), stop everything
- `--exit-code-from test-runner`: Use test-runner's exit code as the command's exit code (0 = tests passed, non-zero = failed)
- `down --volumes`: Remove containers AND volumes — clean state for next run
- `down --remove-orphans`: Remove containers from previous runs that aren't in the current config

**Ensuring clean state between runs**:
1. No persistent volumes in the test Compose file — every `up` starts with an empty database
2. `down --volumes` in the teardown step ensures nothing lingers
3. If tests need to run multiple times within a single `up`, have the test framework handle database cleanup (truncate tables between test suites)

**Handling service readiness**: The `depends_on` with `condition: service_healthy` chain ensures correct startup: db -> api -> test-runner. The `test-runner` won't start until the API's health endpoint returns 200, which in turn won't happen until the API can connect to PostgreSQL. No manual sleep or retry scripts needed.

</details>

## Practical — Security

<details>
<summary>18. Harden a container's runtime security posture — show how to run as a non-root user (USER, file ownership, directory permissions), enable a read-only filesystem with tmpfs for directories that need writes, and drop all Linux capabilities then add back only what's needed. Explain what commonly breaks with each hardening step (port binding below 1024, file permissions, read-only conflicts with logging/temp files, missing capabilities for network binding) and how to fix each issue.</summary>

**Step 1: Run as non-root user**

```dockerfile
FROM node:20-slim

WORKDIR /app

# Copy and install as root (need write access to /app)
COPY package.json package-lock.json ./
RUN npm ci --omit=dev

COPY dist/ ./dist/

# Create app directory ownership, then switch to non-root
RUN chown -R node:node /app
USER node

EXPOSE 3000
CMD ["node", "dist/main.js"]
```

The `node` user (UID 1000) already exists in the official Node.js images. `USER node` switches all subsequent commands and the runtime process to this user.

**What breaks**: Binding to ports below 1024 (e.g., port 80) requires `CAP_NET_BIND_SERVICE` or root. Fix: use a port above 1024 (e.g., 3000) and let the reverse proxy/load balancer handle port 80.

**What breaks**: Writing to directories owned by root. Fix: `chown` before `USER` switch, or create specific writable directories.

**Step 2: Read-only filesystem with tmpfs**

```yaml
# Docker Compose
services:
  api:
    build: .
    read_only: true
    tmpfs:
      - /tmp              # Node.js and libraries write temp files
      - /app/logs          # if the app writes log files
    security_opt:
      - no-new-privileges:true  # prevent privilege escalation via setuid
```

```bash
# Docker run equivalent
docker run \
  --read-only \
  --tmpfs /tmp \
  --tmpfs /app/logs \
  my-app
```

**What breaks**: Any file write to the filesystem fails with `EROFS: read-only file system`. Common culprits:
- `/tmp` — many libraries write temp files here. Fix: `tmpfs` mount on `/tmp`
- Log files — if your app writes to disk. Fix: `tmpfs` on the log directory, or better: log to stdout (Docker captures it)
- npm cache — `npm` tries to write cache. Not relevant in production (no `npm install` at runtime)
- PID files — some process managers write `.pid` files. Fix: `tmpfs` on the PID directory

**Step 3: Drop all capabilities, add back only what's needed**

```yaml
services:
  api:
    build: .
    cap_drop:
      - ALL               # drop every Linux capability
    cap_add: []            # add nothing back — Node.js web server needs none
    # If binding to port <1024:
    # cap_add:
    #   - NET_BIND_SERVICE
```

```bash
# Docker run equivalent
docker run \
  --cap-drop=ALL \
  my-app
```

**What breaks when you drop all capabilities**:
- `CAP_NET_BIND_SERVICE`: Binding to ports below 1024. Fix: use ports above 1024.
- `CAP_CHOWN`, `CAP_SETUID`, `CAP_SETGID`: Changing file ownership or switching users at runtime. Fix: handle all ownership in the Dockerfile before `USER` switch.
- `CAP_NET_RAW`: Raw socket access (ping, some network diagnostics). Usually not needed by applications. Fix: not needed.

**Complete hardened Compose configuration**:

```yaml
services:
  api:
    build: .
    read_only: true
    tmpfs:
      - /tmp
    cap_drop:
      - ALL
    security_opt:
      - no-new-privileges:true
    user: "1000:1000"     # explicit UID:GID (alternative to USER in Dockerfile)
    ports:
      - "3000:3000"
```

A typical Node.js web server needs zero Linux capabilities, can run entirely on a read-only filesystem with just `/tmp` writable, and works fine as a non-root user on port 3000. Most hardening "breaks" are from running on privileged ports or writing files that should be going to stdout instead.

</details>

## Practical — Debugging & Troubleshooting

<details>
<summary>19. A container exits immediately after starting — walk through the exact debugging steps using docker logs, docker inspect (check exit code and state), and docker run with an interactive shell to reproduce the issue. What are the most common causes of immediate exit (missing entrypoint, failed health check, missing env vars, permission errors) and how do you fix each?</summary>

**Step 1: Check logs**

```bash
docker logs <container>
# Shows stdout/stderr output — often reveals the error directly
# e.g., "Error: Cannot find module '/app/dist/main.js'"
```

**Step 2: Check exit code and state**

```bash
docker inspect <container> --format='{{.State.ExitCode}} {{.State.OOMKilled}} {{.State.Error}}'
```

Exit code meanings:
- **0**: Process exited normally (maybe the command just finished — not a long-running server)
- **1**: Application error (unhandled exception, missing file, bad config)
- **126**: Permission denied (file exists but isn't executable)
- **127**: Command not found (entrypoint binary doesn't exist)
- **137**: SIGKILL (OOM kill or `docker kill`) — check `OOMKilled` field
- **139**: Segfault (native addon crash, wrong architecture)

**Step 3: Get a shell to investigate**

```bash
# Override the entrypoint to get an interactive shell
docker run -it --entrypoint /bin/sh my-image

# Inside the container:
ls -la /app/dist/           # Does the expected file exist?
node dist/main.js           # Try running manually — see the real error
env                         # Are expected env vars set?
whoami                      # Check user context
cat /etc/os-release         # Confirm base image
```

**Common causes and fixes**:

**Missing entrypoint/command (exit 127)**:
```bash
docker inspect my-image --format='{{.Config.Entrypoint}} {{.Config.Cmd}}'
# If both are empty or point to a missing binary → fix Dockerfile
```

**Missing environment variables (exit 1)**:
```bash
docker logs <container>
# "Error: DATABASE_URL is required"
# Fix: add -e DATABASE_URL=... or check env_file in Compose
```

**Permission errors (exit 126 or 1)**:
```bash
docker run -it --entrypoint /bin/sh my-image
ls -la /app/dist/main.js
# -rw-r--r-- root root → running as non-root user can't execute
# Fix: chown in Dockerfile or ensure files are readable by the running user
```

**File not found — build artifact missing (exit 1)**:
The multi-stage build didn't `COPY --from=build` the right path, or `.dockerignore` excluded necessary files. Verify by shell-ing in and checking the filesystem.

**OOM kill at startup (exit 137)**:
Application needs more memory than the limit allows just to start. Common with Node.js when `--max-old-space-size` is larger than the container memory limit. Fix: align Node.js heap limit with container memory limit (leave room for non-heap memory).

**Process exits immediately because it's not a server (exit 0)**:
The CMD runs a script that completes, not a long-running process. Check that the entrypoint actually starts a server that listens on a port.

</details>

<details>
<summary>20. A container cannot reach another container or an external service — walk through the systematic debugging process: check which network the containers are on, verify DNS resolution, test connectivity with curl/wget from inside the container, inspect port mappings, and identify whether the issue is network mode, firewall rules, or application-level binding (0.0.0.0 vs 127.0.0.1). Show the exact commands at each step.</summary>

**Step 1: Check which network both containers are on**

```bash
# List networks for each container
docker inspect api --format='{{json .NetworkSettings.Networks}}' | jq
docker inspect db --format='{{json .NetworkSettings.Networks}}' | jq

# They must be on the same network to communicate
# Docker Compose auto-creates a network named <project>_default
docker network ls
docker network inspect myproject_default
```

If containers are on different networks, they can't reach each other. Fix: put them on the same network or connect the second one:
```bash
docker network connect myproject_default api
```

**Step 2: Verify DNS resolution**

```bash
docker exec api nslookup db
# Or if nslookup isn't available:
docker exec api getent hosts db

# Expected: returns the container's IP on the shared network
# If DNS fails: containers are on the default bridge (no DNS) instead of a user-defined bridge
```

DNS only works on user-defined bridge networks (as covered in question 3). On the default `bridge` network, you must use IP addresses.

**Step 3: Test connectivity from inside the container**

```bash
# Test TCP connectivity to the target port
docker exec api sh -c "curl -v http://db:5432" 2>&1 | head -20

# If curl isn't available, use wget or raw TCP test:
docker exec api sh -c "wget -qO- --timeout=3 http://db:5432"
docker exec api sh -c "echo > /dev/tcp/db/5432 && echo 'open' || echo 'closed'"

# For external services:
docker exec api curl -v https://api.example.com
```

**Step 4: Check port mappings and binding address**

```bash
# Check what ports the target container exposes
docker inspect db --format='{{json .Config.ExposedPorts}}'
docker port db

# CRITICAL: Check what address the service binds to INSIDE the container
docker exec db ss -tlnp
# or: docker exec db netstat -tlnp
```

**The 0.0.0.0 vs 127.0.0.1 problem**: If the target service binds to `127.0.0.1` (localhost only), other containers can't reach it even on the same network. The service must bind to `0.0.0.0` to accept connections from other containers.

```bash
# BAD: app.listen(3000, '127.0.0.1') → only reachable from inside same container
# GOOD: app.listen(3000, '0.0.0.0') → reachable from other containers
```

This is the most common cause of "containers are on the same network but still can't connect."

**Step 5: Check for firewall or proxy issues**

```bash
# Test if the host can reach external services
docker exec api ping -c 3 8.8.8.8          # raw IP — tests basic connectivity
docker exec api nslookup google.com          # tests DNS resolution

# If ping works but DNS fails → DNS config issue
docker exec api cat /etc/resolv.conf
```

**Common causes summary**:

| Symptom | Likely cause | Fix |
|---|---|---|
| DNS lookup fails | Containers on different networks or default bridge | Use same user-defined network |
| TCP connection refused | Service binds to 127.0.0.1 | Bind to 0.0.0.0 |
| TCP connection timeout | Different networks or firewall | Check `docker network inspect` |
| Can reach containers but not internet | Docker DNS/network config | Check `resolv.conf`, restart Docker daemon |

</details>

<details>
<summary>21. You need to diagnose a running container that is behaving unexpectedly — show how to use docker exec to get a shell, inspect running processes, check file system state, view environment variables, test network connectivity, and read application logs. What do you do when the container image has no shell or debugging tools installed (distroless/scratch)?</summary>

**Get a shell**:

```bash
docker exec -it <container> /bin/sh
# or /bin/bash if available
```

**Inspect running processes**:

```bash
docker exec <container> ps aux
# or from the host:
docker top <container>
```

Look for unexpected processes, high CPU/memory usage, or zombie processes (`<defunct>`).

**Check filesystem state**:

```bash
docker exec <container> ls -la /app/dist/
docker exec <container> cat /app/config/settings.json
docker exec <container> df -h          # disk usage
docker exec <container> du -sh /app/*  # directory sizes
```

**View environment variables**:

```bash
docker exec <container> env
# or from outside:
docker inspect <container> --format='{{json .Config.Env}}' | jq
```

**Test network connectivity** (same approach as question 20, but from inside):

```bash
docker exec <container> curl -v http://db:5432
docker exec <container> nslookup db
docker exec <container> ss -tlnp        # what ports is this container listening on
```

**Read application logs**:

```bash
docker logs <container>                  # all logs
docker logs <container> --tail 100       # last 100 lines
docker logs <container> --since 5m       # last 5 minutes
docker logs <container> -f               # follow (live tail)

# If the app writes to files instead of stdout:
docker exec <container> tail -f /app/logs/app.log
```

**When there's no shell (distroless/scratch)**:

You cannot `docker exec` into distroless or scratch images because there's no shell binary.

**Option 1: Use the distroless debug variant** (has BusyBox shell):
```bash
# Rebuild with debug variant for troubleshooting
FROM gcr.io/distroless/nodejs20-debian12:debug
# Then: docker exec -it <container> /busybox/sh
```

**Option 2: Kubernetes ephemeral debug containers** (no rebuild needed):
```bash
kubectl debug -it <pod> --image=busybox --target=<container>
# Attaches a debug container sharing the target's process namespace
# You can see processes, network, and filesystem of the target container
```

**Option 3: Copy files out for inspection**:
```bash
docker cp <container>:/app/dist/ ./debug-output/
```

**Option 4: Inspect from the host** (Docker-specific):
```bash
# docker logs still works — no shell needed
docker logs <container>

# docker inspect still works
docker inspect <container>

# Check resource usage
docker stats <container> --no-stream

# Find the container's filesystem on the host (requires root)
docker inspect <container> --format='{{.GraphDriver.Data.MergedDir}}'
# Then browse it directly on the host
```

**Option 5: nsenter from the host** (advanced — enter container's namespaces):
```bash
PID=$(docker inspect <container> --format='{{.State.Pid}}')
nsenter -t $PID -n ss -tlnp   # enter network namespace, check listening ports
nsenter -t $PID -m ls /app     # enter mount namespace, check filesystem
```

</details>

---

## Experience-Based Questions

These questions test real-world experience. Prepare by mapping them to your own projects and situations.

<details>
<summary>22. Tell me about a time you significantly reduced Docker image size or build time — what was the starting point, what specific changes did you make (multi-stage builds, base image switch, layer reordering, .dockerignore), and what was the measurable impact on CI pipeline speed or deployment?</summary>

**What the interviewer is looking for**: Systematic approach to optimization, understanding of why each change works (layers, cache, base images), and measurable results — not just "I made it smaller."

**Structure**: Problem (with numbers) -> Investigation -> Changes (with reasoning) -> Results (with numbers)

**Key points to hit**:
- Start with concrete numbers: "The image was 1.2 GB and builds took 8 minutes in CI"
- Show you understood the root cause, not just applied a checklist
- Explain WHY each change helped (demonstrates understanding of the layer model covered in question 2)
- End with measurable impact on the team/pipeline, not just image size

**Example outline to personalize**:

1. **Context**: "Our Node.js API had a 1.2 GB Docker image. CI builds took 8 minutes, and deployments were slow because image pulls across regions took 45+ seconds."

2. **Investigation**: "I ran `docker history` to see layer sizes. The biggest culprits were: full `node:20` base image (~900 MB), devDependencies shipped in production, and no `.dockerignore` so `.git` and `node_modules` were in the build context."

3. **Changes made**:
   - Switched from `node:20` to `node:20-slim` (cut ~700 MB of unused OS packages)
   - Added multi-stage build — devDependencies and TypeScript compiler only in build stage (as demonstrated in question 11)
   - Reordered layers: `COPY package*.json` before `COPY . .` so dependency install is cached (as covered in question 10)
   - Added `.dockerignore` to exclude `.git`, `node_modules`, `dist`, test files
   - Removed source maps from production image

4. **Results**: "Image went from 1.2 GB to 180 MB. CI builds dropped from 8 minutes to 2 minutes (cache hits on dependency layer). Deployment image pulls went from 45 seconds to under 10 seconds. The team adopted the pattern across all services."

**Avoid**: Listing changes without explaining why they work, or giving vague results ("it was much faster").

</details>

<details>
<summary>23. Describe a time you debugged a container issue in production or staging — what were the symptoms, how did you diagnose the root cause (logs, exec, inspect, resource limits), and what did you change to prevent it from recurring?</summary>

**What the interviewer is looking for**: Systematic debugging process (not guessing), understanding of container-specific failure modes (OOM, signal handling, networking), and preventive thinking.

**Structure**: Symptom -> Diagnosis steps (show your process) -> Root cause -> Fix -> Prevention

**Key points to hit**:
- Show a logical debugging sequence, not "I Googled it"
- Mention specific Docker commands you used (`docker logs`, `docker inspect`, `docker stats`, `docker exec`)
- Explain why the root cause was container-specific (not just an app bug)
- End with what you did to prevent recurrence (monitoring, limits, health checks)

**Example outline to personalize**:

1. **Symptom**: "Our API containers in staging were restarting every few hours. No error in application logs — just sudden restarts with no graceful shutdown message."

2. **Diagnosis**:
   - Checked `docker logs` — no crash message, logs just stopped mid-request
   - Checked `docker inspect` — exit code 137, `OOMKilled: true` (as covered in question 9)
   - Checked `docker stats` — memory usage climbing steadily toward the 512 MB limit
   - Used `docker exec` to take a heap snapshot with `node --inspect` — found a growing cache object that was never evicted

3. **Root cause**: "An in-memory cache had no TTL or size limit. Under production-like load, it grew until it hit the container memory limit, triggering an OOM kill. The container restarted, cache was empty, and the cycle repeated."

4. **Fix**: "Added a bounded LRU cache with TTL eviction. Also increased the memory limit from 512 MB to 768 MB to give headroom for legitimate peak usage."

5. **Prevention**: "Added memory usage alerts at 80% of the container limit. Added the `OOMKilled` field to our container monitoring dashboard. Documented memory limit sizing guidelines for the team."

**Alternative scenarios you could use**: PID 1 signal handling issue (container takes 10 seconds to stop, dropping requests), networking issue (service can't reach database after Compose network change), CPU throttling causing timeout cascades.

</details>

<details>
<summary>24. Tell me about a time you set up or improved a containerized local development environment for your team — what problems existed before (slow rebuilds, environment drift, flaky service dependencies), what did you implement, and how did it affect developer productivity?</summary>

**What the interviewer is looking for**: Understanding of developer experience (DX) as an engineering concern, practical Docker Compose skills, and ability to identify and fix friction points that affect the whole team.

**Structure**: Problem (pain points with impact) -> What you built -> Technical details -> Team impact

**Key points to hit**:
- Quantify the before-state pain ("onboarding took 2 days", "builds took 10 minutes")
- Show specific Compose techniques: bind mounts for hot-reload, health checks for dependency ordering, volume tricks for node_modules (as covered in question 15)
- Mention what you standardized (env vars, service versions, seed data)
- End with team impact — not just "it was faster" but how it changed workflows

**Example outline to personalize**:

1. **Problem**: "New developers spent 1-2 days setting up the local environment. Everyone had different PostgreSQL and Redis versions. 'Works on my machine' was a weekly occurrence. Running the full stack locally required installing 5 tools and following a 3-page Confluence doc."

2. **What I implemented**:
   - Docker Compose for the full local stack: API, PostgreSQL, Redis, and a seed-data service
   - Bind mounts with hot-reload so code changes reflected instantly (no rebuild needed)
   - Anonymous volume for `node_modules` to avoid host/container conflicts (as covered in question 15)
   - Health checks with `depends_on: condition: service_healthy` so services start in the right order (as covered in question 16)
   - Seed data service that runs migrations and loads fixtures on `docker compose up`
   - `.env.example` file committed to the repo with sensible defaults

3. **Technical decisions**:
   - Pinned exact image versions (`postgres:16.2`, not `postgres:latest`) to prevent environment drift
   - Separate `docker-compose.test.yml` for integration tests in CI (as covered in question 17)
   - Added `make dev` and `make test` as simple entry points

4. **Impact**: "Onboarding went from 1-2 days to `git clone && make dev` in 5 minutes. Environment drift bugs disappeared. The team could spin up the full stack with one command, and everyone ran the same versions of everything."

**Avoid**: Focusing only on the Docker config without explaining the team problem it solved.

</details>

<details>
<summary>25. Describe a time you dealt with a container security concern — what was the vulnerability or misconfiguration (running as root, exposed secrets, unscanned images, excessive capabilities), how did you discover it, and what remediation steps did you take?</summary>

**What the interviewer is looking for**: Security awareness beyond "we scanned for CVEs," understanding of container-specific attack vectors (as covered in question 6), and ability to drive remediation across a team — not just fix one container.

**Structure**: Discovery -> Risk assessment -> Remediation -> Systemic prevention

**Key points to hit**:
- How you discovered it (audit, scanning tool, incident, code review)
- Why it was a real risk, not just a theoretical one
- Specific technical remediation steps (not just "we fixed it")
- What you put in place to prevent recurrence (CI checks, base image standards, policy)

**Example outline to personalize**:

1. **Discovery**: "During a security audit, I found that all 12 of our production containers were running as root. Several also had the Docker socket mounted for a legacy deployment script, and none had capability restrictions."

2. **Risk assessment**: "Running as root means any RCE vulnerability gives the attacker root inside the container. Combined with the mounted Docker socket, an attacker could create a privileged container and escape to the host entirely (as covered in question 6). This was a critical risk."

3. **Remediation**:
   - Added `USER node` to all Dockerfiles with proper `chown` for app directories
   - Removed Docker socket mounts — replaced the legacy deployment script with a CI/CD pipeline
   - Added `cap_drop: ALL` to all Compose and Kubernetes deployments (as covered in question 18)
   - Enabled read-only filesystem with `tmpfs` for `/tmp`
   - Added `security_opt: no-new-privileges:true`
   - Integrated Trivy image scanning into CI — builds fail on critical/high CVEs
   - Switched base images from `node:20` to `node:20-slim` to reduce CVE surface area

4. **Systemic prevention**:
   - Created a hardened base Dockerfile template that all new services must use
   - Added a CI step that checks for `USER` directive in Dockerfiles (fails if running as root)
   - Set up weekly Trivy scans on deployed images to catch newly disclosed CVEs
   - Documented the security baseline in the team's engineering handbook

**Alternative scenarios**: Secrets leaked in image layers (ENV with API keys visible in `docker history`), unpatched base image with a critical CVE found by scanning, or a container escape attempt caught in monitoring logs.

</details>
