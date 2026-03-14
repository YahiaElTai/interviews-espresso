# Kubernetes

> **28 questions** — 10 theory, 14 practical, 4 experience

- Core primitives: Pods, Deployments, Services, Namespaces — and why K8s separates them
- Control plane architecture: API server, etcd, scheduler, controller manager — failure modes
- Kubernetes networking model: flat network, CNI plugins, pod-to-pod communication
- DNS and service discovery: CoreDNS internals, DNS-based routing
- When to adopt K8s vs simpler alternatives — complexity and cost tradeoffs
- Service types: ClusterIP, NodePort, LoadBalancer, ExternalName — when to use each
- RBAC: Roles, ClusterRoles, Bindings, ServiceAccounts — common over-permissioning mistakes
- Storage: PV, PVC, StorageClass — differences from Docker volumes
- Workload types: Deployments, StatefulSets, DaemonSets, Jobs, CronJobs
- Init containers and sidecar pattern: dependency setup, migration scripts, logging/proxy sidecars
- ConfigMaps and Secrets: env vars vs volume mounts, multi-environment setup
- Health probes: liveness, readiness, startup — timing parameters and cascading restart pitfalls
- Autoscaling: HPA (CPU + custom metrics), VPA, Cluster Autoscaler interaction
- Resource requests, limits, LimitRanges, ResourceQuotas, and namespace isolation — QoS classes (Guaranteed, Burstable, BestEffort) and eviction priority
- Deployment strategies: rolling updates (maxSurge, maxUnavailable), blue-green, canary — when each is appropriate
- PodDisruptionBudgets and zero-downtime deployments (preStop hooks, connection draining)
- Ingress: TLS termination with cert-manager, ACME flow, rate limiting
- Helm: chart architecture, values overrides, hooks for migrations, Helm vs Kustomize
- Production debugging: CrashLoopBackOff, Pending pods, OOMKilled, service-to-service networking failures, ephemeral containers (kubectl debug)
- Scheduling: taints, tolerations, node affinity, pod priorities, preemption
- Network Policies and microsegmentation (allow/deny traffic between services)

---

## Foundational

<details>
<summary>1. What are the core Kubernetes primitives (Pods, Deployments, Services, Namespaces) and why does K8s separate them into distinct abstractions instead of just running containers directly — what design problem does each primitive solve, and how do they compose together to form a working application?</summary>

Kubernetes separates concerns into composable primitives so each one can evolve independently. Running containers directly gives you no lifecycle management, no networking, and no scaling — you'd have to build all of that yourself.

**Pod** — the smallest deployable unit. A pod wraps one or more containers that share the same network namespace (localhost) and storage volumes. The design problem it solves: tightly coupled processes (like an app + a log shipper) need to be co-located and share resources. You never deploy a bare container — you deploy a pod.

