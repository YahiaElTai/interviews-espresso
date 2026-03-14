# Operating Systems & Linux

> **23 questions** — 15 theory, 8 practical

- Processes vs threads: memory isolation, fault boundaries, multi-process vs multi-thread tradeoffs, how Node.js worker threads and Go goroutines differ
- Process memory: stack vs heap, stack overflow vs heap exhaustion, Node.js --max-old-space-size and V8 memory limits
- Virtual memory: address spaces, page faults, swap, OOM kills
- File descriptors: Unix "everything is a file" model, socket/pipe/device abstraction, FD exhaustion
- Standard streams: stdout, stderr, stdin, redirection, how containers capture stdout for logging, log rotation basics
- I/O models: blocking vs non-blocking, I/O multiplexing (select/poll/epoll), why epoll powers libuv and Node.js
- Inter-process communication: pipes, Unix sockets, shared memory, signals
- Unix signals: SIGTERM, SIGKILL, SIGINT, SIGHUP, SIGUSR1/2, graceful shutdown in containers
- Process management: systemd basics (units, journalctl, service lifecycle), init systems, PID 1 responsibilities, orphan process adoption
- Linux namespaces and cgroups: what the OS provides (PID, network, mount, user isolation), resource limits (CPU, memory), how containers build on these primitives
- File permissions: chmod, chown, setuid/setgid, sticky bit, Docker non-root user issues
- Environment variables: process inheritance, /proc/PID/environ
- Debugging and inspection tools: ps, top/htop, strace, lsof, ss/netstat, vmstat, iostat, df/du, /proc filesystem -- diagnosing FD leaks, zombie processes, D-state, I/O bottlenecks, disk space issues

---

## Foundational

<details>
<summary>1. What is the difference between a process and a thread at the OS level — why do operating systems provide both abstractions instead of just one, how do they differ in memory isolation and fault boundaries, and what are the tradeoffs of multi-process vs multi-thread architectures for a backend service?</summary>

A **process** is an independent execution unit with its own virtual address space, file descriptor table, and signal handlers. A **thread** is an execution unit that shares the address space and resources of its parent process — it gets its own stack and register set, but heap, globals, and FDs are shared with sibling threads.

**Why both exist:** The OS provides both because they solve different problems. Processes give you isolation — one process crashing can't corrupt another's memory. Threads give you concurrency within a shared context — they can communicate via shared memory without serialization overhead, and they're cheaper to create than processes (no need to duplicate page tables).

**Memory isolation and fault boundaries:**

- **Processes** are fully isolated. A segfault in one process doesn't affect others. The OS enforces this via separate virtual address spaces — process A literally cannot address process B's memory.
- **Threads** share everything. A buffer overflow in one thread can corrupt data that another thread is using. An unhandled exception in one thread can bring down the entire process.

**Multi-process vs multi-thread tradeoffs for backend services:**

| Dimension | Multi-process | Multi-thread |
|---|---|---|
| **Isolation** | Strong — crash in one worker doesn't affect others | Weak — one thread can corrupt shared state or crash the process |
| **Communication** | Requires IPC (pipes, sockets, shared memory) — adds latency and complexity | Direct shared memory — fast but requires locks/synchronization |
| **Memory overhead** | Higher — each process has its own address space (though copy-on-write helps) | Lower — threads share the heap |
| **Context switch cost** | More expensive — requires TLB flush | Cheaper — same address space |
| **Scaling model** | Horizontal — easy to spread across CPU cores | Vertical — careful lock management needed |

**In practice:** Node.js uses the cluster module (multi-process) for scaling across cores, giving fault isolation per worker. Nginx uses multi-process with shared memory for the same reason. Languages with true threading (Java, Go) tend toward multi-thread architectures because they have mature synchronization primitives and garbage collectors that handle shared-memory concurrency.

</details>

<details>
<summary>2. Why does Unix treat everything as a file (sockets, pipes, devices) and what does that design decision enable — how do file descriptors work as the unified handle for all I/O, what happens when a process exhausts its file descriptor limit, and what are the typical symptoms of FD exhaustion in a Node.js server?</summary>

**Why "everything is a file":** This is Unix's core design abstraction. By exposing sockets, pipes, devices, and regular files through the same interface (`open`, `read`, `write`, `close`), Unix enables composability. Any tool that reads from a file descriptor can work with any type of I/O source without knowing what's behind it. This is why you can pipe `cat` into `grep` into `sort` — each program just reads from stdin (FD 0) and writes to stdout (FD 1).

**How file descriptors work:** A file descriptor is a small non-negative integer that indexes into a per-process table maintained by the kernel. When you call `open()`, `socket()`, or `pipe()`, the kernel allocates the lowest available FD number and returns it. The kernel's FD table entry points to a file description (in the kernel's open file table) which tracks the actual resource — offset for files, connection state for sockets, buffer for pipes.

Every process starts with three FDs: 0 (stdin), 1 (stdout), 2 (stderr). Subsequent opens get 3, 4, 5, etc.

**What happens when FDs are exhausted:** Every process has a limit on how many FDs it can open (check with `ulimit -n`, typically 1024 soft / 65536 hard). When a process hits this limit, any call to `open()`, `socket()`, `accept()`, or `pipe()` fails with `EMFILE` ("too many open files"). At the system level, there's also a global limit in `/proc/sys/fs/file-max`.

**Symptoms in a Node.js server:**

- **`Error: EMFILE, too many open files`** — the most direct symptom. New incoming connections are refused because `accept()` can't allocate an FD.
- **Database connections fail** — connection pools can't open new sockets.
- **File operations fail** — `fs.readFile`, `fs.createReadStream` throw EMFILE errors.
- **Health checks start failing** — the server appears down even though the process is running.
- **Cascading failures** — once FD exhaustion hits, everything that needs I/O breaks simultaneously, making it look like a much bigger problem.

