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

<!-- Answer will be added later -->

</details>

<details>
<summary>2. Why does Unix treat everything as a file (sockets, pipes, devices) and what does that design decision enable — how do file descriptors work as the unified handle for all I/O, what happens when a process exhausts its file descriptor limit, and what are the typical symptoms of FD exhaustion in a Node.js server?</summary>

<!-- Answer will be added later -->

</details>

## Conceptual Depth

<details>
<summary>3. How is process memory organized into stack and heap — why does the OS give each thread its own stack but share the heap across threads, what causes a stack overflow vs heap exhaustion, and how does Node.js expose control over V8's heap via --max-old-space-size — what happens when you set it too low or too high?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>4. What is virtual memory and why does every process get its own address space instead of directly accessing physical RAM — how do page faults work (minor vs major), what role does swap play, and how does the Linux OOM killer decide which process to terminate when memory is exhausted?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>5. What are the different I/O models available on Linux (blocking, non-blocking, multiplexing with select/poll/epoll, and async I/O) — why did the kernel evolve from select to poll to epoll, what fundamental scalability problem does each one solve that its predecessor couldn't, and why is epoll the mechanism that powers libuv and therefore Node.js?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>6. What are the main inter-process communication mechanisms on Linux (pipes, Unix domain sockets, shared memory, signals) — what are the tradeoffs of each in terms of performance, complexity, and use cases, and when would you choose one over another for communication between backend services on the same host?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>7. How do Unix signals work and why does it matter that SIGKILL and SIGTERM are fundamentally different — what does each common signal (SIGTERM, SIGKILL, SIGINT, SIGHUP, SIGUSR1/2) mean, which ones can be caught and which cannot, and how should a Node.js process running in a container handle SIGTERM to achieve graceful shutdown — what goes wrong if it doesn't?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>8. How do Linux namespaces provide process isolation -- what does each namespace type (PID, network, mount, user, UTS) isolate, how do they compose to create what we call a "container," and why is this isolation fundamentally weaker than a VM -- what can a container escape look like?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>9. How do cgroups enforce resource limits for containers -- what resources can cgroups control (CPU, memory, I/O), how does the OOM killer behave when a container hits its cgroup memory limit vs when the host runs out of memory, and what happens when cgroup limits are set too aggressively?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>10. Why do Linux file permissions use the owner/group/other model with read/write/execute bits — what do setuid, setgid, and the sticky bit do and when are they dangerous, and why does running containers as non-root create permission issues with mounted volumes — what are the common fixes?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>11. How do environment variables work at the OS level -- how does a child process inherit its parent's environment, why can't you modify a parent's environment from a child, and why is /proc/PID/environ a security concern even when you think environment variables are "hidden"?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>12. How do Node.js worker threads differ from OS-level threads, and how do both compare to Go's goroutine model — why does Node.js use a single-threaded event loop with optional worker threads instead of a thread-per-request model, and what are the practical implications for CPU-bound vs I/O-bound workloads in each model?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>13. What are zombie processes and processes stuck in D-state (uninterruptible sleep) — why do zombies exist and why can't you kill them with SIGKILL, what causes D-state and why is it a sign of I/O problems, and what is the impact of each on system health?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>14. Why does the container ecosystem standardize on writing logs to stdout/stderr instead of log files -- how do the three standard streams (stdin, stdout, stderr) work at the OS level, how do container runtimes capture stdout/stderr for log aggregation, and what are the tradeoffs of stdout-based logging vs writing to files including log rotation concerns?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>15. Why does PID 1 have special responsibilities in Linux and what goes wrong when a container runs without a proper init process -- what are PID 1's duties (signal forwarding, orphan reaping), how does systemd handle service lifecycle management (units, dependencies, restart policies), and how do you use journalctl to inspect service logs and diagnose startup failures?</summary>

<!-- Answer will be added later -->

</details>

## Practical — Debugging & Inspection

<details>
<summary>16. A server is slow and you suspect CPU or memory issues — walk through the exact steps using ps and top/htop to identify the offending processes, explain what the key columns mean (RES, VIRT, SHR, %CPU, S state), how to spot zombie processes and what to do about them, and how to identify which process is consuming the most resources</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>17. A Node.js process is hanging and you don't know why — use strace to attach to the running process and diagnose what system calls it's stuck on. Show the exact strace commands, explain how to interpret the output (identifying blocked reads, slow network calls, or filesystem waits), and what the common patterns look like for different types of hangs</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>18. You suspect a Node.js server has a file descriptor leak — use lsof to list open FDs for the process, identify what types of FDs are leaking (sockets, pipes, files), and use ss (or netstat) to examine connection states and identify connections stuck in CLOSE_WAIT. Show the exact commands, explain the output columns, and describe how to correlate lsof and ss output to find the leak source</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>19. A server is experiencing high I/O wait and processes are getting stuck — use vmstat and iostat to identify whether the bottleneck is disk I/O, and use the /proc filesystem to inspect individual process state and open file descriptors. Show the exact commands, explain what %wa in top means, how to read vmstat's bi/bo columns, and how to identify which disk and which process is causing the I/O pressure</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>20. A container's filesystem reports "no space left on device" but the disk isn't full — walk through the diagnosis using df, du, and lsof to determine whether the issue is inode exhaustion, deleted-but-open files, or a full overlay filesystem. Show the exact commands for each scenario and explain the fix for each root cause</summary>

<!-- Answer will be added later -->

</details>

---

## Experience-Based Questions

These questions test real-world experience. Prepare by mapping them to your own projects and situations.

<details>
<summary>21. Tell me about a time you debugged a Linux performance issue in production — what were the symptoms, what tools did you use to narrow down the root cause, and what was the fix?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>22. Describe a time you dealt with resource exhaustion in production (file descriptor leak, OOM kill, memory exhaustion, or connection leak) — how did you detect it, what was the investigation process, and how did you prevent it from recurring?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>23. Describe a time a containerized application behaved differently than it did on a developer's machine — what was the root cause (namespace isolation, cgroup limits, filesystem differences, permission issues), and how did you diagnose and fix it?</summary>

<!-- Answer framework will be added later -->

</details>