**Deployment** — manages the desired state of a set of identical pods. It solves lifecycle management: rolling updates, rollbacks, scaling to N replicas, and self-healing (if a pod dies, the Deployment's underlying ReplicaSet creates a replacement). Without it, you'd manually track which pods are running and restart crashed ones yourself.

**Service** — provides a stable network identity (DNS name + ClusterIP) for a set of pods. Pods are ephemeral — they get new IPs every time they restart. Services solve the service discovery problem by selecting pods via labels and load-balancing traffic across them. Without Services, every consumer would need to track pod IPs dynamically.

**Namespace** — a logical boundary for resource isolation. It solves multi-tenancy within a cluster: different teams or environments (dev, staging) can share a cluster while having separate RBAC rules, resource quotas, and network policies. Resources in different namespaces can have the same name without conflicting.

**How they compose:** A Deployment creates pods with specific labels, a Service selects those pods by label and routes traffic to them, and a Namespace scopes everything so you can have `api-service` in both `staging` and `production` without collision. This separation lets you change routing, scaling, or isolation independently.

</details>

<details>
<summary>2. How does the Kubernetes control plane work — why is it split into API server, etcd, scheduler, and controller manager instead of being a single process, how do these components communicate, and what happens to your running workloads when each one fails?</summary>

The control plane is split because each component has a distinct responsibility, different scaling characteristics, and different failure impacts. A monolithic process would be harder to reason about, harder to make highly available, and a single bug could take down everything.

**API Server (kube-apiserver)** — the only component that talks to etcd. It's the front door for all cluster operations: kubectl commands, controller watches, kubelet status reports. It authenticates, authorizes, validates, and persists every change. It's stateless and horizontally scalable — you run multiple replicas behind a load balancer.

**etcd** — the distributed key-value store that holds all cluster state (desired and actual). It's the single source of truth. It uses Raft consensus for consistency across replicas. Only the API server reads/writes to etcd — no other component touches it directly.

**Scheduler (kube-scheduler)** — watches for newly created pods with no assigned node, evaluates constraints (resource requests, affinity rules, taints/tolerations), scores candidate nodes, and binds the pod to the best node by writing the assignment back to the API server.

**Controller Manager (kube-controller-manager)** — runs dozens of control loops (Deployment controller, ReplicaSet controller, Node controller, etc.). Each loop watches the desired state in etcd (via the API server), compares it to actual state, and takes corrective action. For example, the ReplicaSet controller notices you want 3 replicas but only 2 exist, so it creates a new pod.

**Communication pattern:** Everything goes through the API server. Controllers and the scheduler use watch streams (long-lived HTTP connections) to get notified of changes. Kubelets on worker nodes report pod status back to the API server. No component talks directly to another — the API server is the hub.

**Failure modes:**

- **API server down:** kubectl stops working, no new deployments, no scaling changes. But already-running pods continue running — kubelets keep containers alive independently. The cluster is frozen but not broken.
- **etcd down:** API server can't read or write state. Same effect as API server down, but worse — even if the API server is up, it has no backend. Running pods continue unaffected.
- **Scheduler down:** Existing pods keep running. New pods stay in `Pending` state because nothing assigns them to nodes. No impact on current workloads.
- **Controller manager down:** Existing pods keep running. But self-healing stops — if a pod crashes, no controller notices or creates a replacement. Deployments won't scale, rollouts won't progress.

The key insight: the data plane (kubelet + container runtime on each node) operates independently of the control plane. Your applications survive control plane outages — you just lose the ability to make changes or self-heal.

</details>

<details>
<summary>3. Why does Kubernetes require a flat networking model where every pod gets its own IP and can reach every other pod directly — what problem does this solve compared to Docker's default networking, and what role do CNI plugins play in implementing this model?</summary>

**The problem with Docker's default networking:** Docker uses bridge networking where containers on the same host share a private network, but containers on different hosts can't see each other without explicit port mapping (`-p 8080:3000`). This creates port conflicts (two containers can't both map to host port 8080), forces applications to know about port mappings, and makes cross-host communication complicated. At scale with hundreds of containers across many hosts, managing port mappings becomes unworkable.

**Kubernetes' flat network requirement:** Every pod gets a unique, routable IP address. Any pod can reach any other pod directly using that IP — no NAT, no port mapping, no proxies. A process running in a pod sees the same IP that other pods see. This is called the "IP-per-pod" model.

**What this solves:**

- **No port conflicts** — each pod has its own IP, so every pod can listen on port 3000 without colliding with other pods.
- **Simplified application code** — apps don't need to know about port translations or host networking. They bind to a port and it just works.
- **Consistent networking** — the same networking model works whether pods are on the same node or different nodes. Applications don't need to care about topology.
- **Clean service discovery** — Services can route to pod IPs directly. No complex NAT chains to debug.

**CNI (Container Network Interface) plugins** are what actually implement this model. Kubernetes defines the requirements (every pod gets an IP, pods can reach each other) but doesn't implement the networking itself. CNI plugins handle the actual work:

- **Allocate IPs** to pods from a defined range (IPAM)
- **Set up network routes** so pods on different nodes can communicate
- **Configure virtual network interfaces** on each pod

Popular CNI plugins include Calico (uses BGP routing, supports Network Policies), Cilium (uses eBPF for high performance and advanced security), Flannel (simple overlay network using VXLAN), and AWS VPC CNI (assigns actual VPC IPs to pods for native AWS integration).

The choice of CNI plugin affects performance, Network Policy support, and operational complexity, but the flat network contract remains the same regardless of which plugin you use.

</details>

## Conceptual Depth

<details>
<summary>4. How does DNS-based service discovery work in Kubernetes — what does CoreDNS do, how does a pod resolve another service by name (short name vs FQDN: service.namespace.svc.cluster.local), how does CoreDNS handle search domains, and what breaks when CoreDNS is misconfigured or overloaded?</summary>

**CoreDNS** is the cluster DNS server deployed as a Deployment in the `kube-system` namespace. It watches the Kubernetes API for Service and Endpoint changes and maintains DNS records automatically. Every pod's `/etc/resolv.conf` is configured to point to the CoreDNS Service IP (typically `10.96.0.10`) as its nameserver.

**How name resolution works:**

When a pod calls `http://my-api:3000`, the DNS resolver in the pod tries to resolve `my-api`. Because it's not a fully qualified domain name, the resolver appends search domains from `/etc/resolv.conf`:

```
# /etc/resolv.conf in a pod (namespace: backend)
nameserver 10.96.0.10
search backend.svc.cluster.local svc.cluster.local cluster.local
ndots:5
```

The resolver tries each search domain in order:
1. `my-api.backend.svc.cluster.local` — matches if the Service is in the same namespace
2. `my-api.svc.cluster.local` — tried next if step 1 fails
3. `my-api.cluster.local` — tried next
4. `my-api` — bare name, forwarded to upstream DNS

**Short name vs FQDN:**
- `my-api` — uses search domains, triggers up to 4 DNS queries before finding a match (or failing). Works only if the Service is in the same namespace (or happens to match a later search domain).
- `my-api.other-namespace` — appends search domains, resolves to `my-api.other-namespace.svc.cluster.local`.
- `my-api.other-namespace.svc.cluster.local.` (trailing dot) — treated as an FQDN, no search domain expansion, exactly one DNS query. Most efficient.

**The `ndots:5` setting** means any name with fewer than 5 dots triggers search domain expansion. This is why `my-api` (0 dots) gets search domains appended, but it also means external names like `api.github.com` (2 dots, still < 5) also get search domain expansion first — causing unnecessary DNS queries before falling through to upstream resolution.

**What breaks when CoreDNS fails:**

- **CoreDNS pods down:** All DNS resolution fails. Services can't find each other by name. Pods that cache DNS or use IPs directly continue working, but any new connections using DNS names fail. This effectively partitions the cluster at the application level.
- **CoreDNS overloaded:** DNS queries timeout or return errors slowly. Application latency spikes because every HTTP request starts with a DNS lookup. Services appear intermittently unreachable. Common causes: too many pods for the CoreDNS replica count, or `ndots:5` causing excessive query amplification.
- **Misconfigured CoreDNS Corefile:** Wrong upstream resolvers break external DNS. Missing or wrong zone configurations break internal resolution. Symptoms are confusing because some names resolve and others don't.

**Practical tips:** For cross-namespace calls in performance-sensitive paths, use the FQDN with a trailing dot to avoid search domain expansion. If DNS latency is a problem, consider increasing CoreDNS replicas or using NodeLocal DNSCache (a DaemonSet that caches DNS on each node).

</details>

<details>
<summary>5. When should you adopt Kubernetes vs simpler alternatives like ECS, Cloud Run, or plain VMs — what are the real complexity and cost tradeoffs, and what signals tell you a team is adopting K8s too early?</summary>

**The core tradeoff:** Kubernetes gives you the most flexibility and portability but has the highest operational complexity. Simpler alternatives trade flexibility for reduced operational burden.

**When K8s makes sense:**
- You have 10+ services with complex networking, scheduling, and scaling requirements
- You need multi-cloud or hybrid-cloud portability (K8s is the only real abstraction that works across AWS, GCP, Azure, on-prem)
- You have a dedicated platform team (or at least 1-2 engineers) to manage cluster operations
- You need fine-grained control over networking (Network Policies, service mesh), scheduling (affinity, taints), and resource management
- You're running stateful workloads alongside stateless ones with different lifecycle requirements

**When simpler alternatives are better:**

- **ECS/Fargate** — you're all-in on AWS, have fewer than ~10 services, and want container orchestration without managing nodes. Much less operational surface. No cluster upgrades, no CNI plugin choices, no etcd maintenance.
- **Cloud Run / App Runner** — your workloads are stateless HTTP services with variable traffic. Zero infrastructure management. Scales to zero. Best for straightforward request/response services.
- **Plain VMs + systemd** — you have 1-3 services, simple deployment needs, and the team doesn't have container expertise. Don't underestimate how far a well-configured VM with a process manager gets you.

**Signals a team is adopting K8s too early:**
- No one on the team has operated a K8s cluster before, and you're planning to self-manage it
- You have fewer than 5 services and no plans to grow significantly
- The primary motivation is "it's industry standard" rather than solving a specific operational problem
- You're spending more time on cluster operations than on application development

**Cost tradeoffs:** K8s itself is free, but the real cost is engineering time. Cluster upgrades, node management, monitoring the control plane, debugging networking issues, managing RBAC — this work is invisible until something breaks. A managed K8s service (EKS/GKE/AKS) reduces but doesn't eliminate this. Budget at least 20-30% of one platform engineer's time for ongoing K8s operations.

**Practical recommendation:** Start with the simplest thing that works. If you're on AWS with a small team, start with ECS Fargate. If your workloads are stateless HTTP, start with Cloud Run. Migrate to K8s when you hit real limitations — not when you anticipate them.

</details>

<details>
<summary>6. What are the Kubernetes Service types (ClusterIP, NodePort, LoadBalancer, ExternalName), why does each exist, and how do you decide which one to use — what are the tradeoffs of each and when would you avoid them?</summary>

**ClusterIP** (default) — assigns an internal-only virtual IP. Only reachable from within the cluster. This is your go-to for service-to-service communication. The API talks to the database via a ClusterIP Service. No external exposure, no extra infrastructure cost.

**NodePort** — exposes the Service on a static port (30000-32767) on every node's IP. Builds on top of ClusterIP. Useful for development, testing, or when you need direct access without a cloud load balancer. Avoid in production because: the port range is limited, you expose every node, clients need to know node IPs, and there's no built-in health checking at the node level. Traffic hits any node and gets routed internally — even if the pod isn't on that node, adding a network hop.

**LoadBalancer** — provisions a cloud provider load balancer (AWS NLB/ALB, GCP LB) that routes to the Service. Builds on top of NodePort. This is how you expose services to the internet in cloud environments. The downside: each LoadBalancer Service creates a separate cloud LB, which costs money (~$15-20/month on AWS). If you have 20 services, that's 20 load balancers. This is why most teams use an Ingress controller instead — one LoadBalancer Service for the Ingress controller, with Ingress rules routing to internal ClusterIP services.

**ExternalName** — doesn't proxy traffic at all. It creates a CNAME DNS record pointing to an external hostname (like `my-db.us-east-1.rds.amazonaws.com`). No ClusterIP, no proxying. Useful for giving an in-cluster DNS name to an external service so your application code doesn't hardcode external hostnames. Avoid if the external service needs IP-based access (CNAME doesn't work for that) or if you need port remapping.

**Decision guide:**

| Scenario | Service type |
|---|---|
| Internal service-to-service | ClusterIP |
| External-facing HTTP/HTTPS | ClusterIP + Ingress (preferred) or LoadBalancer |
| External-facing TCP/UDP (non-HTTP) | LoadBalancer |
| Development/debugging access | NodePort or `kubectl port-forward` |
| Alias for an external resource | ExternalName |

**Headless Services** (ClusterIP: None) are also worth knowing — they skip the virtual IP and return pod IPs directly in DNS. Used by StatefulSets where clients need to address specific pods (e.g., connecting to a specific database replica).

</details>

<details>
<summary>7. Why does Kubernetes have multiple workload types (Deployments, StatefulSets, DaemonSets, Jobs, CronJobs) instead of just one — what specific problem does each solve, and what goes wrong when you use the wrong one for a given workload?</summary>

Different workloads have fundamentally different lifecycle requirements. A web server that should always be running with interchangeable replicas is nothing like a database that needs stable identity and persistent storage, which is nothing like a batch job that should run once and exit. A single abstraction would either be too simple (missing critical features) or too complex (every option configurable for every case).

**Deployment** — manages stateless, interchangeable replicas. Pods are fungible (any replica can handle any request). Supports rolling updates, rollbacks, and scaling. Use for: API servers, web frontends, workers that pull from a shared queue. The underlying ReplicaSet ensures the desired count is maintained.

**StatefulSet** — manages stateful workloads that need stable identity. Each pod gets a persistent hostname (`db-0`, `db-1`, `db-2`), ordered startup/shutdown, and a dedicated PersistentVolumeClaim. Use for: databases, Kafka brokers, ZooKeeper, Elasticsearch — anything where pods are not interchangeable and need stable network identity or storage.

**DaemonSet** — runs exactly one pod per node (or a subset of nodes via node selectors). When a new node joins the cluster, the DaemonSet automatically schedules a pod on it. Use for: log collectors (Fluentd), monitoring agents (Prometheus node-exporter), network plugins (CNI), storage drivers. These are node-level infrastructure concerns.

**Job** — runs a pod to completion, then stops. If the pod fails, it retries (configurable). Use for: database migrations, batch processing, data imports, one-off scripts. You can configure parallelism and completion count for batch work.

**CronJob** — creates a Job on a schedule (cron syntax). Use for: periodic reports, cleanup tasks, scheduled backups, sending digest emails. Manages Job history and concurrency policy (allow, forbid, or replace concurrent runs).

**What goes wrong with the wrong type:**

- **Deployment for a database:** Pods get random names, no stable storage binding, rolling updates might start a new replica before the old one properly shuts down — risking data corruption. No ordered scaling.
- **StatefulSet for a stateless API:** Unnecessary complexity. Rolling updates are sequential (pod-by-pod) instead of parallel, making deployments slow. You're paying for ordering guarantees you don't need.
- **Deployment instead of DaemonSet for log collection:** Some nodes might have zero log collectors while others have multiple. No guarantee of node coverage.
- **Deployment instead of Job for a migration:** The pod keeps restarting after the migration completes because the Deployment sees it as "crashed" and tries to maintain the desired replica count.

</details>

<details>
<summary>8. Why does Kubernetes need its own storage abstraction (PV, PVC, StorageClass) instead of just using Docker volumes — how does the PV/PVC binding model work, what does dynamic provisioning via StorageClass solve, and when should you avoid persistent storage in K8s entirely?</summary>

**Why Docker volumes aren't enough:** Docker volumes are local to a single host. If a container moves to a different node (which K8s does routinely — rescheduling after node failures, rolling updates), the data is gone. Kubernetes needs storage that survives pod rescheduling across nodes, which means storage must be decoupled from the node lifecycle.

**The PV/PVC model:**

- **PersistentVolume (PV)** — represents an actual piece of storage (an EBS volume, an NFS share, a GCE Persistent Disk). It's a cluster-level resource, not tied to any namespace. Think of it as "a disk exists somewhere."
- **PersistentVolumeClaim (PVC)** — a request for storage by a pod. It specifies how much storage and what access mode (ReadWriteOnce, ReadOnlyMany, ReadWriteMany). Think of it as "I need a disk with these characteristics."
- **Binding:** When a PVC is created, the control plane finds a PV that satisfies the request (size, access mode, StorageClass) and binds them together. Once bound, it's a 1:1 relationship. The pod references the PVC, the PVC references the PV, the PV points to actual storage.

This separation matters because it decouples the app developer (who writes PVCs) from the infrastructure admin (who manages PVs). Developers don't need to know whether storage is EBS, NFS, or Ceph.

**StorageClass and dynamic provisioning:** Without StorageClass, an admin must pre-create PVs manually — tedious and doesn't scale. StorageClass defines a "template" for storage provisioning. When a PVC references a StorageClass, Kubernetes automatically provisions a new PV on demand. For example, a StorageClass for AWS EBS means every new PVC automatically creates an EBS volume.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
reclaimPolicy: Delete        # delete the PV when PVC is deleted
volumeBindingMode: WaitForFirstConsumer  # don't provision until pod is scheduled
```

**When to avoid persistent storage in K8s:**

- **Databases with high I/O requirements** — managed services (RDS, Cloud SQL) handle replication, backups, failover, and performance tuning. Running PostgreSQL in K8s means you own all of that.
- **When your workload is stateless** — if data belongs in an external store (S3, managed DB), don't add PVCs just because you can.
- **Small teams without StatefulSet expertise** — StatefulSet + PV management is a source of subtle bugs (volume stuck in wrong AZ, PV not released after PVC deletion, data loss from wrong reclaim policy).

**Practical rule:** Use managed services for databases. Use PVs in K8s for caching layers, message brokers, or search engines where data loss is recoverable — and even then, prefer managed alternatives when available.

</details>

<details>
<summary>9. Why does Kubernetes have three separate health probes (liveness, readiness, startup) instead of just one — what specific failure mode does each one detect, how do the timing parameters (initialDelaySeconds, periodSeconds, failureThreshold) interact, and how do misconfigured liveness probes cause cascading restarts that can take down an entire service?</summary>

Each probe answers a different question about the container's health, and conflating them causes real production incidents.

**Liveness probe** — "Is this container still alive, or is it stuck in an unrecoverable state?" If it fails, kubelet kills and restarts the container. Detects: deadlocks, infinite loops, corrupted state where the process is running but can't make progress. Example: a Node.js process that's alive but the event loop is blocked.

**Readiness probe** — "Can this container handle traffic right now?" If it fails, the pod is removed from Service endpoints (stops receiving traffic) but is NOT restarted. Detects: temporary unavailability — the app is booting, warming a cache, a downstream dependency is down. The pod stays running and is re-added when the probe passes again.

**Startup probe** — "Has this container finished starting up?" While the startup probe is active, liveness and readiness probes are disabled. Once it succeeds once, it's done and the other probes take over. Detects: slow startup. Added specifically to solve the problem of apps that take a long time to start (loading large models, warming caches) without needing dangerously high `initialDelaySeconds` on the liveness probe.

**Timing parameters interaction:**

```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 3000
  initialDelaySeconds: 10   # wait 10s before first check
  periodSeconds: 5           # check every 5s
  failureThreshold: 3        # 3 consecutive failures = dead
  timeoutSeconds: 2          # each check must respond within 2s
```

Time to kill = `initialDelaySeconds + (periodSeconds * failureThreshold)` = 10 + (5 * 3) = 25 seconds. The container has 25 seconds from start before a liveness failure triggers a restart.

**The cascading restart trap:**

This is the most dangerous misconfiguration in K8s. Here's how it happens:

1. Your service has a liveness probe hitting `/healthz` with `failureThreshold: 3` and `periodSeconds: 10`
2. A downstream dependency (database) goes slow, causing your `/healthz` to timeout
3. After 30 seconds of failures, K8s restarts the container
4. The new container starts up and immediately hits the same slow dependency
5. It fails liveness again and gets restarted
6. Now multiple pods are in restart loops simultaneously
7. The few pods that are up get all the traffic, overloading them, causing their liveness probes to fail too
8. Entire service goes down — not because of the database, but because of aggressive liveness probes

**How to prevent this:**

- Liveness probes should check the process itself (can it respond at all?), NOT downstream dependencies. Never check database connectivity in a liveness probe.
- Readiness probes should check downstream dependencies — they remove the pod from traffic without restarting it.
- Use a startup probe for slow-starting apps instead of inflating `initialDelaySeconds`.
- Set generous `failureThreshold` and `timeoutSeconds` on liveness probes. Being too aggressive kills healthy pods under temporary load.

```yaml
# Good pattern
startupProbe:
  httpGet: { path: /healthz, port: 3000 }
  failureThreshold: 30
  periodSeconds: 2          # up to 60s to start
livenessProbe:
  httpGet: { path: /healthz, port: 3000 }
  periodSeconds: 10
  failureThreshold: 6       # 60s of failures before restart
readinessProbe:
  httpGet: { path: /ready, port: 3000 }  # checks dependencies
  periodSeconds: 5
  failureThreshold: 3
```

</details>

<details>
<summary>10. Compare rolling updates, blue-green deployments, and canary releases in Kubernetes — what problem does each strategy solve, how do you implement blue-green and canary (since K8s only natively supports rolling updates), and what factors determine which strategy is appropriate for a given service?</summary>

**Rolling update** (K8s native) — gradually replaces old pods with new ones. Configured via `maxSurge` (how many extra pods during update) and `maxUnavailable` (how many pods can be down).

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1          # create 1 extra pod at a time
    maxUnavailable: 0    # never drop below desired count
```

- **Solves:** Zero-downtime deployments with minimal resource overhead.
- **Tradeoff:** During the rollout, both old and new versions serve traffic simultaneously. If the new version has a breaking API change or a database schema change, this mixed-version window causes issues. Rollback is another rolling update (not instant).

**Blue-green deployment** — run two complete environments (blue = current, green = new). Switch traffic all at once by updating the Service selector.

Implementation in K8s:
1. Deploy the new version as a separate Deployment with a different label (e.g., `version: green`)
2. Validate the green Deployment (smoke tests, health checks)
3. Update the Service selector to point to `version: green`
4. Traffic switches instantly
5. Keep the old Deployment around for quick rollback (just flip the selector back)

- **Solves:** No mixed-version traffic. Instant cutover, instant rollback.
- **Tradeoff:** Requires 2x resources during the transition (both full deployments running). Not practical for large, resource-heavy services. No gradual validation — it's all-or-nothing.

**Canary release** — route a small percentage of traffic (e.g., 5%) to the new version. Monitor errors and latency. Gradually increase traffic if everything looks good.

Implementation options:
- **Service mesh (Istio/Linkerd):** Use traffic splitting rules to route a percentage of requests to the canary Deployment. Most flexible — supports weighted routing, header-based routing.
- **Ingress controller (NGINX):** Some Ingress controllers support canary annotations for weight-based routing.
- **Argo Rollouts:** A K8s controller that extends Deployments with canary and blue-green strategies, automated analysis, and progressive delivery.

```yaml
# Argo Rollouts example
apiVersion: argoproj.io/v1alpha1
kind: Rollout
spec:
  strategy:
    canary:
      steps:
        - setWeight: 5
        - pause: { duration: 5m }
        - setWeight: 25
        - pause: { duration: 5m }
        - setWeight: 75
        - pause: { duration: 5m }
```

- **Solves:** Limits blast radius. If the new version has a bug, only 5% of users are affected. Provides real production validation before full rollout.
- **Tradeoff:** Requires traffic splitting infrastructure (mesh or smart Ingress). Needs good observability to detect issues at low traffic percentages. Mixed versions run longer than rolling updates.

**Which strategy to use:**

| Factor | Rolling | Blue-Green | Canary |
|---|---|---|---|
| Resource overhead | Low | High (2x) | Medium |
| Rollback speed | Slow (another rollout) | Instant | Fast (shift traffic back) |
| Mixed versions | Yes, during rollout | No | Yes, intentionally |
| Infrastructure needed | Native K8s | Just labels/selectors | Service mesh or Argo Rollouts |
| Best for | Most services, low-risk changes | Database migrations, breaking changes | High-traffic services, risky changes |

For most teams, rolling updates cover 80% of cases. Add canary releases for critical, high-traffic services. Use blue-green for changes that can't tolerate mixed versions (schema migrations, protocol changes).

</details>

## Practical — Single Resource Configuration

<details>
<summary>11. Configure a Deployment that reads configuration from both a ConfigMap (as environment variables) and a Secret (as a mounted volume) — show the YAML for all three resources, explain when you'd choose env vars vs volume mounts, how you handle multiple environments, and what happens if the ConfigMap changes while pods are running</summary>

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: api-config
  namespace: backend
data:
  LOG_LEVEL: "info"
  API_PORT: "3000"
  CACHE_TTL: "300"
---
apiVersion: v1
kind: Secret
metadata:
  name: api-secrets
  namespace: backend
type: Opaque
data:
  db-password: cGFzc3dvcmQxMjM=    # base64-encoded
  api-key.json: eyJrZXkiOiAidmFsdWUifQ==  # base64-encoded JSON file
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
  namespace: backend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
        - name: api
          image: myapp:1.2.0
          ports:
            - containerPort: 3000
          envFrom:
            - configMapRef:
                name: api-config       # all keys become env vars
          env:
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: api-secrets
                  key: db-password     # single key as env var
          volumeMounts:
            - name: secrets-volume
              mountPath: /etc/secrets
              readOnly: true
      volumes:
        - name: secrets-volume
          secret:
            secretName: api-secrets    # all keys become files
```

**Env vars vs volume mounts — when to use which:**

- **Env vars** — best for simple key-value config (port numbers, log levels, feature flags). Easy to read in code (`process.env.LOG_LEVEL`). Limitation: once the container starts, env vars are frozen — they don't update if the ConfigMap/Secret changes.
- **Volume mounts** — best for structured config files (JSON credentials, TLS certificates, multi-line configs). Each key becomes a file in the mount path. Advantage: kubelet periodically syncs volume-mounted ConfigMaps/Secrets (default ~60s), so changes propagate without pod restarts — but your app must watch the file or re-read it.

**Multi-environment setup:**

Use one ConfigMap/Secret per environment with the same keys but different values. The Deployment YAML stays identical — only the ConfigMap/Secret content differs.

```
# Directory structure for Kustomize or Helm
environments/
  dev/
    configmap.yaml     # LOG_LEVEL: debug
    secret.yaml
  staging/
    configmap.yaml     # LOG_LEVEL: info
    secret.yaml
  production/
    configmap.yaml     # LOG_LEVEL: warn
    secret.yaml
```

With Helm, you'd template the values and use per-environment values files (covered in question 20).

**What happens when a ConfigMap changes while pods are running:**

- **Env vars:** Nothing. The env var was set at container start and is now a static value in the process. The pod must be restarted to pick up changes. Use `kubectl rollout restart deployment/api` to trigger a rolling restart.
- **Volume mounts:** Kubelet updates the mounted files within ~60 seconds (configurable via `--sync-frequency`). But the app must detect the file change — Node.js won't automatically re-read config files.
- **Common pattern for forcing restarts:** Add a hash annotation to the pod template that changes when the ConfigMap changes. Helm does this with `checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}`. When the hash changes, K8s sees it as a pod spec change and triggers a rolling update.

</details>

<details>
<summary>12. When do you use init containers vs the sidecar pattern — show a pod spec that uses an init container to run a database migration before the main app starts, and a sidecar for log shipping or a proxy (e.g., Envoy). Explain the lifecycle differences (init runs to completion sequentially, sidecar runs for the pod's lifetime), what problems each pattern solves, and when you'd use one over the other.</summary>

**Init container — database migration before app starts:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      initContainers:
        - name: run-migrations
          image: myapp:1.2.0
          command: ["node", "dist/migrate.js"]
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: db-secret
                  key: url
        - name: wait-for-redis
          image: busybox:1.36
          command: ["sh", "-c", "until nc -z redis 6379; do sleep 1; done"]
      containers:
        - name: api
          image: myapp:1.2.0
          ports:
            - containerPort: 3000
```

Init containers run sequentially — `run-migrations` must complete successfully before `wait-for-redis` starts, and both must succeed before the main `api` container starts. If an init container fails, the pod restarts and runs all init containers again from the beginning.

**Sidecar — Envoy proxy + log shipper:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: api-pod
spec:
  containers:
    - name: api
      image: myapp:1.2.0
      ports:
        - containerPort: 3000
      volumeMounts:
        - name: logs
          mountPath: /var/log/app
    - name: envoy-proxy
      image: envoyproxy/envoy:v1.28
      ports:
        - containerPort: 8080
      volumeMounts:
        - name: envoy-config
          mountPath: /etc/envoy
    - name: log-shipper
      image: fluent/fluent-bit:2.2
      volumeMounts:
        - name: logs
          mountPath: /var/log/app
          readOnly: true
  volumes:
    - name: logs
      emptyDir: {}
    - name: envoy-config
      configMap:
        name: envoy-config
```

Sidecars are regular containers in the same pod — they start alongside the main container, share the same network namespace (localhost), and run for the pod's entire lifetime. The `log-shipper` reads log files written by the app via a shared `emptyDir` volume. The `envoy-proxy` handles inbound/outbound traffic on behalf of the app.

**Lifecycle differences:**

| Aspect | Init container | Sidecar |
|---|---|---|
| When it runs | Before main containers, sequentially | Alongside main containers, for the pod's lifetime |
| Must complete? | Yes — must exit 0 to proceed | No — runs continuously |
| Failure behavior | Pod restarts, retries all init containers | Pod continues (unless restart policy kills it) |
| Shares network? | Yes (same pod) | Yes (same pod) |
| Resource accounting | Resources are separate from main containers | Counted as part of the pod's total resources |

**When to use which:**

- **Init containers:** One-time setup tasks that must complete before the app runs — migrations, schema creation, downloading config files, waiting for dependencies. The key signal: the task has a defined end.
- **Sidecars:** Ongoing concerns that run alongside the app — proxying (Envoy, Istio), log collection (Fluent Bit), monitoring agents, TLS certificate rotation. The key signal: the task runs continuously for the pod's lifetime.

**Gotcha with migrations in init containers:** If you have 3 replicas, all 3 pods run the init container. Make sure your migration script is idempotent — it should be safe to run concurrently or multiple times. Alternatively, run migrations as a separate Job (as covered in question 7) rather than an init container, so it runs exactly once regardless of replica count.

**Native sidecars (Kubernetes 1.28+):** K8s 1.28 introduced native sidecar support via `restartPolicy: Always` on init containers. These "sidecar containers" start before the main containers (like init containers) but run for the pod's entire lifetime (like traditional sidecars). This solves the longstanding issue where traditional sidecars (regular containers) had no guaranteed startup ordering and no clean shutdown coordination with the main container. For new clusters on 1.28+, prefer native sidecars over the traditional pattern for proxies, log shippers, and monitoring agents.

</details>

<details>
<summary>13. Configure resource requests and limits for a Deployment, then set up a LimitRange and ResourceQuota for a namespace — show the YAML for all three, explain the difference between requests and limits, what happens when a container exceeds its memory limit vs CPU limit, how requests and limits determine a pod's QoS class (Guaranteed, Burstable, BestEffort) and its eviction priority, and why getting requests wrong causes scheduling failures and noisy-neighbor issues</summary>

**Deployment with resource requests and limits:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
  namespace: backend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
        - name: api
          image: myapp:1.2.0
          resources:
            requests:
              cpu: "250m"       # 0.25 CPU cores — scheduler guarantee
              memory: "256Mi"   # 256 MiB — scheduler guarantee
            limits:
              cpu: "500m"       # 0.5 CPU cores — hard ceiling
              memory: "512Mi"   # 512 MiB — hard ceiling (OOMKilled if exceeded)
```

**LimitRange — default and max per container in the namespace:**

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: container-limits
  namespace: backend
spec:
  limits:
    - type: Container
      default:           # applied if no limits specified
        cpu: "500m"
        memory: "512Mi"
      defaultRequest:    # applied if no requests specified
        cpu: "100m"
        memory: "128Mi"
      max:               # no container can exceed these
        cpu: "2"
        memory: "2Gi"
      min:               # no container can request less than these
        cpu: "50m"
        memory: "64Mi"
```

**ResourceQuota — total budget for the entire namespace:**

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: backend-quota
  namespace: backend
spec:
  hard:
    requests.cpu: "8"        # total CPU requests across all pods
    requests.memory: "16Gi"
    limits.cpu: "16"
    limits.memory: "32Gi"
    pods: "50"               # max 50 pods in this namespace
```

**Requests vs limits:**

- **Requests** = what the scheduler guarantees. The scheduler only places a pod on a node that has enough unreserved resources to satisfy the request. This is the "minimum" the container is promised.
- **Limits** = the hard ceiling the container cannot exceed. Enforced by the kernel (cgroups).

**What happens when limits are exceeded:**

- **Memory limit exceeded:** The kernel OOMKills the container immediately (exit code 137). Memory is an incompressible resource — you can't slow down memory usage, so the only option is to kill the process.
- **CPU limit exceeded:** The container is throttled — it's forced to wait. It still runs, just slower. CPU is a compressible resource, so the kernel can limit how much CPU time the container gets per scheduling period. The app doesn't crash, but it becomes sluggish.

**QoS classes and eviction priority:**

| QoS class | Condition | Eviction priority |
|---|---|---|
| **Guaranteed** | Every container has requests = limits for both CPU and memory | Last to be evicted (highest priority) |
| **Burstable** | At least one container has requests set, but requests != limits | Evicted after BestEffort |
| **BestEffort** | No requests or limits set on any container | First to be evicted (lowest priority) |

When a node runs low on memory, kubelet evicts pods in order: BestEffort first, then Burstable (starting with those using the most above their request), then Guaranteed last.

**Why wrong requests cause problems:**

- **Requests too high:** The scheduler thinks the node is full when it isn't. Pods stay `Pending` even though the node has plenty of actual free resources. This wastes cluster capacity.
- **Requests too low:** The scheduler packs too many pods onto a node. They all fit on paper, but when they actually use resources, they compete for CPU (throttling) and memory (OOMKills). This is the noisy-neighbor problem — one pod's burst starves others on the same node.
- **No requests at all (BestEffort):** The pod gets whatever is leftover and is first to be evicted under pressure. Never do this in production.

**Practical guidance:** Set requests to the p95 actual usage (what the container normally uses). Set memory limits to 1.5-2x the request to handle spikes. For CPU limits, many teams now omit them entirely — CPU throttling causes latency spikes that are harder to debug than letting pods burst. Use LimitRanges to enforce sane defaults so no one accidentally deploys without requests.

</details>

<details>
<summary>14. Configure RBAC so a CI/CD pipeline's service account can deploy to a specific namespace but nothing else — show the ServiceAccount, Role, and RoleBinding YAML, explain the difference between Role and ClusterRole, and identify the common over-permissioning mistakes teams make (wildcard verbs, cluster-admin for CI, overly broad ServiceAccounts)</summary>

```yaml
# 1. ServiceAccount for the CI/CD pipeline
apiVersion: v1
kind: ServiceAccount
metadata:
  name: ci-deployer
  namespace: production
---
# 2. Role — scoped to the 'production' namespace
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: deployer
  namespace: production
rules:
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
  - apiGroups: [""]
    resources: ["services", "configmaps"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]     # read-only — CI shouldn't delete pods directly
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get", "list"]              # read existing secrets, not create new ones
---
# 3. RoleBinding — connects the ServiceAccount to the Role
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: ci-deployer-binding
  namespace: production
subjects:
  - kind: ServiceAccount
    name: ci-deployer
    namespace: production
roleRef:
  kind: Role
  name: deployer
  apiGroup: rbac.authorization.k8s.io
```

**Role vs ClusterRole:**

- **Role** — scoped to a single namespace. Can only grant permissions on resources within that namespace. Use for: namespace-specific access like deploying to `production`.
- **ClusterRole** — cluster-wide. Can grant permissions on cluster-scoped resources (nodes, PersistentVolumes, namespaces themselves) or be reused across namespaces via RoleBindings. Use for: cluster admins, resources that aren't namespaced, or defining a reusable set of permissions that multiple RoleBindings reference.

A ClusterRole can be bound to a namespace via a RoleBinding (granting its permissions only in that namespace) or via a ClusterRoleBinding (granting permissions cluster-wide). This is a common pattern: define a ClusterRole once, bind it per-namespace as needed.

**Common over-permissioning mistakes:**

1. **`cluster-admin` for CI** — the nuclear option. Grants full access to everything in the cluster. If CI credentials leak, the attacker owns the entire cluster. Always scope CI to the specific namespace and resources it needs.

2. **Wildcard verbs (`verbs: ["*"]`)** — grants every action including `delete`, `deletecollection`, `escalate`. CI should not be able to delete all pods or modify RBAC rules. List explicit verbs.

3. **Wildcard resources (`resources: ["*"]`)** — grants access to secrets, RBAC objects, and everything else in the namespace. CI probably doesn't need to read secrets or modify network policies.

4. **Using the `default` ServiceAccount** — every namespace has a `default` ServiceAccount that all pods use unless specified. If you give it deploy permissions, every pod in that namespace inherits those permissions. Always create dedicated ServiceAccounts for specific purposes.

5. **ClusterRoleBinding when RoleBinding would suffice** — binding a deployer role cluster-wide when CI only needs access to one namespace. This means CI can deploy to every namespace, including `kube-system`.

6. **Not auditing RBAC regularly** — permissions accumulate. The CI pipeline that needed `secrets` access for one migration still has it months later. Use `kubectl auth can-i --list --as=system:serviceaccount:production:ci-deployer` to audit what a ServiceAccount can actually do.

</details>

<details>
<summary>15. Configure pod scheduling using taints, tolerations, and node affinity — show the YAML for a scenario where certain pods must run on GPU nodes while others must avoid them, explain the difference between required and preferred affinity rules, and how pod priorities and preemption work when the cluster is resource-constrained</summary>

**Step 1: Taint the GPU nodes** so non-GPU workloads are repelled:

```bash
kubectl taint nodes gpu-node-1 gpu=true:NoSchedule
kubectl taint nodes gpu-node-2 gpu=true:NoSchedule
kubectl label nodes gpu-node-1 gpu-node-2 hardware=gpu
```

The taint `gpu=true:NoSchedule` prevents any pod from scheduling on these nodes unless it explicitly tolerates the taint. The label `hardware=gpu` is used for affinity-based targeting.

**Step 2: GPU workload — toleration + node affinity to target GPU nodes:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ml-training
spec:
  replicas: 2
  selector:
    matchLabels:
      app: ml-training
  template:
    metadata:
      labels:
        app: ml-training
    spec:
      tolerations:
        - key: "gpu"
          operator: "Equal"
          value: "true"
          effect: "NoSchedule"
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: hardware
                    operator: In
                    values: ["gpu"]
      containers:
        - name: training
          image: ml-training:latest
          resources:
            limits:
              nvidia.com/gpu: 1
```

The toleration allows the pod to schedule on tainted GPU nodes. The `requiredDuringScheduling` affinity ensures it ONLY runs on GPU nodes. Without both, the pod could end up on a non-GPU node (toleration alone doesn't force placement — it just removes the restriction).

**Step 3: Regular workload — no toleration, so it automatically avoids GPU nodes:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  # ... replicas, selector, template.metadata.labels omitted for brevity
  template:
    spec:
      affinity:
        nodeAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              preference:
                matchExpressions:
                  - key: hardware
                    operator: NotIn
                    values: ["gpu"]
      containers:
        - name: api
          image: myapp:1.2.0
```

Since this pod has no toleration for the `gpu` taint, it's already blocked from GPU nodes by the taint alone. The preferred affinity is a belt-and-suspenders approach — it adds a soft preference to avoid GPU-labeled nodes even if the taint is ever removed.

**Required vs preferred affinity:**

- **`requiredDuringSchedulingIgnoredDuringExecution`** — hard rule. The pod will NOT schedule if no node matches. It stays `Pending` forever. Use when placement is a correctness requirement (GPU workload must have GPU).
- **`preferredDuringSchedulingIgnoredDuringExecution`** — soft preference with a weight (1-100). The scheduler tries to satisfy it but will place the pod elsewhere if no matching node has capacity. Use for optimization hints (prefer nodes in the same AZ as the database, but don't fail if unavailable).

The `IgnoredDuringExecution` part means already-running pods aren't evicted if the node stops matching. If you relabel a node, existing pods stay put.

**Pod priorities and preemption:**

When the cluster is full, pod priorities determine which pods survive:

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: critical
value: 1000000
globalDefault: false
description: "For production-critical services"
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: batch
value: 100
globalDefault: false
```

```yaml
# High-priority pod
spec:
  priorityClassName: critical
```

**How preemption works:** If a high-priority pod can't schedule because no node has enough resources, the scheduler looks for a node where evicting lower-priority pods would free enough resources. It evicts the lowest-priority pods necessary and schedules the high-priority pod in their place. The evicted pods go back to `Pending` and wait for capacity.

**Gotcha:** Preemption can cause cascading evictions if not carefully designed. A sudden burst of high-priority pods can evict many lower-priority ones. Set priorities intentionally — most workloads should share a default priority, with only truly critical services elevated.

</details>

## Practical — Multi-Resource Composition

<details>
<summary>16. Set up a HorizontalPodAutoscaler that scales on both CPU utilization and a custom metric (e.g., queue depth) — show the YAML, explain how HPA calculates the desired replica count using the scaling formula, and what happens when multiple metrics disagree on the target replica count</summary>

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: worker-hpa
  namespace: backend
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: worker
  minReplicas: 2
  maxReplicas: 20
  metrics:
    # Scale on CPU utilization
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70    # target 70% CPU across all pods
    # Scale on custom metric (queue depth from Prometheus/KEDA)
    - type: Pods
      pods:
        metric:
          name: sqs_queue_depth     # exposed via custom metrics API
        target:
          type: AverageValue
          averageValue: "30"        # target 30 messages per pod
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300  # wait 5min before scaling down
      policies:
        - type: Percent
          value: 25                    # scale down max 25% at a time
          periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0    # scale up immediately
      policies:
        - type: Pods
          value: 4                     # add max 4 pods at a time
          periodSeconds: 60
```

**How HPA calculates desired replicas:**

The formula for each metric:

```
desiredReplicas = ceil(currentReplicas * (currentMetricValue / targetMetricValue))
```

Example with CPU: if you have 3 replicas at 90% average CPU utilization with a target of 70%:
```
desiredReplicas = ceil(3 * (90 / 70)) = ceil(3 * 1.286) = ceil(3.857) = 4
```

Example with queue depth: if queue has 200 messages, 3 replicas, target is 30 per pod:
```
currentAverageValue = 200 / 3 = 66.7 messages per pod
desiredReplicas = ceil(3 * (66.7 / 30)) = ceil(3 * 2.22) = ceil(6.67) = 7
```

**When multiple metrics disagree:**

HPA evaluates each metric independently and takes the **maximum** recommended replica count. This is a "scale to the most demanding metric" strategy.

In the example above: CPU says 4 replicas, queue depth says 7 replicas. HPA scales to 7. This ensures no metric is under-served — the cost is potentially over-provisioning for the less demanding metric.

**Key details:**

- HPA checks metrics every 15 seconds by default (configurable via `--horizontal-pod-autoscaler-sync-period`).
- The `behavior` section controls scaling velocity. The `stabilizationWindowSeconds` for scale-down prevents flapping — HPA looks at the highest recommended replica count over the window and won't scale below it. This avoids the pattern of scaling down, immediately scaling back up, then down again.
- Custom metrics require a metrics adapter (Prometheus Adapter, KEDA, or Datadog Cluster Agent) that exposes metrics through the Kubernetes custom metrics API (`custom.metrics.k8s.io`).
- HPA requires resource requests to be set on the target Deployment for resource-type metrics. Without requests, HPA can't calculate utilization percentages.

**Gotcha:** HPA and VPA (Vertical Pod Autoscaler) can conflict. HPA scales horizontally based on utilization percentages, but VPA changes the resource requests, which changes the denominator in HPA's calculation. Don't use both on the same metric — use HPA for scaling and VPA only in recommendation mode to inform request tuning.

</details>

<details>
<summary>17. Configure a zero-downtime deployment — show the Deployment spec with rolling update strategy (maxUnavailable/maxSurge), a PodDisruptionBudget, and a pod spec with a preStop hook and proper connection draining. Explain why you need each piece, how to set terminationGracePeriodSeconds correctly, and what happens to in-flight requests without these configurations</summary>

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
  namespace: production
spec:
  replicas: 4
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0    # never drop below 4 pods
      maxSurge: 1          # create 1 new pod before removing an old one
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      terminationGracePeriodSeconds: 45   # total time K8s waits before force-killing
      containers:
        - name: api
          image: myapp:2.0.0
          ports:
            - containerPort: 3000
          readinessProbe:
            httpGet:
              path: /ready
              port: 3000
            periodSeconds: 5
            failureThreshold: 3
          lifecycle:
            preStop:
              exec:
                command: ["sh", "-c", "sleep 10"]  # delay to let endpoints update
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: api-pdb
  namespace: production
spec:
  minAvailable: 3          # at least 3 of 4 pods must always be running
  selector:
    matchLabels:
      app: api
```

**Why each piece is needed:**

**Rolling update with `maxUnavailable: 0`** — ensures the old pod isn't removed until the new pod is Ready. With `maxSurge: 1`, K8s creates one new pod, waits for its readiness probe to pass, then terminates one old pod. At no point does capacity drop below the desired replica count.

**Readiness probe** — tells K8s when a new pod is actually ready to receive traffic. Without it, K8s adds the pod to Service endpoints immediately on start, before it's ready. Requests hit a not-yet-initialized app and fail.

**preStop hook (`sleep 10`)** — this is the critical piece most people miss. When K8s decides to terminate a pod, two things happen simultaneously:
1. The pod is removed from Service endpoints (iptables/IPVS rules updated)
2. The preStop hook runs, then SIGTERM is sent

The problem: endpoint removal is **asynchronous**. It takes time for kube-proxy to update iptables rules on every node, and for Ingress controllers to update their backends. During this window, traffic is still being routed to a pod that's shutting down. The `sleep 10` in preStop gives the cluster time to propagate the endpoint removal before the app starts shutting down.

**`terminationGracePeriodSeconds: 45`** — the total time budget from when K8s initiates termination to when it force-kills (SIGKILL) the container. The timeline:

```
0s  — preStop starts (sleep 10)
10s — preStop ends, SIGTERM sent to the app
10s-45s — app drains connections and shuts down gracefully
45s — SIGKILL if still running
```

Set this to: `preStop duration + max expected request duration + buffer`. If your longest request takes 30s and preStop is 10s, set it to at least 45s.

**PodDisruptionBudget (PDB)** — protects against voluntary disruptions (node drains, cluster upgrades, autoscaler scale-downs). With `minAvailable: 3`, K8s won't drain a node if it would bring the available pod count below 3. Without a PDB, a node drain could terminate multiple pods simultaneously, causing downtime.

**What happens without these configurations:**

- **No readiness probe:** New pods receive traffic before they're ready. Users get connection refused or 502 errors during every deployment.
- **No preStop hook:** The app receives SIGTERM and starts shutting down while traffic is still being routed to it. In-flight requests get connection reset errors. The faster the shutdown, the worse the problem.
- **No PDB:** A cluster upgrade drains a node and kills 2 of your 4 pods simultaneously. If `maxUnavailable: 0` applies only to rolling updates, not node drains.
- **`terminationGracePeriodSeconds` too low:** K8s force-kills the container before it finishes draining connections. Long-running requests are aborted mid-flight.

**App-side connection draining (Node.js example):**

```typescript
process.on('SIGTERM', () => {
  console.log('SIGTERM received, draining connections...');
  server.close(() => {
    // All connections closed, exit cleanly
    process.exit(0);
  });
  // Force exit if draining takes too long
  setTimeout(() => process.exit(1), 30_000);
});
```

</details>

<details>
<summary>18. Set up an Ingress with TLS termination using cert-manager — show the Ingress YAML, the Certificate and Issuer resources, explain the ACME flow (how cert-manager obtains and renews a Let's Encrypt certificate via HTTP-01 or DNS-01 challenge), and how to add rate limiting via Ingress annotations.</summary>

**ClusterIssuer for Let's Encrypt:**

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: platform@mycompany.com
    privateKeySecretRef:
      name: letsencrypt-prod-key
    solvers:
      - http01:
          ingress:
            class: nginx
      # Alternative: DNS-01 for wildcard certs or private clusters
      # - dns01:
      #     route53:
      #       region: us-east-1
      #       hostedZoneID: Z1234567890
```

**Ingress with TLS and rate limiting (NGINX Ingress Controller):**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
  namespace: production
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    # Rate limiting (NGINX Ingress specific)
    nginx.ingress.kubernetes.io/limit-rps: "50"           # 50 requests/sec per IP
    nginx.ingress.kubernetes.io/limit-burst-multiplier: "5" # allow burst to 250
    nginx.ingress.kubernetes.io/limit-connections: "10"     # max 10 concurrent connections per IP
    # Other useful annotations
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - api.mycompany.com
      secretName: api-tls-cert    # cert-manager creates and manages this Secret
  rules:
    - host: api.mycompany.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api
                port:
                  number: 3000
```

When cert-manager sees the `cert-manager.io/cluster-issuer` annotation, it automatically creates a Certificate resource. You can also create it explicitly:

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: api-tls-cert
  namespace: production
spec:
  secretName: api-tls-cert
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
    - api.mycompany.com
    - "*.api.mycompany.com"   # wildcard requires DNS-01
```

**The ACME flow:**

ACME (Automatic Certificate Management Environment) is the protocol Let's Encrypt uses to verify you own a domain before issuing a certificate.

**HTTP-01 challenge:**
1. cert-manager requests a certificate from Let's Encrypt for `api.mycompany.com`
2. Let's Encrypt responds with a random token
3. cert-manager creates a temporary pod and Ingress rule that serves the token at `http://api.mycompany.com/.well-known/acme-challenge/<token>`
4. Let's Encrypt makes an HTTP request to that URL from the public internet
5. If the token matches, Let's Encrypt issues the certificate
6. cert-manager stores the cert + private key in the Secret (`api-tls-cert`)
7. cert-manager cleans up the temporary resources
8. cert-manager renews automatically ~30 days before expiry

**DNS-01 challenge:**
1. Same initial request, but instead of HTTP, cert-manager creates a TXT DNS record (`_acme-challenge.api.mycompany.com`)
2. Let's Encrypt queries DNS for the record
3. If it matches, the cert is issued

**When to use which:**
- **HTTP-01:** Simplest. Works for any domain reachable from the public internet. Cannot issue wildcard certificates.
- **DNS-01:** Required for wildcard certs (`*.mycompany.com`). Works for domains not publicly accessible (internal services). Requires cert-manager to have permissions to modify DNS records (Route53, CloudDNS, etc.).

**Rate limiting notes:** The annotations shown are NGINX Ingress-specific. Other Ingress controllers (Traefik, HAProxy) have different annotation schemes. For more sophisticated rate limiting (per-user, per-API-key), you'd typically use an API gateway or service mesh instead of Ingress annotations.

</details>

<details>
<summary>19. Why does Kubernetes default to allow-all traffic between pods and what does that mean for security? Write a default-deny NetworkPolicy and then add rules that allow specific traffic paths (e.g., frontend → API on port 3000, API → database on port 5432) — show the YAML and explain the common mistakes teams make when first adopting Network Policies</summary>

**Why allow-all is the default:** Kubernetes prioritizes ease of getting started. The flat networking model (covered in question 3) means every pod can reach every other pod — this makes service discovery simple and eliminates networking as a barrier to deploying applications. Security is opt-in, not opt-out.

**What this means for security:** Any compromised pod can reach every other pod in the cluster, across all namespaces. An attacker who gains code execution in a frontend pod can directly connect to your database, internal admin services, or other tenants' workloads. There's no network-level segmentation by default.

**Step 1: Default-deny all ingress and egress in a namespace:**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}          # empty = applies to ALL pods in the namespace
  policyTypes:
    - Ingress
    - Egress
```

Once applied, no pod in `production` can send or receive any traffic — not even DNS. You must now explicitly allow every traffic path.

**Step 2: Allow DNS egress (critical — without this, nothing works):**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
```

**Step 3: Allow frontend to API on port 3000:**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-api
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: api                 # this policy applies to API pods
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend    # only frontend pods can connect
      ports:
        - protocol: TCP
          port: 3000
```

**Step 4: Allow API to database on port 5432:**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-api-to-db
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: api                 # this policy applies to API pods
  policyTypes:
    - Egress
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: database
      ports:
        - protocol: TCP
          port: 5432
```

**Step 5: Allow API egress to external services (if needed):**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-api-external
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
    - Egress
  egress:
    - to:
        - ipBlock:
            cidr: 0.0.0.0/0
            except:
              - 10.0.0.0/8      # block internal ranges to prevent lateral movement
              - 172.16.0.0/12
              - 192.168.0.0/16
      ports:
        - protocol: TCP
          port: 443
```

**Common mistakes teams make:**

1. **Forgetting DNS egress** — default-deny blocks DNS (port 53 to CoreDNS). Pods can't resolve service names. Everything appears broken with confusing timeout errors. Always add DNS egress as the first policy after default-deny.

2. **CNI plugin doesn't support NetworkPolicy** — Flannel, the default CNI in many setups, does NOT enforce NetworkPolicies. You create policies, they're accepted by the API server, but they do nothing. You need Calico, Cilium, or another CNI with policy enforcement. This is a silent failure — you think you're protected but you're not.

3. **Applying default-deny to `kube-system`** — breaks CoreDNS, metrics-server, and control plane components. Start with application namespaces, not system namespaces.

4. **Mixing `podSelector` and `namespaceSelector` incorrectly** — a single `from` entry with both selectors is an AND (pod must match both). Separate entries in the `from` array are OR. This subtle difference causes policies that are either too restrictive or too permissive:

   ```yaml
   # AND — pod must be in namespace X AND have label Y
   - from:
       - namespaceSelector: { matchLabels: { env: prod } }
         podSelector: { matchLabels: { app: frontend } }

   # OR — any pod in namespace X OR any pod with label Y (anywhere)
   - from:
       - namespaceSelector: { matchLabels: { env: prod } }
       - podSelector: { matchLabels: { app: frontend } }
   ```

5. **Not testing with actual traffic** — use `kubectl exec` to test connectivity from source pods: `kubectl exec -it frontend-pod -- curl -v api:3000`. Don't assume the policy works because `kubectl apply` succeeded.

</details>

## Practical — Helm & Tooling

<details>
<summary>20. Walk through the structure of a Helm chart — show the directory layout, explain what Chart.yaml, values.yaml, and templates/ contain, demonstrate how values overrides work (--set vs -f values files), and how you manage different configurations across environments (dev, staging, production)</summary>

**Directory layout:**

```
my-api/
  Chart.yaml              # chart metadata (name, version, dependencies)
  values.yaml             # default configuration values
  values-dev.yaml         # environment override: dev
  values-staging.yaml     # environment override: staging
  values-prod.yaml        # environment override: production
  templates/
    deployment.yaml       # Deployment template
    service.yaml          # Service template
    ingress.yaml          # Ingress template
    configmap.yaml        # ConfigMap template
    hpa.yaml              # HPA template
    _helpers.tpl          # reusable template snippets (labels, names)
    NOTES.txt             # post-install instructions (printed after helm install)
  charts/                 # dependency charts (downloaded)
```

**Chart.yaml — chart metadata:**

```yaml
apiVersion: v2
name: my-api
version: 1.3.0          # chart version (changes with chart structure changes)
appVersion: "2.1.0"     # version of the app being deployed
description: My API service
dependencies:
  - name: redis
    version: "18.x.x"
    repository: https://charts.bitnami.com/bitnami
    condition: redis.enabled
```

**values.yaml — default values:**

```yaml
replicaCount: 2
image:
  repository: myapp
  tag: "latest"
  pullPolicy: IfNotPresent
service:
  type: ClusterIP
  port: 3000
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    memory: 256Mi
ingress:
  enabled: false
  host: ""
autoscaling:
  enabled: false
  minReplicas: 2
  maxReplicas: 10
redis:
  enabled: false
```

**templates/deployment.yaml — uses Go templating:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "my-api.fullname" . }}
  labels:
    {{- include "my-api.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "my-api.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "my-api.selectorLabels" . | nindent 8 }}
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          ports:
            - containerPort: {{ .Values.service.port }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
```

The `checksum/config` annotation forces a rolling restart when the ConfigMap changes (as discussed in question 11).

**templates/_helpers.tpl — reusable template snippets:**

```yaml
{{- define "my-api.fullname" -}}
{{- printf "%s-%s" .Release.Name .Chart.Name | trunc 63 | trimSuffix "-" -}}
{{- end -}}

{{- define "my-api.selectorLabels" -}}
app.kubernetes.io/name: {{ .Chart.Name }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end -}}

{{- define "my-api.labels" -}}
{{ include "my-api.selectorLabels" . }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end -}}
```

These helpers are referenced via `{{ include "my-api.fullname" . }}` in the templates above. Centralizing labels and names here keeps templates DRY and ensures consistency across all resources in the chart.

**Values overrides — `--set` vs `-f`:**

```bash
# -f: override with a file (preferred for complex/repeatable config)
helm upgrade --install my-api ./my-api \
  -f values-prod.yaml \
  --namespace production

# --set: override individual values (good for CI, single overrides)
helm upgrade --install my-api ./my-api \
  -f values-prod.yaml \
  --set image.tag="2.1.0" \
  --namespace production

# Priority (last wins): values.yaml < -f file < --set
```

Use `-f` for environment-specific files. Use `--set` for dynamic values like the image tag from CI. Avoid `--set` for complex nested structures — it gets unreadable.

**Environment-specific values files:**

```yaml
# values-dev.yaml
replicaCount: 1
image:
  tag: "dev-latest"
resources:
  requests:
    cpu: 50m
    memory: 64Mi
  limits:
    memory: 128Mi
ingress:
  enabled: true
  host: api.dev.mycompany.com

# values-prod.yaml
replicaCount: 4
image:
  tag: "2.1.0"
resources:
  requests:
    cpu: 500m
    memory: 512Mi
  limits:
    memory: 1Gi
ingress:
  enabled: true
  host: api.mycompany.com
autoscaling:
  enabled: true
  minReplicas: 4
  maxReplicas: 20
redis:
  enabled: true
```

**CI/CD deployment pattern:**

```bash
# In CI pipeline
helm upgrade --install my-api ./my-api \
  -f values-${ENVIRONMENT}.yaml \
  --set image.tag="${GIT_SHA}" \
  --namespace ${ENVIRONMENT} \
  --wait --timeout 5m
```

The `--wait` flag makes Helm wait until all pods are Ready before marking the release as successful. If the readiness probe never passes (bad image, misconfiguration), Helm reports failure and CI fails.

**Helm vs Kustomize (brief comparison):**
- **Helm:** Full templating engine, package management, release history, rollback support. Better for reusable charts shared across teams or published publicly. More complex.
- **Kustomize:** No templating — uses patches and overlays on plain YAML. Built into kubectl (`kubectl apply -k`). Simpler, easier to understand, and YAML stays valid. Better for teams that own their manifests and just need per-environment overrides.

Many teams use both: Helm for third-party charts (NGINX Ingress, cert-manager, Prometheus) and Kustomize for their own application manifests.

</details>

## Practical — Debugging & Troubleshooting

<details>
<summary>21. A pod is stuck in CrashLoopBackOff — walk through the exact kubectl commands to diagnose the root cause step by step: checking pod status, reading events, pulling logs from the previous crashed container, interpreting exit codes, and using ephemeral containers (kubectl debug) when the container has no shell or debugging tools. What are the most common causes and how do you fix each?</summary>

CrashLoopBackOff means the container keeps crashing and K8s keeps restarting it, with exponentially increasing backoff delays (10s, 20s, 40s, up to 5 minutes).

**Step 1: Check pod status and restart count**

```bash
kubectl get pods -n production
# NAME      READY   STATUS             RESTARTS      AGE
# api-xyz   0/1     CrashLoopBackOff   7 (2m ago)    15m

kubectl describe pod api-xyz -n production
```

Look at the `Last State` section in describe output:

```
Last State:  Terminated
  Reason:    Error
  Exit Code: 1          # application error
  # or Exit Code: 137   # OOMKilled (covered in question 23)
  # or Exit Code: 139   # segfault
```

**Step 2: Read events**

Events at the bottom of `kubectl describe pod` show the timeline:

```
Events:
  Warning  BackOff  2m   kubelet  Back-off restarting failed container
  Normal   Pulled   3m   kubelet  Container image "myapp:2.0.0" already present
  Normal   Created  3m   kubelet  Created container api
  Normal   Started  3m   kubelet  Started container api
```

Look for: image pull errors, volume mount failures, or liveness probe failures triggering restarts.

**Step 3: Pull logs from the crashed container**

```bash
# Current container logs (might be empty if it crashes immediately)
kubectl logs api-xyz -n production

# Logs from the PREVIOUS crashed container (this is what you usually need)
kubectl logs api-xyz -n production --previous

# If the pod has multiple containers
kubectl logs api-xyz -n production -c api --previous
```

**Step 4: Interpret exit codes**

| Exit code | Meaning | Common cause |
|---|---|---|
| 0 | Success (but container exited) | Main process ended — wrong command or entrypoint |
| 1 | Application error | Unhandled exception, missing env var, bad config |
| 126 | Permission denied | Binary not executable |
| 127 | Command not found | Wrong entrypoint/command in Dockerfile or pod spec |
| 137 | SIGKILL (OOMKilled) | Memory limit exceeded (see question 23) |
| 139 | SIGSEGV | Segmentation fault — native module crash |
| 143 | SIGTERM | Graceful shutdown (not usually a crashloop cause) |

**Step 5: Use ephemeral containers when the image has no shell**

Distroless or minimal images don't include `sh`, `bash`, or debugging tools. Use `kubectl debug` to attach an ephemeral container with tools:

```bash
# Attach a debug container to the running (or crashing) pod
kubectl debug -it api-xyz -n production --image=busybox:1.36 --target=api

# This gives you a shell in a new container that shares the pod's:
# - network namespace (can curl localhost:3000)
# - process namespace (can see the app's processes with ps)

# Or copy the pod with a different command to keep it alive for debugging
kubectl debug api-xyz -n production --copy-to=api-debug --container=api -- sh
# This creates a new pod 'api-debug' with the same spec but overridden command
```

**Most common causes and fixes:**

1. **Missing environment variable / bad config** — app crashes at startup because a required env var isn't set or a config file is missing. Fix: check ConfigMap/Secret references, verify the env var names match what the app expects.

2. **Image issue** — wrong tag, entrypoint doesn't exist, binary compiled for wrong architecture. Fix: verify image tag, check Dockerfile entrypoint, ensure amd64 vs arm64 matches your nodes.

3. **Liveness probe killing healthy containers** — the probe is too aggressive and kills containers that are still starting up. Fix: add a startup probe or increase `initialDelaySeconds`/`failureThreshold` (as discussed in question 9).

4. **OOMKilled (exit code 137)** — memory limit too low for the application. Fix: increase memory limit or investigate the memory leak (covered in question 23).

5. **Dependency not available** — app tries to connect to a database or service at startup and crashes if it can't. Fix: use init containers to wait for dependencies (as shown in question 12), or make the app resilient to missing dependencies at startup.

6. **Read-only filesystem or permission errors** — the container tries to write to a path that's read-only. Fix: add an `emptyDir` volume for writable paths, or set the correct `securityContext`.

</details>

<details>
<summary>22. A pod has been Pending for 10 minutes and won't schedule — walk through the exact steps to diagnose why: checking scheduler events, identifying resource constraints vs node selector mismatches vs taint issues vs PVC binding failures. What are the common fixes for each cause?</summary>

A pod in `Pending` means the scheduler hasn't found a suitable node. The reason is almost always in the pod's events.

**Step 1: Check events on the pod**

```bash
kubectl describe pod api-xyz -n production
```

The Events section will contain the scheduler's reason. This is your primary diagnostic — every Pending cause produces a different message.

**Step 2: Diagnose by event message**

**Cause 1: Insufficient resources**

```
Events:
  Warning  FailedScheduling  default-scheduler
  0/5 nodes are available: 5 Insufficient cpu, 3 Insufficient memory.
```

The pod's resource requests exceed available capacity on all nodes. "Available" means unreserved capacity (total - sum of all other pods' requests), NOT actual usage.

```bash
# Check node capacity and allocations
kubectl describe node <node-name> | grep -A 10 "Allocated resources"
# Shows requests vs allocatable for each node

# Check what the pod is requesting
kubectl get pod api-xyz -n production -o yaml | grep -A 5 requests
```

**Fixes:**
- Reduce the pod's resource requests if they're over-provisioned
- Scale up the cluster (add nodes or increase node size)
- Evict lower-priority pods if priorities are configured (covered in question 15)
- Check if other pods have inflated requests wasting capacity (as discussed in question 13)

**Cause 2: Node selector or affinity mismatch**

```
Events:
  Warning  FailedScheduling  default-scheduler
  0/5 nodes are available: 5 node(s) didn't match Pod's node affinity/selector.
```

```bash
# Check what the pod requires
kubectl get pod api-xyz -o yaml | grep -A 10 nodeSelector
kubectl get pod api-xyz -o yaml | grep -A 20 affinity

# Check node labels
kubectl get nodes --show-labels
```

**Fixes:**
- Label the appropriate nodes: `kubectl label node <node> <key>=<value>`
- Fix the pod's nodeSelector or affinity to match existing labels
- If using `requiredDuringScheduling`, consider switching to `preferred` if strict placement isn't necessary

**Cause 3: Taints not tolerated**

```
Events:
  Warning  FailedScheduling  default-scheduler
  0/5 nodes are available: 5 node(s) had taints that the pod didn't tolerate.
```

```bash
# Check node taints
kubectl get nodes -o custom-columns=NAME:.metadata.name,TAINTS:.spec.taints

# Check pod tolerations
kubectl get pod api-xyz -o yaml | grep -A 10 tolerations
```

**Fixes:**
- Add a toleration to the pod spec (if the pod should run on tainted nodes)
- Remove the taint from nodes: `kubectl taint node <node> <key>:<effect>-`
- Common surprise: a node in `NotReady` state automatically gets a `node.kubernetes.io/not-ready:NoSchedule` taint

**Cause 4: PVC binding failure**

```
Events:
  Warning  FailedScheduling  default-scheduler
  0/5 nodes are available: 5 node(s) didn't find available persistent volumes to bind.
```

```bash
# Check PVC status
kubectl get pvc -n production
# NAME       STATUS    VOLUME   CAPACITY   STORAGECLASS
# data-pvc   Pending                       fast-ssd

# Check PVC events
kubectl describe pvc data-pvc -n production
```

**Fixes:**
- If using dynamic provisioning: verify the StorageClass exists and the provisioner is running
- If using `WaitForFirstConsumer` binding mode: this is normal — the PV is created only after the pod is assigned to a node. Check if other scheduling constraints are preventing node assignment.
- AZ mismatch: the PV exists in `us-east-1a` but the pod can only schedule on nodes in `us-east-1b`. Fix: use `WaitForFirstConsumer` to provision storage in the same AZ as the selected node.
- Check if ResourceQuota is blocking PVC creation: `kubectl describe resourcequota -n production`

**Cause 5: Too many pods**

```
Events:
  Warning  FailedScheduling  default-scheduler
  0/5 nodes are available: 5 node(s) exceed max pod count.
```

Each node has a max pod limit (default 110 on most setups, 17-58 on AWS depending on instance type due to ENI limits with VPC CNI).

**Fixes:**
- Use larger instance types (more ENIs = more pods on AWS)
- Add more nodes
- Increase the max-pods kubelet setting (if the limit is artificial)

</details>

<details>
<summary>23. A container keeps getting OOMKilled — walk through how you detect this (pod describe, events, exit code 137), how you determine whether the memory limit is too low or the application has a genuine memory leak, and what the fix looks like for each case</summary>

**Detecting OOMKilled:**

```bash
kubectl describe pod api-xyz -n production
```

Look for:

```
Last State:     Terminated
  Reason:       OOMKilled
  Exit Code:    137
  Started:      ...
  Finished:     ...
```

Also check events:

```bash
kubectl get events -n production --field-selector reason=OOMKilling
```

Exit code 137 = 128 + 9 (SIGKILL). The kernel's OOM killer terminated the process because it exceeded the container's memory cgroup limit.

**Step 1: Check current memory limit vs actual usage**

```bash
# What's the memory limit?
kubectl get pod api-xyz -o jsonpath='{.spec.containers[0].resources.limits.memory}'
# 256Mi

# What was the pod actually using? (if using metrics-server)
kubectl top pod api-xyz -n production
# NAME       CPU    MEMORY
# api-xyz    45m    248Mi    # very close to 256Mi limit — likely too low
```

If `kubectl top` isn't available, check your monitoring system (Prometheus/Grafana/Datadog) for `container_memory_working_set_bytes`.

**Step 2: Determine if the limit is too low vs memory leak**

**Memory limit too low** — the application has a stable, predictable memory footprint that simply exceeds the limit. Signs:
- OOMKill happens shortly after startup (within minutes)
- Memory usage quickly plateaus near the limit
- The same version ran fine with a higher limit previously
- Memory usage is consistent across restarts

**Memory leak** — memory grows unboundedly over time. Signs:
- OOMKill happens after hours or days, not immediately
- Memory usage steadily climbs from startup until hitting the limit
- Restarting the pod temporarily fixes the problem
- The issue appeared after a specific code change

**Step 3: Investigate with monitoring**

Graph `container_memory_working_set_bytes` over time:
- **Flat line near the limit** = limit too low
- **Steadily climbing line** = memory leak

For Node.js specifically:

```bash
# Check Node.js heap usage — exec into the pod
kubectl exec -it api-xyz -n production -- node -e "console.log(process.memoryUsage())"
# {
#   rss: 180000000,        # total resident memory
#   heapTotal: 120000000,  # V8 heap allocated
#   heapUsed: 95000000,    # V8 heap actually used
#   external: 15000000,    # C++ objects bound to JS
#   arrayBuffers: 5000000
# }
```

**Fix for limit too low:**

```yaml
resources:
  requests:
    memory: "256Mi"     # set to normal usage
  limits:
    memory: "512Mi"     # set to 1.5-2x request for headroom
```

For Node.js, also ensure the V8 heap limit aligns with the container limit. By default, Node.js sets the heap limit based on available system memory, which inside a container might read the node's total memory (not the cgroup limit). Set it explicitly:

```yaml
env:
  - name: NODE_OPTIONS
    value: "--max-old-space-size=384"  # ~75% of 512Mi limit, leaving room for non-heap memory
```

**Fix for memory leak:**

1. **Profile the application** — use `--inspect` flag with Chrome DevTools or `clinic.js` to take heap snapshots. Compare snapshots over time to find growing objects.
2. **Common Node.js leak sources:**
   - Event listeners not removed (`.on()` without `.off()`)
   - Growing caches without eviction (in-memory Maps/Objects that never clear)
   - Closures holding references to large objects
   - Streams not properly destroyed
3. **Short-term mitigation** while investigating: increase the memory limit and set up an alert when memory exceeds 80% of the limit. This buys time without hiding the problem.

</details>

<details>
<summary>24. Service A cannot reach Service B even though both pods are running — walk through the systematic debugging process: checking DNS resolution from within a pod, verifying service selectors match pod labels, inspecting endpoints, testing connectivity with curl/wget, and checking whether Network Policies are blocking traffic. Show the exact commands at each step</summary>

Work through the networking stack systematically: DNS first, then Service/Endpoints, then direct connectivity, then Network Policies.

**Step 1: Verify both pods are Running and Ready**

```bash
kubectl get pods -n production -l app=service-a
kubectl get pods -n production -l app=service-b
# Both should be Running with READY 1/1
# If READY is 0/1, the readiness probe is failing — fix that first
```

**Step 2: Test DNS resolution from inside Service A's pod**

```bash
# Exec into Service A's pod
kubectl exec -it service-a-xyz -n production -- sh

# Resolve Service B's DNS name
nslookup service-b
# or
nslookup service-b.production.svc.cluster.local

# Expected: returns the ClusterIP of service-b Service
# If DNS fails: CoreDNS issue (check CoreDNS pods in kube-system, see question 4)
```

If the pod's image has no DNS tools:

```bash
kubectl debug -it service-a-xyz -n production --image=busybox:1.36 -- nslookup service-b
```

**Step 3: Verify Service selector matches pod labels**

```bash
# Check Service B's selector
kubectl get svc service-b -n production -o yaml | grep -A 5 selector
#   selector:
#     app: service-b

# Check Service B's pods' labels
kubectl get pods -n production -l app=service-b --show-labels
# Verify the labels actually match the selector exactly
# Common mistake: selector says "app: service-b" but pods have "app: serviceB"
```

**Step 4: Inspect Endpoints**

```bash
kubectl get endpoints service-b -n production
# NAME        ENDPOINTS                           AGE
# service-b   10.244.1.15:3000,10.244.2.23:3000   5d

# If ENDPOINTS is empty: no pods match the Service selector, or all pods are not Ready
# If ENDPOINTS shows IPs: the Service is correctly wired to pods
```

Empty endpoints is one of the most common causes. It means either:
- The selector labels don't match pod labels (check for typos)
- All pods are failing their readiness probes
- Pods exist in a different namespace than the Service

**Step 5: Test direct connectivity (bypassing DNS)**

```bash
# From inside Service A's pod:

# Test via Service ClusterIP
curl -v http://10.96.45.123:3000/health

# Test via pod IP directly (from endpoints output)
curl -v http://10.244.1.15:3000/health

# If pod IP works but ClusterIP doesn't: kube-proxy issue (iptables rules not updated)
# If neither works: network-level problem or the app isn't listening on that port
```

**Step 6: Verify the target pod is actually listening**

```bash
# Exec into Service B's pod and check
kubectl exec -it service-b-xyz -n production -- netstat -tlnp
# or
kubectl exec -it service-b-xyz -n production -- ss -tlnp
# Look for the expected port (e.g., 0.0.0.0:3000 LISTEN)

# If nothing is listening on the expected port: the app isn't binding to the right port,
# or it's binding to 127.0.0.1 instead of 0.0.0.0 (a common container mistake)
```

**Step 7: Check Network Policies**

```bash
# List all Network Policies in the namespace
kubectl get networkpolicies -n production

# Describe each one
kubectl describe networkpolicy -n production

# Key question: is there a default-deny policy?
# If yes, there must be an explicit allow rule for service-a → service-b traffic
```

If there's a default-deny policy (covered in question 19), verify:
- Service B has an ingress rule allowing traffic from pods with Service A's labels
- Service A has an egress rule allowing traffic to pods with Service B's labels
- Both ingress and egress rules specify the correct port

**Step 8: Check across namespaces**

If Service A and Service B are in different namespaces:

```bash
# Service A must use the full DNS name
curl http://service-b.other-namespace.svc.cluster.local:3000/health

# Network Policies must reference the other namespace via namespaceSelector
# A podSelector alone only matches pods in the same namespace
```

**Quick diagnostic cheat sheet:**

| Symptom | Likely cause |
|---|---|
| DNS resolution fails | CoreDNS down or misconfigured |
| DNS resolves but connection refused | Empty endpoints (selector mismatch or readiness failure) |
| ClusterIP doesn't work, pod IP does | kube-proxy issue |
| Pod IP doesn't work | App not listening, wrong port, or Network Policy blocking |
| Works intermittently | Some pods not Ready, or Network Policy allows some pods but not others |

</details>

---

## Experience-Based Questions

These questions test real-world experience. Prepare by mapping them to your own projects and situations.

<details>
<summary>25. Tell me about a time you migrated a workload to Kubernetes or set up Kubernetes infrastructure from scratch — what drove the decision, what challenges did you face during the migration, and what would you do differently?</summary>

**What the interviewer is looking for:**
- Ability to evaluate and justify infrastructure decisions (not just "everyone uses K8s")
- Awareness of migration complexity and risk management
- Practical experience with K8s operational challenges
- Humility — what you'd improve shows maturity

**Suggested structure (STAR-ish):**

1. **Context** — What was the existing setup? What pain points motivated the change? (scaling limits, deployment friction, environment inconsistency, team growth)
2. **Decision process** — Why K8s specifically vs alternatives? What tradeoffs did you weigh? Who was involved in the decision?
3. **Migration approach** — Did you do a big-bang migration or gradual? How did you handle the transition period (running both old and new)?
4. **Challenges** — Be specific. Generic "it was hard" doesn't impress. Good examples: networking differences causing service discovery issues, storage migration, managing secrets, learning curve for the team, CI/CD pipeline rework, debugging skills gap.
5. **Outcome** — What improved? Deployment frequency, reliability, developer experience?
6. **What you'd do differently** — This is the senior-level signal. Examples: "I'd invest more in local development tooling first," "I'd start with a smaller pilot service," "I'd set up proper resource requests/limits from day one instead of retrofitting."

**Example outline to personalize:**
- Migrated a monorepo of 6 services from EC2 + CodeDeploy to EKS
- Motivated by slow deployments (20 min), environment drift, and inability to scale individual services
- Chose EKS over self-managed K8s to reduce ops burden; chose K8s over ECS because we needed Network Policies and multi-cluster portability
- Migrated service-by-service, running both old and new in parallel with traffic splitting
- Biggest challenge: health probe misconfiguration caused cascading restarts during the first production deployment of the second service — took the team 2 hours to diagnose
- Would do differently: set up a staging K8s cluster and run load tests before the first production migration; invest in better observability before migrating, not after

</details>

<details>
<summary>26. Describe a time you debugged a critical Kubernetes issue in production — what were the symptoms, how did you diagnose it, and what was the root cause?</summary>

**What the interviewer is looking for:**
- Systematic debugging approach (not guessing)
- K8s-specific diagnostic skills (kubectl commands, understanding of control plane behavior)
- Calm under pressure — how you handle production incidents
- Root cause analysis, not just "I restarted it"

**Suggested structure:**

1. **Symptoms** — What did users/alerts see? Be specific: error rates, latency spikes, pods restarting, services unreachable.
2. **Initial triage** — What did you check first? How did you narrow down the scope? (Which service, which namespace, which nodes?)
3. **Diagnostic steps** — Walk through the actual commands and reasoning. Show your thought process: "I checked X, which ruled out Y, so I looked at Z."
4. **Root cause** — The actual underlying issue. Bonus points if it's non-obvious (not just "the pod was OOMKilled").
5. **Resolution** — Immediate fix + long-term fix. Shows you think beyond the firefight.
6. **Prevention** — What did you put in place to prevent recurrence? (Alerts, config changes, runbooks, post-mortem)

**Example outline to personalize:**
- Symptoms: intermittent 502 errors on the API, ~5% of requests failing, no pattern in timing
- Initial triage: pods all Running and Ready, no restarts, no OOM events
- Diagnostic: noticed 502s correlated with deployment rollouts; dug into Ingress controller logs and found requests hitting pods that were shutting down; the preStop hook was missing and `terminationGracePeriodSeconds` was default 30s
- Root cause: during rolling updates, endpoint removal was slower than pod shutdown — in-flight requests hit terminating pods
- Fix: added `preStop: sleep 10` hook and readiness probe (as covered in question 17)
- Prevention: added the preStop hook to the Helm chart template so all services get it by default; added an SLO alert on 5xx error rate

**Key signals of seniority:** You explain WHY each diagnostic step was chosen, not just what you typed. You distinguish between the immediate fix and the systemic improvement.

</details>

<details>
<summary>27. Tell me about a time you had to optimize Kubernetes resource usage or cost — what was the situation, what changes did you make (right-sizing, autoscaling, spot instances), and what was the measurable impact?</summary>

**What the interviewer is looking for:**
- Data-driven approach to optimization (measured, not guessed)
- Understanding of K8s resource model (requests vs limits, QoS classes, scheduling implications)
- Ability to balance cost reduction with reliability
- Quantifiable results

**Suggested structure:**

1. **Situation** — What triggered the optimization? (Cloud bill too high, cluster running hot, scaling issues, management asking "why is K8s so expensive?")
2. **Analysis** — How did you measure the problem? What tools did you use? (Prometheus, Kubecost, VPA recommendations, `kubectl top`)
3. **Changes made** — Be specific about what you changed and why. Multiple levers are better than one.
4. **Risk management** — How did you ensure changes didn't hurt reliability? (Staged rollout, monitoring, PDBs)
5. **Impact** — Numbers. Cost reduction percentage, resource efficiency improvement, pods per node improvement.

**Key optimization levers to draw from:**

- **Right-sizing requests** — Most teams over-request by 2-5x. Use VPA in recommendation mode or Prometheus metrics (`container_memory_working_set_bytes`, `container_cpu_usage_seconds_total`) to set requests to actual p95 usage. This is usually the single biggest win.
- **HPA** — Scale down during low-traffic periods instead of running peak capacity 24/7.
- **Spot/Preemptible instances** — For fault-tolerant workloads (stateless APIs with multiple replicas, batch jobs). 60-90% cheaper but can be terminated with 2 minutes notice.
- **Cluster Autoscaler** — Scale nodes down when pods don't need them. Ensure PDBs are set so the autoscaler can safely drain nodes.
- **Namespace ResourceQuotas** — Prevent teams from over-provisioning by setting per-namespace budgets.
- **Remove idle resources** — Old test deployments, unused PVCs still provisioned, LoadBalancer Services for internal-only traffic.

**Example outline to personalize:**
- Situation: K8s cluster costs grew 3x in 6 months as more services migrated; leadership asked for a 30% cost reduction
- Analysis: Kubecost showed 65% of requested CPU was never used; most services had copy-pasted resource specs with 1 CPU request when they used 50m
- Changes: right-sized requests using VPA recommendations (biggest win — freed 60% of cluster capacity), added HPA to 8 stateless services, moved batch jobs to spot node pools, set up namespace quotas
- Impact: 40% cost reduction ($12k/month saved), cluster node count dropped from 15 to 9, no reliability incidents during the change

</details>

<details>
<summary>28. Describe a time you designed the deployment strategy or CI/CD pipeline for a Kubernetes-based application — what decisions did you make about rollout strategy, environment promotion, and developer experience?</summary>

**What the interviewer is looking for:**
- End-to-end thinking: from developer push to production traffic
- Deliberate decisions about safety vs speed tradeoffs
- Awareness of developer experience as a design constraint
- Understanding of GitOps, promotion strategies, and rollback mechanisms

**Suggested structure:**

1. **Context** — What was the team size, number of services, deployment frequency goal, and existing pain points?
2. **Pipeline design decisions** — Walk through the key choices you made and why:
   - **Build stage:** How images are built and tagged (Git SHA vs semantic version)
   - **Rollout strategy:** Rolling update, canary, or blue-green — and why (covered in question 10)
   - **Environment promotion:** How does a change go from dev to staging to production? Automatic or manual gates?
   - **GitOps vs push-based:** Did you use ArgoCD/Flux (pull-based) or CI pushing to the cluster (push-based)?
3. **Developer experience considerations** — What did you do to make it easy for developers to deploy, debug, and roll back?
4. **Safety mechanisms** — What prevents a bad deploy from causing downtime? (Health checks, automated rollback, canary analysis, PDBs)
5. **Outcome** — Deployment frequency, lead time, failure rate, developer satisfaction

**Key decisions to highlight:**

- **Image tagging strategy:** Git SHA tags (immutable, traceable) vs `latest` (never do this in production — no way to know what's running or roll back to a specific version)
- **GitOps vs push:** GitOps (ArgoCD/Flux watches a Git repo and reconciles cluster state) gives you auditability, drift detection, and easy rollback (revert a Git commit). Push-based (CI runs `helm upgrade`) is simpler but has no drift detection and credentials live in CI.
- **Environment promotion:** Promote the same image through environments (same SHA in dev, staging, prod) — only config changes per environment via Helm values files (as shown in question 20). Never rebuild for each environment.
- **Rollback strategy:** Helm `rollback` or ArgoCD sync to previous Git commit. Automated rollback based on error rate thresholds (Argo Rollouts).
- **Local development:** How do developers run services locally? (Docker Compose, Tilt, Skaffold, or just `npm start` with mocked dependencies)

**Example outline to personalize:**
- Context: team of 8 engineers, 12 services in a monorepo, deploying 2-3 times per week (wanted daily)
- Designed a GitHub Actions pipeline: lint/test on PR, build + push image on merge to main (tagged with Git SHA), update Helm values in a GitOps repo
- ArgoCD watches the GitOps repo, auto-syncs to staging, manual approval for production
- Rolling updates with `maxUnavailable: 0` for all services, canary via Argo Rollouts for the 2 highest-traffic services
- Developer experience: `make deploy-dev` command that builds and pushes to the dev cluster, Slack notifications on deploy success/failure, one-click rollback button in ArgoCD UI
- Result: deployment frequency went from 2-3/week to 2-3/day, rollback time dropped from 30 minutes (manual) to 2 minutes (ArgoCD revert)

</details>