**Common causes in Node.js:** Not closing database connections, not destroying HTTP response streams, leaked event listeners holding socket references, or opening files in a tight loop without backpressure. The fix is finding and closing the leaked FDs (use `lsof -p <pid>` to see what's open) and raising `ulimit -n` as a short-term mitigation.

</details>

## Conceptual Depth

<details>
<summary>3. How is process memory organized into stack and heap — why does the OS give each thread its own stack but share the heap across threads, what causes a stack overflow vs heap exhaustion, and how does Node.js expose control over V8's heap via --max-old-space-size — what happens when you set it too low or too high?</summary>

**Stack vs heap organization:**

The **stack** is a contiguous block of memory that grows and shrinks automatically as functions are called and return. It stores local variables, function arguments, and return addresses. Access is LIFO — allocation and deallocation are just pointer adjustments, making it extremely fast.

The **heap** is a large pool of memory for dynamic allocation. When you create objects, arrays, closures, or anything with a lifetime beyond the current function call, it goes on the heap. Allocation requires finding a free block (more complex), and deallocation is handled by the garbage collector in managed languages.

**Why each thread gets its own stack but shares the heap:**

Each thread needs its own stack because threads execute different call chains simultaneously — thread A might be 10 functions deep in an HTTP handler while thread B is 3 functions deep in a database query. If they shared a stack, their frames would interleave and corrupt each other. The stack is inherently tied to execution flow.

The heap is shared because that's the whole point of threads — they need access to common data structures without serialization. If each thread had its own heap, you'd lose the main advantage over multi-process and need IPC for every shared object.

**Stack overflow vs heap exhaustion:**

- **Stack overflow**: The stack has a fixed size (typically 1-8 MB per thread). Deep or infinite recursion pushes frames until the stack guard page is hit, triggering a `SIGSEGV`. In Node.js this appears as `RangeError: Maximum call stack size exceeded`.
- **Heap exhaustion**: The heap can grow until the OS or runtime limit is hit. In Node.js/V8, when the garbage collector can't reclaim enough memory and the heap hits its limit, you get `FATAL ERROR: CALL_AND_RETRY_LAST Allocation failed - JavaScript heap out of memory`.

**Node.js `--max-old-space-size`:**

V8 divides its heap into "young generation" (short-lived objects, small) and "old generation" (long-lived objects, the bulk of heap). `--max-old-space-size` controls the old generation limit in MB:

```bash
# Set heap limit to 4GB
node --max-old-space-size=4096 server.js
```

**Too low** (e.g., 256 MB for a service processing large payloads): The GC runs constantly trying to free memory, causing high CPU usage and long GC pauses. Eventually the process OOMs and crashes. You'll see GC taking 90%+ of CPU time in the logs before the crash.

**Too high** (e.g., 16 GB on a machine with 8 GB RAM): V8 happily allocates until the OS intervenes. The process starts swapping (massive latency spikes), and eventually the Linux OOM killer terminates it — with no graceful shutdown, no cleanup. Worse, in a container with a memory cgroup limit, the OOM kill happens at the cgroup boundary, which can be confusing because the container metrics show "plenty" of system memory.

**Practical guidance:** Set `--max-old-space-size` to roughly 75% of the memory available to the process (accounting for cgroup limits in containers), leaving room for the stack, native memory (buffers, C++ objects), and OS overhead.

</details>

<details>
<summary>4. What is virtual memory and why does every process get its own address space instead of directly accessing physical RAM — how do page faults work (minor vs major), what role does swap play, and how does the Linux OOM killer decide which process to terminate when memory is exhausted?</summary>

**Virtual memory** is an abstraction layer between what a process sees (virtual addresses) and what actually exists (physical RAM). Every process gets its own virtual address space — a flat range of addresses (0 to 2^48 on x86-64) that the process believes is its private memory. The CPU's Memory Management Unit (MMU) translates virtual addresses to physical addresses using page tables maintained by the kernel.

**Why this design instead of direct RAM access:**

1. **Isolation** — Process A can't read or write process B's memory because their virtual addresses map to different physical frames. This is the foundation of process security.
2. **Abstraction** — The process doesn't need to know where in physical RAM it lives, or that its memory might not be in RAM at all (swap).
3. **Overcommit** — The OS can promise more memory than physically exists because not every allocated page needs to be resident simultaneously. Most processes allocate much more than they touch.
4. **Shared memory efficiency** — Multiple processes can map the same physical page (shared libraries, copy-on-write after fork) while each believing they have their own copy.

**Page faults (minor vs major):**

Memory is managed in pages (4 KB typically). A page fault occurs when a process accesses a virtual address whose page isn't in the process's page table or isn't resident in RAM.

- **Minor page fault**: The page is in RAM but not yet mapped to the process's page table. Common after `mmap` or `malloc` — the kernel allocates the physical page on first access (demand paging). No disk I/O, just a page table update. Fast.
- **Major page fault**: The page is not in RAM — it must be read from disk (swap or a file-backed mapping). This involves disk I/O and is orders of magnitude slower (microseconds vs milliseconds).

**Swap's role:** Swap is disk space used as overflow for RAM. When physical memory is under pressure, the kernel evicts infrequently-used pages to swap, freeing RAM for active pages. This keeps the system running under memory pressure but at a massive performance cost — disk I/O is ~1000x slower than RAM. For latency-sensitive backend services, swapping is effectively a failure mode. Many production setups disable swap entirely (Kubernetes recommends this).

**The OOM killer:**

When the system is truly out of memory (RAM exhausted, swap full or disabled, no pages can be reclaimed), the kernel's OOM killer selects a process to terminate. It uses the `oom_score` (0-1000) for each process, calculated from:

- **Memory usage** — The more memory a process uses relative to total, the higher its score (primary factor).
- **oom_score_adj** — A tunable value (-1000 to 1000) that biases the score. Kubernetes sets this for pods based on QoS class: Guaranteed pods get -997 (almost never killed), BestEffort pods get 1000 (killed first).
- **Process age and role** — Kernel threads and PID 1 are protected.

The process with the highest `oom_score` is sent `SIGKILL` (uncatchable — no graceful shutdown). You can check scores at `/proc/<pid>/oom_score` and adjust via `/proc/<pid>/oom_score_adj`.

</details>

<details>
<summary>5. What are the different I/O models available on Linux (blocking, non-blocking, multiplexing with select/poll/epoll, and async I/O) — why did the kernel evolve from select to poll to epoll, what fundamental scalability problem does each one solve that its predecessor couldn't, and why is epoll the mechanism that powers libuv and therefore Node.js?</summary>

**The I/O models:**

**Blocking I/O** — The default. A `read()` call blocks the thread until data arrives. Simple to program but means one thread per connection. At 10,000 concurrent connections, you need 10,000 threads — each consuming ~1-8 MB of stack memory, and context-switching between them becomes a bottleneck (the "C10K problem").

**Non-blocking I/O** — Set `O_NONBLOCK` on a file descriptor. `read()` returns immediately with `EAGAIN` if no data is available. But now you need to poll repeatedly (busy-wait), wasting CPU cycles checking FDs that aren't ready.

**I/O multiplexing** — The solution: ask the kernel "which of these FDs are ready?" and only operate on those. This is where `select`, `poll`, and `epoll` come in.

**The evolution:**

**`select` (1983):**
```c
select(nfds, &readfds, &writefds, &exceptfds, &timeout);
```
- You pass a bitmask of FDs you're interested in. The kernel scans all of them and returns which are ready.
- **Problem 1**: Hard limit of 1024 FDs (FD_SETSIZE). Can't handle more concurrent connections than that.
- **Problem 2**: O(n) on every call — the kernel scans all FDs even if only one is ready. And you have to rebuild the bitmask after every call.

**`poll` (1986):**
```c
poll(fds_array, nfds, timeout);
```
- Uses an array of `pollfd` structs instead of a bitmask. No hard FD limit.
- **Solves**: The 1024 FD limit.
- **Still broken**: O(n) per call — the kernel still scans the entire array on every invocation. At 50,000 connections, you're copying and scanning 50,000 entries every time one socket has data.

**`epoll` (Linux 2.6, 2002):**
```c
epoll_create();          // create an epoll instance
epoll_ctl(epfd, ADD, fd, &event);  // register interest in an FD
epoll_wait(epfd, events, maxevents, timeout);  // get ready FDs
```
- **Solves the fundamental O(n) problem.** You register FDs once with `epoll_ctl`. The kernel maintains an internal data structure (red-black tree + ready list). When an FD becomes ready, the kernel adds it to the ready list via a callback. `epoll_wait` returns only the ready FDs — O(1) per ready FD, not O(n) per total FD.
- **No redundant copying**: The FD set lives in kernel space; you don't rebuild it on every call.
- **Scales to millions of connections** with the same efficiency.

**Why epoll powers libuv and Node.js:**

Node.js uses libuv as its event loop. libuv uses the best available I/O multiplexer for each platform: `epoll` on Linux, `kqueue` on macOS/BSD, `IOCP` on Windows. The event loop calls `epoll_wait` to block until one or more FDs are ready, then dispatches callbacks for each ready FD. This is how a single Node.js thread can handle tens of thousands of concurrent connections — it never blocks waiting on any individual connection; it waits on all of them simultaneously via epoll.

**io_uring (2019, Linux 5.1):** The newest evolution — true async I/O via a shared ring buffer between userspace and kernel. Avoids even the system call overhead of `epoll_wait`. Still maturing for general use but already adopted for high-performance storage workloads.

</details>

<details>
<summary>6. What are the main inter-process communication mechanisms on Linux (pipes, Unix domain sockets, shared memory, signals) — what are the tradeoffs of each in terms of performance, complexity, and use cases, and when would you choose one over another for communication between backend services on the same host?</summary>

| Mechanism | Direction | Performance | Complexity | Best For |
|---|---|---|---|---|
| **Pipes** | Unidirectional (named pipes bidirectional in theory but tricky) | Moderate — kernel buffer copy | Low | Parent-child process communication, shell pipelines |
| **Unix domain sockets** | Bidirectional | High — ~2x throughput of TCP loopback, no network stack overhead | Moderate | Same-host service-to-service communication |
| **Shared memory** | Bidirectional | Highest — no kernel copy, direct memory access | High — needs synchronization (semaphores, mutexes) | High-throughput, low-latency data sharing (databases, caches) |
| **Signals** | Unidirectional, no data | N/A (notification only) | Low but fragile | Process lifecycle events (SIGTERM, SIGHUP reload) |

**Pipes:**
```bash
# Anonymous pipe — shell example
cat access.log | grep "500" | wc -l
```
Created with `pipe()` system call. Data flows one way through a kernel buffer (64 KB default on Linux). Anonymous pipes work only between related processes (parent-child after fork). Named pipes (FIFOs) exist on the filesystem and can connect unrelated processes. Simple but limited — unidirectional, no message boundaries (byte stream), and the writer blocks if the buffer is full.

**Unix domain sockets:**
```bash
# Server listens on a socket file
/var/run/myservice.sock
```
Use the socket API (`socket`, `bind`, `listen`, `accept`) with `AF_UNIX` instead of `AF_INET`. Support both stream (SOCK_STREAM) and datagram (SOCK_DGRAM) modes. Much faster than TCP loopback because they skip the entire network stack — no TCP handshake, no checksums, no routing. They also support passing file descriptors between processes (ancillary data), which is how Nginx's master process shares listening sockets with workers.

**Shared memory:**
```c
// POSIX shared memory
int fd = shm_open("/my-shm", O_CREAT | O_RDWR, 0666);
ftruncate(fd, 4096);
void *addr = mmap(NULL, 4096, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
```
Zero-copy — both processes read/write the same physical pages. But you must synchronize access yourself (semaphores, spinlocks). One bug and you corrupt data in both processes. Used by PostgreSQL for shared buffers, Redis for persistence forks (copy-on-write).

**Signals:**
Can only carry a signal number — no payload (except with `sigqueue` and a single integer). Non-queueable by default (if a signal arrives while the same signal is blocked, it's lost). Useful only for simple notifications: "shut down," "reload config," "dump debug info."

**When to choose what for same-host backend services:**

- **Two services need request/response communication** → Unix domain sockets. They're the standard choice — bidirectional, reliable, fast, and you can use existing protocols (HTTP, gRPC) over them.
- **High-throughput data pipeline between processes** → Shared memory if you need maximum throughput and can handle the synchronization complexity. Otherwise, Unix sockets are fast enough.
- **Simple parent-child data flow** → Pipes. Node.js `child_process.spawn` uses pipes for stdin/stdout/stderr by default.
- **"Hey, reload your config"** → Signals (SIGHUP is conventional for this).

</details>

<details>
<summary>7. How do Unix signals work and why does it matter that SIGKILL and SIGTERM are fundamentally different — what does each common signal (SIGTERM, SIGKILL, SIGINT, SIGHUP, SIGUSR1/2) mean, which ones can be caught and which cannot, and how should a Node.js process running in a container handle SIGTERM to achieve graceful shutdown — what goes wrong if it doesn't?</summary>

**How signals work:** A signal is an asynchronous notification sent to a process by the kernel, another process, or itself. When a signal arrives, the kernel interrupts the process's normal execution and either runs a registered signal handler or takes the default action (terminate, ignore, stop, or core dump). Signals are identified by number (1-31 for standard signals).

**Common signals:**

| Signal | Number | Default | Catchable? | Purpose |
|---|---|---|---|---|
| **SIGTERM** | 15 | Terminate | Yes | "Please shut down gracefully." The polite ask. |
| **SIGKILL** | 9 | Terminate | **No** | "Die immediately." Kernel terminates the process — no handler, no cleanup, no finally blocks. |
| **SIGINT** | 2 | Terminate | Yes | Interrupt from terminal (Ctrl+C). |
| **SIGHUP** | 1 | Terminate | Yes | Terminal hangup. Conventionally used to signal "reload config." |
| **SIGUSR1** | 10 | Terminate | Yes | User-defined. Node.js uses it to start the debugger by default. |
| **SIGUSR2** | 12 | Terminate | Yes | User-defined. Often used for custom diagnostics (heap dump, etc.). |
| **SIGSTOP** | 19 | Stop | **No** | Pause the process (like Ctrl+Z's SIGTSTP but uncatchable). |

**Why SIGTERM vs SIGKILL matters fundamentally:**

SIGTERM runs your handler — you can finish in-flight requests, close database connections, flush logs, and exit cleanly. SIGKILL bypasses everything — the kernel removes the process from the scheduler. No handler runs. No `finally` blocks. No cleanup. File locks aren't released gracefully. Transactions are left half-written.

This is critical in containers: Kubernetes sends SIGTERM first, waits `terminationGracePeriodSeconds` (default 30s), then sends SIGKILL. If your process doesn't handle SIGTERM, it sits there for 30 seconds doing nothing, then gets SIGKILL'd anyway — with no graceful shutdown.

**Graceful shutdown in Node.js:**

```typescript
import { createServer } from 'http';

const server = createServer(/* ... */);

function gracefulShutdown(signal: string) {
  console.log(`Received ${signal}, shutting down gracefully...`);

  // 1. Stop accepting new connections
  server.close(() => {
    console.log('HTTP server closed');

    // 2. Close database connections, flush buffers
    // await db.end();
    // await cache.quit();

    // 3. Exit cleanly
    process.exit(0);
  });

  // 4. Force exit if graceful shutdown takes too long
  setTimeout(() => {
    console.error('Graceful shutdown timed out, forcing exit');
    process.exit(1);
  }, 25_000); // Leave buffer before K8s sends SIGKILL at 30s
}

process.on('SIGTERM', () => gracefulShutdown('SIGTERM'));
process.on('SIGINT', () => gracefulShutdown('SIGINT'));
```

**What goes wrong without handling SIGTERM:**

1. **In-flight requests are dropped** — Clients get connection resets instead of responses.
2. **Database connections are abandoned** — Connection pools on the DB side see stale connections and may hit limits.
3. **Transactions are left uncommitted** — Data inconsistency.
4. **Rolling deployments are disruptive** — Every deploy drops traffic for 30 seconds while Kubernetes waits before SIGKILL.
5. **PID 1 problem** — If Node.js runs as PID 1 in a container (no init system), it doesn't have default signal handlers and SIGTERM is literally ignored. The process sits there until SIGKILL. Fix with `tini` or `dumb-init` as the entrypoint, or `docker --init`.

</details>

<details>
<summary>8. How do Linux namespaces provide process isolation -- what does each namespace type (PID, network, mount, user, UTS) isolate, how do they compose to create what we call a "container," and why is this isolation fundamentally weaker than a VM -- what can a container escape look like?</summary>

**Namespaces** are a Linux kernel feature that partitions kernel resources so that one set of processes sees one set of resources and another set of processes sees a different set. Each namespace type isolates a specific resource:

| Namespace | What it isolates | Container effect |
|---|---|---|
| **PID** | Process ID number space | Container sees its own PID tree; the entrypoint is PID 1 inside but a regular PID outside |
| **Network** | Network interfaces, routing tables, ports, iptables rules | Container gets its own network stack, IP address, and port space |
| **Mount** | Filesystem mount points | Container has its own filesystem tree (overlay FS), can't see host mounts |
| **UTS** | Hostname and domain name | Container can have its own hostname |
| **User** | User/group IDs | Root (UID 0) inside the container can map to a non-root UID on the host |
| **IPC** | System V IPC, POSIX message queues | Shared memory segments are isolated per container |
| **Cgroup** | Cgroup root directory visibility | Container can only see its own cgroup hierarchy |

**How they compose into a "container":**

A container is fundamentally just a process running with a set of namespaces and cgroup limits. When Docker (via containerd/runc) starts a container, it:

1. Creates new namespaces for PID, network, mount, UTS, IPC, and user
2. Sets up an overlay filesystem (mount namespace)
3. Configures a virtual network interface (network namespace)
4. Applies cgroup limits for CPU, memory, I/O
5. Drops Linux capabilities (restricting what root can do)
6. Calls `exec` to start the container's entrypoint

There's no hypervisor, no separate kernel. It's a regular Linux process with resource restrictions.

**Why this is fundamentally weaker than a VM:**

A VM runs a complete separate kernel on virtualized hardware. The attack surface between a VM guest and the host is the hypervisor (small, heavily audited). A container shares the host kernel — every system call goes to the same kernel. The attack surface is the entire system call interface (~400 syscalls on Linux).

**What container escape looks like:**

- **Kernel exploit** — A vulnerability in the shared kernel can be exploited from inside the container to gain host access. CVE-2022-0185 (heap overflow in filesystem context) allowed container escape.
- **Privileged containers** — Running with `--privileged` disables most namespace isolation and gives access to host devices. Mounting the host's `/` filesystem or Docker socket from inside a container gives full host control.
- **Exposed Docker socket** — Mounting `/var/run/docker.sock` inside a container lets the container create new containers on the host with arbitrary mounts — effectively root on the host.
- **Misconfigured user namespace** — Running as root inside the container without user namespace remapping means you're root on the host if you can escape the mount namespace.
- **`/proc` and `/sys` exposure** — Writable cgroup files via `/sys/fs/cgroup` have been used to escape containers.

**Defense in depth:**
- Use non-root containers
- Enable user namespace remapping
- Apply seccomp profiles to restrict syscalls
- Use AppArmor/SELinux
- Never run `--privileged`
- Never mount the Docker socket
- Keep the host kernel patched

</details>

<details>
<summary>9. How do cgroups enforce resource limits for containers -- what resources can cgroups control (CPU, memory, I/O), how does the OOM killer behave when a container hits its cgroup memory limit vs when the host runs out of memory, and what happens when cgroup limits are set too aggressively?</summary>

**cgroups (control groups)** are a Linux kernel feature that organizes processes into hierarchical groups and applies resource limits to those groups. While namespaces control what a process can *see*, cgroups control what it can *use*.

**Resources cgroups control:**

- **Memory** (`memory` controller): Limit resident memory (RAM), swap, and kernel memory per group. The most commonly configured limit in containers.
- **CPU** (`cpu` controller): CPU shares (relative weight), CPU quota (hard limit — e.g., 0.5 CPU means 50ms per 100ms period), and CPU pinning to specific cores via `cpuset`.
- **Block I/O** (`blkio`/`io` controller): Throttle read/write bytes per second and IOPS per device.
- **PIDs** (`pids` controller): Limit the number of processes in a group — prevents fork bombs.
- **Network** (indirectly via `net_cls`/`net_prio`): Tag packets for tc (traffic control) classification.

**In Docker/Kubernetes, these map to:**
```yaml
# Kubernetes resource limits → cgroup settings
resources:
  requests:
    memory: "256Mi"   # Scheduling hint + OOM score adjustment
    cpu: "250m"        # CPU shares (relative)
  limits:
    memory: "512Mi"   # Hard cgroup memory limit
    cpu: "500m"        # CPU quota (hard ceiling)
```

**OOM killer behavior — cgroup limit vs host exhaustion:**

**Container hits its cgroup memory limit:**
- The kernel's cgroup-aware OOM killer activates *within the cgroup*. It kills a process inside the container — typically the main process, since it usually uses the most memory.
- The host is unaffected. Other containers keep running.
- Kubernetes records this as `OOMKilled` in the pod status. The container restarts per its restart policy.
- This is scoped and predictable — it's the whole point of setting memory limits.

**Host runs out of memory (no per-container limits or limits sum to more than host RAM):**
- The global OOM killer activates, using the scoring mechanism covered in question 4. It can kill any container on the host, or even system processes — unpredictable and cascading.
- This is why Kubernetes requires memory limits — without them, one misbehaving pod can take out unrelated workloads.

**What happens when limits are too aggressive:**

**Memory too low:**
- Constant OOMKills and container restarts. The application never stabilizes.
- Subtle variant: the limit is *just barely* enough during normal load but any spike (burst of requests, GC pause that delays freeing memory) triggers OOMKill. You see intermittent restarts that correlate with traffic.
- The GC in Node.js/V8 may run more frequently trying to stay under the limit, consuming CPU and increasing latency.

**CPU too low:**
- CPU throttling — the process gets paused mid-work when its quota is exhausted for the scheduling period. This shows up as increased latency and tail-latency spikes, not as high CPU usage in metrics (the process is *waiting*, not *running*).
- `cat /sys/fs/cgroup/cpu/cpu.stat` shows `nr_throttled` and `throttled_time`. In Kubernetes, check `container_cpu_cfs_throttled_seconds_total`.
- Common gotcha: Node.js event loop gets throttled during GC, causing everything to stall. A 250m CPU limit on a Node.js service is usually too aggressive.

**I/O too low:**
- Disk operations queue up. Processes go into D-state (uninterruptible sleep). Database queries that hit disk become extremely slow. Logs may not flush.

</details>

<details>
<summary>10. Why do Linux file permissions use the owner/group/other model with read/write/execute bits — what do setuid, setgid, and the sticky bit do and when are they dangerous, and why does running containers as non-root create permission issues with mounted volumes — what are the common fixes?</summary>

**The owner/group/other model:**

Every file has an owner (user), a group, and permissions for three categories: owner (u), group (g), and other (o). Each category gets read (r=4), write (w=2), and execute (x=1) bits.

```bash
-rwxr-xr-- 1 app developers 4096 Mar 13 file.js
#  u=rwx     g=r-x         o=r--
#  owner     group          everyone else
```

This model exists because Unix was designed as a multi-user system. It's simple, lightweight (stored in the inode — just a few bits), and covers the vast majority of access control needs. For finer-grained control, ACLs (Access Control Lists) exist but are rarely needed.

**Special bits:**

**setuid (4xxx):** When set on an executable, the process runs as the *file owner* instead of the user who launched it. `passwd` uses this — it's owned by root with setuid, so any user can run it and it can write to `/etc/shadow`.
```bash
chmod u+s /usr/bin/passwd  # -rwsr-xr-x
```
**Dangerous because:** A setuid-root binary with a vulnerability gives an attacker root access. Any buffer overflow or injection in a setuid binary is a privilege escalation.

**setgid (2xxx):** On an executable, the process runs with the file's group. On a directory, new files inherit the directory's group instead of the creator's primary group — useful for shared project directories.

**Sticky bit (1xxx):** On a directory, only the file owner (or root) can delete files, even if others have write permission on the directory. `/tmp` uses this — everyone can create files but can't delete each other's.
```bash
chmod +t /tmp  # drwxrwxrwt
```

**Container non-root permission issues:**

When you mount a host volume into a container, the files on the host have UIDs/GIDs from the host. Inside the container, the process runs as a different UID. The mismatch causes `EACCES` errors.

Example: Host files are owned by UID 1000. Container runs as UID 1001 (because of `USER 1001` in the Dockerfile). The container process can't write to the mounted volume.

```dockerfile
# This creates the problem
FROM node:20-alpine
RUN adduser -D -u 1001 appuser
USER appuser
# Now the process runs as UID 1001 but volume files might be owned by UID 1000
```

**Common fixes:**

1. **Match UIDs** — Set the container user's UID to match the host file owner:
   ```dockerfile
   RUN adduser -D -u 1000 appuser
   USER appuser
   ```

2. **Init container or entrypoint chown** — Change ownership at startup:
   ```bash
   # entrypoint.sh
   chown -R appuser:appuser /app/data
   exec su-exec appuser "$@"
   ```
   Downside: requires running as root initially, then dropping privileges.

3. **Use a shared group** — Add the container user to a group that has access:
   ```dockerfile
   RUN addgroup -g 1000 sharedgroup && \
       adduser -D -u 1001 -G sharedgroup appuser
   ```

4. **fsGroup in Kubernetes** — Kubernetes can set the group ownership of mounted volumes:
   ```yaml
   securityContext:
     runAsUser: 1001
     fsGroup: 1000  # All mounted volumes get this GID
   ```
   This is the cleanest solution in Kubernetes — the kubelet handles the permission adjustment.

5. **emptyDir or volume with correct permissions** — For ephemeral data, `emptyDir` volumes are created fresh with permissions matching the pod's security context, avoiding the mismatch entirely.

</details>

<details>
<summary>11. How do environment variables work at the OS level -- how does a child process inherit its parent's environment, why can't you modify a parent's environment from a child, and why is /proc/PID/environ a security concern even when you think environment variables are "hidden"?</summary>

**How environment variables work at the OS level:**

Environment variables are key-value string pairs stored in a process's memory. When the kernel starts a process via `execve()`, the environment is passed as the third argument — an array of `KEY=VALUE` strings placed on the process's stack, right after the argument list. The C runtime exposes this as `environ` (or `process.env` in Node.js).

**Child process inheritance:**

When a process calls `fork()`, the child gets a complete copy of the parent's memory — including the environment block. When the child calls `execve()` to run a new program, it passes its environment forward. This is why environment variables "flow down" — a shell sets `DATABASE_URL`, and every process it spawns inherits it.

```typescript
import { spawn } from 'child_process';

// Child inherits all of process.env by default
const child = spawn('node', ['worker.js']);

// Override or extend for the child
const child2 = spawn('node', ['worker.js'], {
  env: { ...process.env, LOG_LEVEL: 'debug' }
});
```

**Why a child can't modify the parent's environment:**

After `fork()`, parent and child have separate copies of memory (copy-on-write). The child's environment is its own copy. Modifying it changes only the child's memory — the parent's address space is untouched. There's no system call to write into another process's environment. This is a fundamental consequence of process isolation (as covered in question 1). It's why running `export FOO=bar` in a subshell doesn't affect the parent shell.

**Why `/proc/PID/environ` is a security concern:**

The Linux `/proc` filesystem exposes each process's initial environment at `/proc/<pid>/environ`. Any user who can read this file can see every environment variable the process was started with — including secrets.

```bash
# Read another process's environment (needs same UID or root)
cat /proc/12345/environ | tr '\0' '\n'
# DATABASE_URL=postgres://user:password@host:5432/db
# API_KEY=sk-live-abc123...
```

The concern: teams put secrets in environment variables thinking they're "hidden" because they're not in config files or source code. But:

- **Any process running as the same UID** can read `/proc/<pid>/environ`. In a container, that's every process in the container.
- **Root on the host** can read any process's environment — including containers (since they're just processes).
- **The environment is set at startup and never updates.** Even if you rotate a secret, `/proc/PID/environ` still shows the original value for the lifetime of the process.
- **Crash dumps and debug tools** often include the environment, leaking secrets into logs or error tracking systems.

**Better alternatives:** Use secret management systems (Vault, AWS Secrets Manager) that inject secrets via files or API calls, or use Kubernetes secrets mounted as files rather than environment variables.

</details>

<details>
<summary>12. How do Node.js worker threads differ from OS-level threads, and how do both compare to Go's goroutine model — why does Node.js use a single-threaded event loop with optional worker threads instead of a thread-per-request model, and what are the practical implications for CPU-bound vs I/O-bound workloads in each model?</summary>

**Node.js worker threads vs OS threads:**

Node.js worker threads *are* real OS threads (pthreads), but with a critical difference: each worker thread gets its own V8 isolate — its own JavaScript heap, GC, and event loop. They do **not** share JavaScript objects in memory. Communication happens via message passing (`postMessage`) which serializes data, or via `SharedArrayBuffer` for raw byte-level sharing.

This is fundamentally different from OS threads in languages like Java or C++, where threads share the same heap and can directly read/write shared objects (with locks for safety).

**Go's goroutine model:**

Goroutines are user-space "green threads" — lightweight (2 KB initial stack, grows dynamically up to 1 GB by default) and multiplexed onto a small pool of OS threads by Go's runtime scheduler (M:N scheduling). You can spawn millions of goroutines. They share memory directly (like OS threads) and synchronize via channels or mutexes.

| Dimension | Node.js worker threads | OS threads (Java/C++) | Go goroutines |
|---|---|---|---|
| **Weight** | Heavy — full V8 isolate per thread (~2-10 MB) | Medium — ~1-8 MB stack each | Light — ~2 KB initial, grows dynamically |
| **Memory sharing** | No shared JS heap; `SharedArrayBuffer` for raw bytes | Full heap sharing | Full heap sharing |
| **Scheduling** | OS-scheduled (1:1) | OS-scheduled (1:1) | Runtime-scheduled (M:N) |
| **Communication** | Message passing (structured clone) | Shared memory + locks | Channels (preferred) or shared memory + locks |
| **Practical concurrency** | ~Cores (4-16 workers) | ~Hundreds to low thousands | Millions |

**Why Node.js uses a single-threaded event loop:**

Node.js was designed for I/O-bound workloads — web servers, API gateways, real-time apps. For I/O-bound work, the event loop + epoll model (as covered in question 5) handles tens of thousands of concurrent connections on a single thread without the complexity of locks, race conditions, or deadlocks. Adding threads would add synchronization complexity for no throughput gain on I/O-bound work.

Worker threads exist for the exception: CPU-bound work (image processing, compression, crypto, heavy computation) that would block the event loop. They're not meant for handling requests — they're for offloading computation.

**Practical implications:**

**I/O-bound workloads (API servers, database queries):**
- Node.js excels — one thread handles thousands of concurrent connections via the event loop. Worker threads add nothing here.
- Go also excels — goroutines block on I/O and the scheduler runs other goroutines on the same OS thread. Feels synchronous but is highly concurrent.
- Java thread-per-request works but is heavier. Virtual threads (Project Loom, Java 21+) bring it closer to Go's model.

**CPU-bound workloads (video encoding, ML inference):**
- Node.js struggles — CPU work blocks the event loop, stalling all requests. Worker threads help but you're limited to core count and each worker is expensive.
- Go handles this naturally — goroutines doing CPU work are scheduled across OS threads automatically.
- Java/C++ threads work well for CPU-bound parallelism with direct memory sharing.

**The practical takeaway for Node.js:** Use the event loop for I/O. Use worker threads (or better, a separate microservice) for CPU-heavy operations. Use the cluster module (multi-process) for scaling across cores on I/O-bound workloads.

</details>

<details>
<summary>13. What are zombie processes and processes stuck in D-state (uninterruptible sleep) — why do zombies exist and why can't you kill them with SIGKILL, what causes D-state and why is it a sign of I/O problems, and what is the impact of each on system health?</summary>

**Zombie processes:**

A zombie is a process that has finished executing but still has an entry in the process table. When a process exits, the kernel keeps its PID and exit status around so the parent can retrieve it via `wait()`/`waitpid()`. Until the parent calls `wait()`, the dead process remains as a zombie (shown as state `Z` in `ps`).

**Why they exist:** The kernel needs to preserve the exit status for the parent. If the kernel cleaned up immediately, the parent would have no way to know whether its child succeeded or failed.

**Why SIGKILL doesn't work on zombies:** The process is already dead — there's no running code to receive or handle a signal. SIGKILL tells the kernel to terminate a process, but a zombie has already terminated. The only way to clear a zombie is for its parent to call `wait()`. If the parent is buggy and never calls `wait()`, you can kill the *parent* — the zombie gets reparented to PID 1 (init/systemd), which will reap it.

**Impact on system health:** Each zombie consumes a PID and a small amount of kernel memory (the process table entry). A few zombies are harmless. Thousands of zombies can exhaust the PID space (`/proc/sys/kernel/pid_max`, typically 32768 or 4194304), preventing new processes from starting — which is a serious problem.

**D-state (uninterruptible sleep):**

A process in D-state is waiting for an I/O operation to complete and **cannot be interrupted** — not even by SIGKILL. The kernel puts processes in this state during critical I/O operations (disk reads, NFS calls) where interrupting mid-operation would corrupt data or leave kernel data structures in an inconsistent state.

**What causes D-state:**
- **Slow or unresponsive disk** — failing drive, saturated I/O queue
- **NFS or network filesystem hangs** — the NFS server is unreachable and the mount is using hard mounts (default)
- **Kernel driver issues** — a device driver waiting for hardware that isn't responding

**Why it signals I/O problems:** A process should be in D-state for microseconds during a normal disk read. If you see processes stuck in D-state for seconds or minutes, something is wrong with the I/O subsystem — the disk is failing, the I/O queue is saturated, or a network filesystem is hung.

**Impact on system health:**
- D-state processes can't be killed, so they accumulate.
- They hold resources (file locks, memory) that can't be reclaimed.
- If enough processes pile up in D-state, the system becomes unresponsive — new processes trying to do any I/O also get stuck.
- High load average with low CPU usage is the classic symptom — load average counts D-state processes, but they're not using CPU.

**Quick diagnostic:**
```bash
# Find zombie processes
ps aux | awk '$8 == "Z"'

# Find D-state processes
ps aux | awk '$8 ~ /D/'

# Or with top — look at the S (state) column
top  # then press 'O' and filter by S=Z or S=D
```

</details>

<details>
<summary>14. Why does the container ecosystem standardize on writing logs to stdout/stderr instead of log files -- how do the three standard streams (stdin, stdout, stderr) work at the OS level, how do container runtimes capture stdout/stderr for log aggregation, and what are the tradeoffs of stdout-based logging vs writing to files including log rotation concerns?</summary>

**The three standard streams at the OS level:**

Every process starts with three open file descriptors (as covered in question 2):
- **stdin (FD 0)** — Input stream. By default connected to the terminal. Processes read user input from here.
- **stdout (FD 1)** — Standard output. Normal program output goes here. Line-buffered when connected to a terminal, fully buffered when piped.
- **stderr (FD 2)** — Error output. Separate from stdout so errors aren't mixed into data pipelines. Always unbuffered or line-buffered.

These are just file descriptors — they can point to a terminal, a file, a pipe, or a socket. Shell redirection (`>`, `2>&1`, `|`) works by manipulating which kernel file description these FDs point to before `exec`.

**How container runtimes capture stdout/stderr:**

When Docker (via containerd) starts a container, the container's PID 1 has its stdout and stderr connected to pipes held by the container runtime. The runtime reads from these pipes and writes to JSON log files on the host:

```
/var/lib/docker/containers/<container-id>/<container-id>-json.log
```

Each line is a JSON object with the stream source, timestamp, and message:
```json
{"log":"Server started on port 3000\n","stream":"stdout","time":"2026-03-13T10:00:00.123Z"}
```

`docker logs` reads from this file. Kubernetes log collection (Fluentd, Fluent Bit, Promtail) also reads these files and ships them to a centralized system.

**Why the ecosystem standardized on stdout/stderr:**

1. **Separation of concerns** — The application just writes output. The infrastructure handles collection, rotation, shipping, and storage. The app doesn't need to know about log files, rotation policies, or aggregation pipelines.
2. **Composability** — Logs flow through the standard Unix pipeline. You can `docker logs | grep ERROR` just like any other Unix tool.
3. **Ephemeral containers** — Container filesystems are ephemeral. Logs written to files inside the container are lost when it restarts. stdout/stderr are captured externally.
4. **The 12-Factor App principle** — "Treat logs as event streams." The app emits events; the environment routes them.

**Tradeoffs — stdout vs file-based logging:**

| Dimension | stdout/stderr | Log files |
|---|---|---|
| **Simplicity** | Just `console.log()` — nothing to configure | Need log library config: paths, rotation, format |
| **Rotation** | Handled by container runtime (`--log-opt max-size=10m`) | Application or `logrotate` must handle it |
| **Performance** | Pipe writes are fast but runtime JSON-wrapping adds overhead | Direct file writes, slightly faster for high throughput |
| **Multi-line logs** | Problematic — stack traces split across JSON lines | Easier to handle within a single file |
| **Disk risk** | Runtime-managed rotation prevents disk fill (if configured) | Misconfigured rotation = disk fills up, service crashes |
| **Structured logging** | Write JSON to stdout; the pipeline preserves structure | Same, but you control the file format directly |

**Log rotation concerns:**

With stdout logging, Docker's default log driver (`json-file`) has no rotation by default — logs grow unbounded until the disk fills. You must configure it:

```json
// /etc/docker/daemon.json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "50m",
    "max-file": "3"
  }
}
```

In Kubernetes, the kubelet rotates container logs when they exceed 10 MB (configurable via `containerLogMaxSize`), keeping up to 5 rotated files by default.

With file-based logging, you manage rotation yourself via `logrotate` or a logging library with built-in rotation (e.g., `pino` with `pino-roll`, or Winston's file transport). The risk is that if rotation fails or isn't configured, logs fill the disk — and a full disk can cause cascading failures across all services on the host (as explored in question 20).

</details>

<details>
<summary>15. Why does PID 1 have special responsibilities in Linux and what goes wrong when a container runs without a proper init process -- what are PID 1's duties (signal forwarding, orphan reaping), how does systemd handle service lifecycle management (units, dependencies, restart policies), and how do you use journalctl to inspect service logs and diagnose startup failures?</summary>

**PID 1's special responsibilities:**

The kernel gives PID 1 two unique duties that no other process has:

1. **Orphan reaping:** When a process exits, its children become orphans — they get reparented to PID 1. PID 1 must call `wait()` on these orphans to clean up their zombie entries in the process table. If PID 1 doesn't reap orphans, zombies accumulate.

2. **Signal handling defaults:** Normal processes have default signal handlers (SIGTERM = terminate). PID 1 does **not** get default handlers for signals it hasn't explicitly registered. If PID 1 hasn't set up a SIGTERM handler, SIGTERM is silently ignored. Only SIGKILL can terminate PID 1 (and doing so causes a kernel panic on the host — inside a container, it stops the container).

**What goes wrong in containers without a proper init:**

If your Dockerfile has `CMD ["node", "server.js"]`, Node.js runs as PID 1. Problems:

- **SIGTERM is ignored** — When running as PID 1, the Linux kernel only delivers signals for which the process has explicitly registered a handler. Node.js doesn't register a SIGTERM handler by default, so SIGTERM is silently ignored. Kubernetes sends SIGTERM, nothing happens, waits 30 seconds, sends SIGKILL. Every deployment has a 30-second delay with no graceful shutdown (as covered in question 7).
- **Zombies accumulate** — If your Node.js app spawns child processes (e.g., `child_process.exec`), it needs to reap them. Node.js does call `wait()` for its direct children, but orphaned grandchildren aren't reaped.

**The fix — use a lightweight init:**
```dockerfile
# Option 1: tini
RUN apk add --no-cache tini
ENTRYPOINT ["tini", "--"]
CMD ["node", "server.js"]

# Option 2: Docker's built-in init
docker run --init myimage

# Option 3: Node.js handles signals explicitly (as shown in question 7)
# Works for SIGTERM but doesn't solve zombie reaping
```

**systemd — service lifecycle management:**

On traditional Linux hosts (not containers), systemd is PID 1. It manages services through **unit files**:

```ini
# /etc/systemd/system/myapp.service
[Unit]
Description=My Node.js Application
After=network.target postgresql.service
Requires=postgresql.service

[Service]
Type=simple
User=appuser
WorkingDirectory=/opt/myapp
ExecStart=/usr/bin/node server.js
Restart=on-failure
RestartSec=5
Environment=NODE_ENV=production

[Install]
WantedBy=multi-user.target
```

Key concepts:
- **Units** — Configuration files describing services, mounts, timers, etc.
- **Dependencies** — `After=` controls ordering; `Requires=` controls hard dependencies (if PostgreSQL fails, stop myapp).
- **Restart policies** — `Restart=on-failure` restarts on non-zero exit; `Restart=always` restarts unconditionally.
- **Service types** — `simple` (systemd tracks the main PID), `forking` (for daemons that fork), `notify` (service signals readiness via `sd_notify`).

Common management commands:
```bash
systemctl start myapp        # Start the service
systemctl stop myapp         # Send SIGTERM, then SIGKILL after TimeoutStopSec
systemctl restart myapp      # Stop + start
systemctl enable myapp       # Start on boot
systemctl status myapp       # Show running state, PID, recent logs
systemctl daemon-reload      # Reload unit files after editing
```

**journalctl — inspecting service logs:**

systemd captures stdout/stderr of managed services and stores them in a structured binary journal.

```bash
# View logs for a specific service
journalctl -u myapp.service

# Follow logs in real time (like tail -f)
journalctl -u myapp.service -f

# Show logs since last boot
journalctl -u myapp.service -b

# Show logs from a time range
journalctl -u myapp.service --since "2026-03-13 09:00" --until "2026-03-13 10:00"

# Show only errors (priority 0-3)
journalctl -u myapp.service -p err

# Show kernel messages (useful for OOM kills)
journalctl -k | grep -i oom

# Diagnose startup failure — show the last 50 lines
journalctl -u myapp.service -n 50 --no-pager
```

**Diagnosing startup failures:** Run `systemctl status myapp` first — it shows the exit code and the last few log lines. Then `journalctl -u myapp.service -b` to see the full startup log from this boot. Common causes: wrong `ExecStart` path, missing environment variables, port already in use, permission denied on files.

</details>

## Practical — Debugging & Inspection

<details>
<summary>16. A server is slow and you suspect CPU or memory issues — walk through the exact steps using ps and top/htop to identify the offending processes, explain what the key columns mean (RES, VIRT, SHR, %CPU, S state), how to spot zombie processes and what to do about them, and how to identify which process is consuming the most resources</summary>

**Step 1: Quick overview with `top`**

```bash
top
```

The header shows system-wide stats:
- **load average: 12.5, 10.2, 8.1** — 1/5/15 min averages. If these exceed your CPU count, the system is overloaded. Compare to `nproc` output.
- **%Cpu(s): 85 us, 5 sy, 0 ni, 8 id, 2 wa** — `us` = user CPU, `sy` = kernel/system, `wa` = I/O wait (see question 19), `id` = idle. High `us` = application CPU usage. High `wa` = I/O bottleneck.
- **MiB Mem: 16384 total, 512 free, 14000 used, 1872 buff/cache** — Low free + low buff/cache = real memory pressure.

**Key columns in the process list:**

| Column | Meaning | What to watch for |
|---|---|---|
| **PID** | Process ID | Need this for further investigation |
| **%CPU** | CPU usage (can exceed 100% on multi-core) | Sort by this (press `P`) to find CPU hogs |
| **%MEM** | Percentage of physical RAM used | Sort by this (press `M`) to find memory hogs |
| **VIRT** | Virtual memory size — everything the process has mapped (code, heap, shared libs, mmap'd files, swap) | Can be huge and misleading — not all of it is in RAM |
| **RES** | Resident Set Size — physical RAM actually being used right now | **This is the number that matters for memory pressure** |
| **SHR** | Shared memory — portion of RES shared with other processes (shared libraries, shared mappings) | RES - SHR = roughly the process's private memory |
| **S** | Process state: `S`=sleeping, `R`=running, `D`=uninterruptible sleep, `Z`=zombie, `T`=stopped | `D` and `Z` are the flags (see question 13) |

**Step 2: Sort and identify the offenders**

```bash
# In top, press:
# P — sort by CPU (find CPU hogs)
# M — sort by memory (find memory hogs)
# 1 — toggle per-CPU view (see if one core is pinned at 100%)
# c — show full command line
```

**Step 3: Detailed view with `ps`**

```bash
# Top 10 CPU consumers
ps aux --sort=-%cpu | head -11

# Top 10 memory consumers
ps aux --sort=-rss | head -11

# Show specific process with thread detail
ps -eLf -p <pid>   # -L shows threads, useful for multi-threaded processes

# Show process tree — see parent-child relationships
ps auxf
```

**Step 4: Spot zombie processes**

```bash
# Find zombies
ps aux | awk '$8 == "Z" { print }'

# Or in top — look for processes with S=Z
# The zombie count is also shown in top's header: "Tasks: 250 total, 3 zombie"
```

**What to do about zombies:** You can't kill a zombie (it's already dead — as covered in question 13). Find its parent:

```bash
# Find parent PID of zombie
ps -o pid,ppid,stat,cmd -p <zombie_pid>

# Then either:
# 1. Fix the parent process (it should be calling wait())
# 2. Kill the parent — orphaned zombie gets reparented to PID 1, which reaps it
kill <parent_pid>
```

**Step 5: Deeper investigation with `htop`**

`htop` is the interactive, color-coded version of `top`. Advantages:
- **Tree view** (F5) — see parent-child process hierarchy
- **Filter** (F4) — filter by process name in real-time
- **Sort** (F6) — choose sort column from a menu
- **Per-thread view** — shows individual threads with their CPU usage (useful for diagnosing if one thread in a multi-threaded app is pinned)
- **Setup** (F2) — add columns like IO_RATE to see per-process disk I/O

```bash
htop -p <pid>   # Monitor a specific process and its threads
```

**Practical pattern for a slow Node.js server:**

1. Run `top`, press `P` — if the Node.js process is at 100% CPU, it's likely stuck in a CPU-bound operation or infinite loop on the event loop.
2. Check RES — if it's close to the container's memory limit, the GC is likely thrashing (high CPU + high memory = GC pressure).
3. If CPU and memory look normal but the server is slow, the bottleneck is probably I/O (database, external API calls) — move to the tools in questions 17-19.

</details>

<details>
<summary>17. A Node.js process is hanging and you don't know why — use strace to attach to the running process and diagnose what system calls it's stuck on. Show the exact strace commands, explain how to interpret the output (identifying blocked reads, slow network calls, or filesystem waits), and what the common patterns look like for different types of hangs</summary>

`strace` intercepts and logs every system call a process makes. When a process is hanging, strace tells you *exactly* what it's waiting on at the kernel level.

**Step 1: Attach to the running process**

```bash
# Attach to PID, follow all threads (-f), show timestamps (-t), and string details (-s 256)
strace -f -t -s 256 -p <pid>

# If you only care about specific syscall categories:
strace -f -e trace=network -p <pid>    # Only network-related syscalls
strace -f -e trace=file -p <pid>       # Only file-related syscalls
strace -f -e trace=read,write -p <pid> # Only read/write

# Show time spent in each syscall (great for finding slow calls)
strace -f -T -p <pid>   # -T shows time spent per syscall in <angle brackets>
```

**Note:** In containers, you may need `--cap-add=SYS_PTRACE` on the container or use `nsenter` from the host. On Kubernetes, use an ephemeral debug container or add the `SYS_PTRACE` capability.

**Step 2: Interpret the output**

Each line shows: `syscall(arguments) = return_value <time_spent>`

```
[pid 12345] 10:15:32 epoll_wait(5, [], 1024, 30000) = 0 <30.001>
```
This means: thread 12345 called `epoll_wait` with a 30-second timeout, got 0 events (nothing happened), and it took 30 seconds. This is the **normal idle state of a Node.js event loop** — it's waiting for I/O events. Don't panic when you see this.

**Common hang patterns:**

**Pattern 1: Blocked on DNS resolution**
```
[pid 12345] connect(15, {sa_family=AF_INET, sin_addr=8.8.8.8, sin_port=53}, 16) = -1 EINPROGRESS
[pid 12345] poll([{fd=15, events=POLLOUT}], 1, 5000) = 0 (Timeout) <5.001>
```
The process is trying to connect to a DNS server and timing out. DNS resolution is blocking. Check `/etc/resolv.conf`, DNS server reachability, and whether the DNS resolver is overloaded.

**Pattern 2: Stuck waiting for a remote service (database, API)**
```
[pid 12345] write(18, "GET /api/data HTTP/1.1\r\n...", 156) = 156 <0.000>
[pid 12345] read(18, <unfinished ...>
```
The process sent a request and is stuck waiting for a response. FD 18 is a socket — use `lsof -p <pid> -a -d 18` to find out which remote host it's connected to. The remote service is likely slow or hung.

**Pattern 3: Filesystem hang (NFS or slow disk)**
```
[pid 12345] open("/mnt/nfs/data/file.json", O_RDONLY) = ... <stuck, no return>
```
The process is stuck in a syscall that hasn't returned. If it's on an NFS mount, the NFS server is likely unreachable. The process is in D-state (as covered in question 13) and can't even be interrupted.

**Pattern 4: Deadlock / event loop blocked by CPU**
```
[pid 12345] futex(0x7f..., FUTEX_WAIT, 0, NULL) = ...
```
Lots of `futex` calls suggest the process is waiting on a lock (mutex/semaphore). In Node.js, this could be a native addon holding a lock. If you see no syscalls at all and the process is at 100% CPU, the event loop is stuck in a synchronous JavaScript computation (infinite loop, huge JSON.parse, synchronous crypto).

**Pattern 5: epoll_wait returning but nothing progressing**
```
[pid 12345] epoll_wait(5, [{EPOLLIN, {fd=12}}], 1024, -1) = 1 <0.050>
[pid 12345] read(12, "...", 65536) = 65536 <0.000>
[pid 12345] epoll_wait(5, [{EPOLLIN, {fd=12}}], 1024, -1) = 1 <0.000>
[pid 12345] read(12, "...", 65536) = 65536 <0.000>
# ... repeating rapidly
```
Data is arriving faster than the process can handle. This isn't a hang — it's a flood. The event loop is busy processing incoming data and never gets to your request handlers.

**Step 3: Get a statistical summary**

```bash
# Run for 30 seconds and get aggregate stats
timeout 30 strace -f -c -p <pid>

# Output:
# % time     seconds  calls  syscall
# ------  ----------  -----  -------
#  85.00    4.250000     12  epoll_wait    ← mostly idle, normal
#  10.00    0.500000   1500  read
#   3.00    0.150000   1200  write
```

The `-c` flag gives you a syscall count and time summary — immediately shows where time is being spent without reading thousands of lines of output.

</details>

<details>
<summary>18. You suspect a Node.js server has a file descriptor leak — use lsof to list open FDs for the process, identify what types of FDs are leaking (sockets, pipes, files), and use ss (or netstat) to examine connection states and identify connections stuck in CLOSE_WAIT. Show the exact commands, explain the output columns, and describe how to correlate lsof and ss output to find the leak source</summary>

**Step 1: Confirm FD count is growing**

```bash
# Count open FDs for a process
ls /proc/<pid>/fd | wc -l

# Or with lsof
lsof -p <pid> | wc -l

# Check the process's FD limit
cat /proc/<pid>/limits | grep "open files"
# Max open files    1024    65536    files
#                   soft    hard
```

Run this periodically. If the count grows steadily without dropping, you have a leak.

**Step 2: Identify what types of FDs are open with `lsof`**

```bash
# All open FDs for the process
lsof -p <pid>

# Key output columns:
# COMMAND  PID  USER  FD    TYPE   DEVICE  SIZE/OFF  NODE  NAME
# node     1234 app   0u    CHR    136,0   0t0       3     /dev/pts/0      (stdin)
# node     1234 app   3u    IPv4   12345   0t0       TCP   *:3000 (LISTEN)
# node     1234 app   12u   IPv4   23456   0t0       TCP   10.0.0.5:3000->10.0.0.8:54321 (ESTABLISHED)
# node     1234 app   15r   REG    8,1     4096      789   /app/data.json
# node     1234 app   18u   unix   0x...   0t0       34567 /var/run/pg.sock
```

Column meanings:
- **FD** — File descriptor number + mode (`r`=read, `w`=write, `u`=read/write)
- **TYPE** — `IPv4`/`IPv6` = network socket, `REG` = regular file, `unix` = Unix socket, `FIFO` = pipe, `CHR` = character device
- **NAME** — What it points to: file path, socket address, or device

**Step 3: Categorize the leak**

```bash
# Count FDs by type
lsof -p <pid> | awk '{print $5}' | sort | uniq -c | sort -rn
#  3500  IPv4     ← likely socket leak
#    25  REG
#     5  unix
#     3  FIFO

# If sockets are leaking, see where they're connected
lsof -p <pid> -i   # Only network connections

# Filter by state
lsof -p <pid> -i -a | grep CLOSE_WAIT
# node 1234 app 45u IPv4 TCP 10.0.0.5:3000->10.0.0.8:54321 (CLOSE_WAIT)
```

**Step 4: Examine connection states with `ss`**

```bash
# All TCP connections for the process
ss -tnp | grep <pid>

# Key output:
# State      Recv-Q  Send-Q  Local Address:Port  Peer Address:Port  Process
# ESTAB      0       0       10.0.0.5:3000       10.0.0.8:54321     users:(("node",pid=1234,fd=12))
# CLOSE-WAIT 0       0       10.0.0.5:3000       10.0.0.8:54322     users:(("node",pid=1234,fd=45))
# CLOSE-WAIT 0       0       10.0.0.5:3000       10.0.0.8:54323     users:(("node",pid=1234,fd=46))

# Count connections by state
ss -tn state close-wait | grep <pid> | wc -l

# Show all connection states summary
ss -s
```

**Understanding CLOSE_WAIT:**

CLOSE_WAIT means the remote side has closed its end of the connection (sent FIN), but the local process hasn't closed its end yet. A few CLOSE_WAIT connections are normal (brief transition). Hundreds or thousands growing over time means the application is not calling `.end()` or `.destroy()` on sockets after the remote side disconnects.

Common causes in Node.js:
- HTTP response objects not consumed or destroyed
- Database connections not returned to the pool
- HTTP client requests without proper error handling (the response socket hangs open)

**Step 5: Correlate lsof and ss to find the leak source**

```bash
# lsof shows: fd=45, connected to 10.0.0.8:54322, state CLOSE_WAIT
# ss shows the same connection with the fd number in the Process column

# Group CLOSE_WAIT connections by remote host
ss -tn state close-wait | awk '{print $4}' | cut -d: -f1 | sort | uniq -c | sort -rn
#  850  10.0.1.50     ← database server
#   12  10.0.2.100    ← external API

# If 850 CLOSE_WAIT connections point to your database, the leak is in database
# connection handling — connections aren't being released back to the pool
```

**Step 6: Track growth over time**

```bash
# Quick monitoring script
while true; do
  echo "$(date): FDs=$(ls /proc/<pid>/fd | wc -l) CLOSE_WAIT=$(ss -tn state close-wait | grep <pid> | wc -l)"
  sleep 10
done
```

If FD count and CLOSE_WAIT count grow in lockstep, your leak is specifically socket connections not being closed. Correlate the timing with your application's traffic patterns and recent deployments to narrow down which code path introduced the leak.

</details>

<details>
<summary>19. A server is experiencing high I/O wait and processes are getting stuck — use vmstat and iostat to identify whether the bottleneck is disk I/O, and use the /proc filesystem to inspect individual process state and open file descriptors. Show the exact commands, explain what %wa in top means, how to read vmstat's bi/bo columns, and how to identify which disk and which process is causing the I/O pressure</summary>

**Step 1: Confirm I/O wait with `top`**

```bash
top
# Look at the CPU line:
# %Cpu(s):  5.0 us,  2.0 sy,  0.0 ni, 10.0 id, 83.0 wa,  0.0 hi,  0.0 si
#                                                  ^^^^
#                                        83% I/O wait — system is I/O bound
```

**%wa (I/O wait)** = percentage of time the CPU is idle *because* it's waiting for I/O to complete. High %wa with low %us and low %id means the CPU has work to do but is blocked waiting on disk. This is the classic sign of an I/O bottleneck.

Also check load average: if it's high (much greater than CPU count) but %us is low, processes are piling up in the run queue waiting for I/O — confirmed by D-state processes (as covered in question 13).

**Step 2: System-wide I/O picture with `vmstat`**

```bash
# Sample every 2 seconds, 10 samples
vmstat 2 10

# procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
#  r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs  us sy id wa st
#  1 12      0  51200  25600 512000    0    0  85000  2000  500  800   3  2 12 83  0
#     ^^                                       ^^^^^  ^^^^                    ^^
#     b=12 blocked                             bi=85K bo=2K                   wa=83%
```

Key columns:
- **r** — Processes in run queue (waiting for CPU). High = CPU bottleneck.
- **b** — Processes in uninterruptible sleep (D-state, waiting for I/O). **High `b` is the red flag for I/O issues.**
- **si/so** — Swap in/out (KB/s). Non-zero means the system is swapping — bad for performance.
- **bi** — Blocks read from disk (KB/s). High bi = heavy read I/O.
- **bo** — Blocks written to disk (KB/s). High bo = heavy write I/O.
- **wa** — Same I/O wait percentage as top.

In the example: 12 processes blocked on I/O, reading 85 MB/s from disk, 83% I/O wait. Clear disk read bottleneck.

**Step 3: Identify which disk with `iostat`**

```bash
# Show per-device stats every 2 seconds, skip the first (boot-average) line
iostat -xdz 2

# Device  r/s    w/s   rMB/s  wMB/s  rrqm/s  wrqm/s  %util  await  r_await  w_await
# sda     4500   50    85.0   2.0    0       0        99.8   45.2   46.0     2.0
# sdb     10     5     0.1    0.05   0       0        1.2    0.5    0.4      0.8
```

Key columns:
- **%util** — How busy the device is (0-100%). **99.8% = disk is completely saturated.** This is the most important column.
- **await** — Average time (ms) for I/O requests to be served (queuing + service time). High await = slow disk or overloaded queue.
- **r/s, w/s** — Read/write operations per second.
- **rMB/s, wMB/s** — Throughput.
- **r_await, w_await** — Separate read vs write latency.

In the example: `sda` is at 99.8% utilization with 4500 reads/s and 45ms average wait. It's the bottleneck.

**Step 4: Find which process is causing the I/O with `/proc` and `iotop`**

```bash
# Easiest: iotop shows per-process I/O in real-time
iotop -o   # -o = only show processes doing I/O

# Total DISK READ:  85.00 M/s | Total DISK WRITE: 2.00 M/s
#   PID  PRIO  USER     DISK READ  DISK WRITE  SWAPIN     IO>    COMMAND
# 4567  be/4  postgres  82.50 M/s  1.50 M/s    0.00 %    95.00 % postgres: SELECT...
```

If `iotop` isn't installed, use `/proc`:

```bash
# Check a specific process's I/O stats
cat /proc/<pid>/io
# rchar: 890000000     ← bytes read (includes cache hits)
# wchar: 50000000
# read_bytes: 850000000 ← actual bytes read from disk
# write_bytes: 45000000

# Check process state
cat /proc/<pid>/status | grep State
# State: D (disk sleep)   ← confirmed D-state

# See what the process has open (which files is it reading?)
ls -la /proc/<pid>/fd | head -20
# Or use lsof
lsof -p <pid> | grep REG   # Regular files being accessed
```

**Step 5: Find what file/device the stuck process is waiting on**

```bash
# The wchan shows what kernel function the process is waiting in
cat /proc/<pid>/wchan
# io_schedule   ← waiting for disk I/O

# Stack trace shows the full kernel call stack
cat /proc/<pid>/stack
# [<0>] io_schedule+0x12/0x40
# [<0>] blkdev_read_iter+0x...
```

**Common causes and fixes:**

- **Database doing sequential scans** — Missing index causes full table reads. Fix the query/add an index.
- **Log files on the same disk** — Heavy logging saturates the disk. Move logs to a separate volume.
- **Insufficient IOPS** — Cloud disks have IOPS limits (e.g., gp2 volumes scale with size). Upgrade to gp3 with provisioned IOPS.
- **Swapping** — Check `vmstat` si/so columns. If non-zero, the system is swapping and all I/O competes with swap. Add memory or reduce workload.

</details>

<details>
<summary>20. A container's filesystem reports "no space left on device" but the disk isn't full — walk through the diagnosis using df, du, and lsof to determine whether the issue is inode exhaustion, deleted-but-open files, or a full overlay filesystem. Show the exact commands for each scenario and explain the fix for each root cause</summary>

The error is `ENOSPC: no space left on device`, but `df` shows available space. There are three common causes — work through them in order.

**Step 1: Check disk space AND inode usage**

```bash
# Check disk space (the obvious one)
df -h
# Filesystem      Size  Used  Avail Use%  Mounted on
# /dev/sda1       100G   45G   55G   45%  /
# overlay          50G   50G     0  100%  /var/lib/docker/overlay2/...

# Check inode usage — this is what most people forget
df -i
# Filesystem      Inodes   IUsed   IFree  IUse%  Mounted on
# /dev/sda1       6553600  6553600  0     100%   /
```

**Cause 1: Inode exhaustion**

Every file and directory consumes one inode, regardless of file size. Millions of tiny files (session files, temp files, cache entries) can exhaust inodes while barely using any disk space.

```bash
# Find directories with the most files
find / -xdev -printf '%h\n' | sort | uniq -c | sort -rn | head -20
# 2500000  /tmp/sessions
#  500000  /var/cache/app

# Or count files per top-level directory
for d in /*/; do echo "$(find "$d" -xdev -type f 2>/dev/null | wc -l) $d"; done | sort -rn
```

**Fix:** Delete the mass of small files. Then fix the application — add TTLs to session files, clean up temp files, or use a database instead of the filesystem for ephemeral data.

**Cause 2: Deleted-but-open files**

A file can be deleted (`rm`) but still consume disk space if a process has it open. The filesystem entry is removed, but the data blocks aren't freed until the file descriptor is closed. `du` reports less usage than `df` because `du` can't see deleted files.

```bash
# Compare df and du — if there's a big mismatch, deleted-open files are the cause
df -h /           # Shows 95G used
du -sh / 2>/dev/null  # Shows 60G used
# 35G gap = deleted-but-open files

# Find them with lsof
lsof +L1   # +L1 = files with link count < 1 (deleted)
# COMMAND  PID  USER  FD   TYPE  DEVICE  SIZE/OFF  NLINK  NODE  NAME
# node     1234 app   5w   REG   8,1     35G       0      789   /var/log/app.log (deleted)
```

**Fix:** Either restart the process (closes all FDs, space is reclaimed) or truncate the file descriptor without restarting:

```bash
# Truncate the deleted file via /proc — reclaims space without restart
: > /proc/1234/fd/5
```

This is the most common cause in containers — a log file grows huge, logrotate deletes the old file, but the process still has it open and keeps writing to it. The file is "deleted" but the space isn't freed.

**Cause 3: Full overlay filesystem (Docker-specific)**

Docker uses overlay2 filesystems. Each container's writable layer has a size limit (default varies by storage driver config). The host disk may have space, but the container's overlay is full.

```bash
# Check Docker's disk usage
docker system df
# TYPE          TOTAL   ACTIVE  SIZE    RECLAIMABLE
# Images        15      5       12.5GB  8.2GB
# Containers    8       3       45GB    42GB    ← container writable layers
# Local Volumes 10      3       25GB    20GB
# Build Cache                   8GB     8GB

# Check per-container disk usage
docker system df -v

# See what's using space inside the container
docker exec <container> du -sh /* 2>/dev/null | sort -rh | head -10
# 40G  /var/log
# 3G   /tmp
```

**Fix:**
```bash
# Immediate relief — clean up unused Docker resources
docker system prune -a --volumes  # WARNING: removes all stopped containers, unused images, volumes

# Per-container: find and delete large files
docker exec <container> find /var/log -name "*.log" -size +100M -delete

# Prevent recurrence: set container storage limits
# In Docker daemon config, or use tmpfs for writable dirs:
docker run --tmpfs /tmp:size=100m myimage
```

**Diagnostic flowchart summary:**

1. `df -h` — If a filesystem is 100%, that's your answer. Check overlay mounts too.
2. `df -i` — If IUse% is 100%, it's inode exhaustion.
3. `df -h` vs `du -sh` mismatch — Deleted-but-open files. Confirm with `lsof +L1`.
4. `docker system df` — If Docker is eating space, clean up images/containers/volumes.

</details>

---

## Experience-Based Questions

These questions test real-world experience. Prepare by mapping them to your own projects and situations.

<details>
<summary>21. Tell me about a time you debugged a Linux performance issue in production — what were the symptoms, what tools did you use to narrow down the root cause, and what was the fix?</summary>

**What the interviewer is looking for:**
- Systematic debugging approach, not random guessing
- Familiarity with real Linux diagnostic tools (not just "I looked at the dashboard")
- Ability to correlate symptoms to root causes
- Understanding of the fix AND how you prevented recurrence

**Suggested structure (STAR-ish, adapted for debugging):**

1. **Context** — What system, what scale, what was the normal state
2. **Symptoms** — How the issue manifested (alerts, user reports, metrics)
3. **Triage** — First steps to narrow the scope (which host, which process, what kind of bottleneck)
4. **Investigation** — The specific tools and commands you used, what each one told you, how you narrowed from system-wide to the specific cause
5. **Root cause** — What was actually wrong
6. **Fix** — Immediate mitigation + permanent fix
7. **Prevention** — What you added to catch this earlier next time

**Key points to hit:**
- Name specific tools: `top`/`htop`, `vmstat`, `iostat`, `strace`, `lsof`, `ss`, `dmesg`, `journalctl` — whichever are relevant to your story
- Show that you understand the difference between CPU-bound, memory-bound, and I/O-bound problems and how the tools distinguish them
- Mention metrics/alerts and how they guided (or failed to guide) your investigation
- If the issue involved containers, mention cgroup limits, throttling, or namespace-related complications

**Example outline to personalize:**

> "We had a Node.js API service handling event processing. Latency spiked to 10x normal during peak hours, and p99 went from 200ms to 2+ seconds. Alerts fired on response time.
>
> I SSH'd into the affected pod's host and ran `top` — CPU usage was low (~15%) but I/O wait was at 60%. That pointed away from CPU and toward disk. I ran `vmstat 2` and confirmed: the `b` (blocked) column showed 8-10 processes in D-state, and `bi` was extremely high.
>
> `iostat -xdz 2` showed the EBS volume at 99% utilization. `iotop` revealed our logging sidecar was doing massive disk reads — it was re-reading old log files during rotation instead of just tailing.
>
> Immediate fix: we adjusted the log collector configuration to stop re-reading historical logs. Permanent fix: we moved to stdout-based logging (as covered in question 14), separated log storage to a dedicated volume with provisioned IOPS, and added I/O wait alerts to our monitoring to catch this earlier."

</details>

<details>
<summary>22. Describe a time you dealt with resource exhaustion in production (file descriptor leak, OOM kill, memory exhaustion, or connection leak) — how did you detect it, what was the investigation process, and how did you prevent it from recurring?</summary>

**What the interviewer is looking for:**
- Understanding of finite OS resources (FDs, memory, connections, PIDs) and what happens when they're exhausted
- Ability to detect and investigate resource leaks — not just "we restarted it"
- Root cause analysis that goes deeper than the symptom
- Preventive measures (monitoring, limits, code fixes)

**Suggested structure:**

1. **Detection** — How you first noticed (alert, error logs, user reports, OOMKilled status)
2. **Triage** — Confirming which resource was exhausted and on which process/container
3. **Investigation** — Tools used to identify the leak pattern (lsof, ss, /proc, memory profiling, metrics over time)
4. **Root cause** — The specific code path or configuration that caused the leak
5. **Fix** — Immediate (restart, raise limits) and permanent (code fix, configuration change)
6. **Prevention** — Monitoring, alerts, tests, or architectural changes to catch it before it becomes an incident

**Key points to hit:**
- Show the difference between the symptom and the root cause (OOMKill is the symptom; a memory leak in a request handler is the root cause)
- Mention the tools from this file's practical section: `lsof` for FD leaks, `ss` for connection leaks and CLOSE_WAIT, container OOMKilled status and `dmesg`/`journalctl -k` for OOM kills, heap snapshots for Node.js memory leaks
- Explain how you determined the leak was growing over time (trending FD count, memory usage over hours/days)
- Describe what you added to monitoring — resource usage dashboards, alerts on FD count approaching limits, memory growth rate alerts

**Example outline to personalize:**

> "Our API service started returning 502s intermittently. The load balancer health checks were passing, but random requests failed. The error logs showed `EMFILE: too many open files`.
>
> I checked FD count with `ls /proc/<pid>/fd | wc -l` — it was at 980 out of a 1024 soft limit. I ran `lsof -p <pid>` and categorized the FDs: 900+ were TCP sockets. `ss -tn state close-wait` showed 850 connections in CLOSE_WAIT to our downstream payment service.
>
> The root cause was a code path in our HTTP client that didn't consume or destroy the response body on non-200 responses. The socket stayed open because HTTP keep-alive was waiting for the body to be read. The fix was adding `.destroy()` on the response stream in the error handler.
>
> For prevention, we raised `ulimit -n` to 65536 as a safety buffer, added a Prometheus gauge tracking open FD count per pod, and set an alert at 80% of the limit. We also added a connection pool max-size to the HTTP client so even a leak couldn't exhaust all FDs."

</details>

<details>
<summary>23. Describe a time a containerized application behaved differently than it did on a developer's machine — what was the root cause (namespace isolation, cgroup limits, filesystem differences, permission issues), and how did you diagnose and fix it?</summary>

**What the interviewer is looking for:**
- Understanding that containers are not VMs — they share the host kernel and are subject to namespace, cgroup, and filesystem differences
- Ability to diagnose "works on my machine" problems systematically
- Knowledge of the specific ways containers differ from local development (memory limits, CPU throttling, filesystem permissions, DNS resolution, network isolation)

**Suggested structure:**

1. **Context** — What the app was, how it ran locally vs in production (Docker Compose vs Kubernetes, etc.)
2. **Symptom** — What was different (crashes, slow performance, permission errors, missing files, networking failures)
3. **Initial mismatch** — Why the local environment masked the issue
4. **Diagnosis** — How you narrowed it from "it works locally" to the specific container difference
5. **Root cause** — The specific OS/container primitive that caused the behavior difference
6. **Fix** — Both the immediate fix and any changes to local development setup to catch this earlier

**Key points to hit:**
- Reference specific container primitives: cgroup memory limits causing OOMKill that doesn't happen locally (as covered in question 9), CPU throttling causing latency spikes (question 9), file permission UID mismatches (question 10), PID 1 signal handling differences (question 15)
- Show awareness that local Docker often runs with more resources, as root, without CPU limits, and with host networking — all of which mask production issues
- Mention how you made local development more representative of production (matching resource limits, running as non-root, testing with realistic configs)

**Example outline to personalize:**

> "We deployed a new Node.js service to Kubernetes. It worked perfectly in local Docker and staging, but in production it crashed with OOMKilled every 30-60 minutes under load.
>
> Locally, Docker Desktop had 8 GB allocated and no per-container memory limit. In production, the pod had a 512 Mi memory limit. The Node.js process had no `--max-old-space-size` set, so V8 defaulted to ~1.5 GB heap — far above the cgroup limit. Under load, heap usage grew past 512 Mi and the cgroup OOM killer terminated the process instantly (as covered in question 9).
>
> The fix was setting `--max-old-space-size=384` (75% of the 512 Mi limit, leaving room for native memory and overhead). We also discovered a memory leak in a request handler that was accumulating parsed response bodies in an array — this was masked locally because the larger heap meant it took hours to surface.
>
> Going forward, we added resource limits to our Docker Compose files matching production, added `--max-old-space-size` to all service Dockerfiles, and set up memory usage alerts that fire at 80% of the cgroup limit so we catch growth before the OOM killer does."

**Common root causes to have in your back pocket:**
- **Memory**: No `--max-old-space-size` + cgroup limit = OOMKill (most common)
- **CPU**: CPU throttling in K8s causing event loop stalls and timeout cascades
- **Permissions**: Container runs as non-root (UID 1001) but volume files are owned by UID 1000 — `EACCES` on writes
- **DNS**: Local Docker uses host DNS; Kubernetes uses CoreDNS. `ndots:5` default causes excessive DNS lookups for external domains, adding latency
- **Filesystem**: Local bind mounts are writable; production uses read-only root filesystem with specific writable tmpfs mounts. App tries to write to a read-only path and crashes
- **Signals**: App runs as PID 1 in production but not locally (Docker Compose uses shell form CMD which wraps in `/bin/sh`). SIGTERM handling differs (question 15)

</details>
