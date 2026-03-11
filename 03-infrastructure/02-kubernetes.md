# Kubernetes

> **41 questions** — 14 theory, 27 practical

- Core primitives: Pods, Deployments, Services, Namespaces — and why K8s separates them
- Control plane architecture: API server, etcd, scheduler, controller manager — failure modes
- Kubernetes networking model: flat network, CNI plugins, pod-to-pod communication
- When to adopt K8s vs simpler alternatives — complexity and cost tradeoffs
- DNS and service discovery: CoreDNS internals, DNS-based routing, headless services for StatefulSet discovery
- Service types: ClusterIP, NodePort, LoadBalancer, ExternalName — when to use each
- Ingress: TLS termination with cert-manager, ACME flow, rate limiting and authentication at the edge
- Network Policies and microsegmentation (allow/deny traffic between services)
- Scheduling: taints, tolerations, node affinity, pod priorities, preemption
- RBAC: Roles, ClusterRoles, Bindings, ServiceAccounts — common over-permissioning mistakes
- API access control flow: Authentication → Authorization (RBAC) → Admission Control (webhooks, OPA/Gatekeeper)
- Storage: PV, PVC, StorageClass — differences from Docker volumes
- Workload types: Deployments, StatefulSets, DaemonSets, Jobs, CronJobs
- CRDs and operators: what they are, common examples (cert-manager, database operators), when they matter
- Service mesh concepts (Istio/Linkerd) and when it's overkill
- ConfigMaps and Secrets: env vars vs volume mounts, multi-environment setup
- Health probes: liveness, readiness, startup — timing parameters and cascading restart pitfalls
- Autoscaling: HPA (CPU + custom metrics), VPA, Cluster Autoscaler interaction
- Resource requests, limits, LimitRanges, ResourceQuotas, and namespace isolation — QoS classes (Guaranteed, Burstable, BestEffort) and eviction priority
- Deployment strategies: rolling updates (maxSurge, maxUnavailable), blue-green, canary — when each is appropriate
- PodDisruptionBudgets and zero-downtime deployments (preStop hooks, connection draining)
- Init containers and sidecar pattern: dependency setup, migration scripts, logging/proxy sidecars
- Helm: chart architecture, values overrides, hooks for migrations, Helm vs Kustomize
- Production debugging: CrashLoopBackOff, Pending pods, OOMKilled, service-to-service networking failures, ephemeral containers (kubectl debug)
- Day-to-day kubectl workflows, multi-cluster context management, and k9s

---

## Foundational

<details>
<summary>1. What are the core Kubernetes primitives (Pods, Deployments, Services, Namespaces) and why does K8s separate them into distinct abstractions instead of just running containers directly — what design problem does each primitive solve, and how do they compose together to form a working application?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>2. How does the Kubernetes control plane work — why is it split into API server, etcd, scheduler, and controller manager instead of being a single process, how do these components communicate, and what happens to your running workloads when each one fails?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>3. Why does Kubernetes require a flat networking model where every pod gets its own IP and can reach every other pod directly — what problem does this solve compared to Docker's default networking, and what role do CNI plugins play in implementing this model?</summary>

<!-- Answer will be added later -->

</details>

## Conceptual Depth

<details>
<summary>4. When should you adopt Kubernetes vs simpler alternatives like ECS, Cloud Run, or plain VMs — what are the real complexity and cost tradeoffs, and what signals tell you a team is adopting K8s too early?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>5. What are the Kubernetes Service types (ClusterIP, NodePort, LoadBalancer, ExternalName), why does each exist, and how do you decide which one to use — what are the tradeoffs of each and when would you avoid them?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>6. How does DNS-based service discovery work in Kubernetes — what does CoreDNS do, how do services get DNS names, what is a headless service and when would you use one over a regular ClusterIP, and how does ExternalName fit in?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>7. Why does Kubernetes have multiple workload types (Deployments, StatefulSets, DaemonSets, Jobs, CronJobs) instead of just one — what specific problem does each solve, and what goes wrong when you use the wrong one for a given workload?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>8. Why does Kubernetes need its own storage abstraction (PV, PVC, StorageClass) instead of just using Docker volumes — how does the PV/PVC binding model work, what does dynamic provisioning via StorageClass solve, and when should you avoid persistent storage in K8s entirely?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>9. What are Custom Resource Definitions (CRDs) and operators, why did Kubernetes introduce them, and how do common operators like cert-manager and database operators extend the cluster — when do CRDs add real value vs unnecessary complexity?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>10. What problems does a service mesh (Istio, Linkerd) solve that Kubernetes doesn't handle out of the box — what capabilities does it add, what operational overhead does it introduce, and when is adopting a service mesh overkill for your team?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>11. How does the Kubernetes API access control flow work — why is it split into Authentication, Authorization (RBAC), and Admission Control as three sequential stages, what does each stage decide, and how do admission controllers (validating and mutating webhooks, OPA/Gatekeeper) enforce policies that RBAC alone cannot — such as requiring resource limits, blocking privileged containers, or enforcing image registries?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>12. Why does Kubernetes have three separate health probes (liveness, readiness, startup) instead of just one — what specific failure mode does each one detect, how do the timing parameters (initialDelaySeconds, periodSeconds, failureThreshold) interact, and how do misconfigured liveness probes cause cascading restarts that can take down an entire service?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>13. Why does Kubernetes have init containers and the sidecar pattern as separate concepts from the main application container — what problems do init containers solve (dependency checks, migration scripts, config generation), how do sidecar containers work for cross-cutting concerns (logging, proxying, service mesh data planes), and what ordering and lifecycle guarantees does Kubernetes provide for each?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>14. Compare rolling updates, blue-green deployments, and canary releases in Kubernetes — what problem does each strategy solve, how do you implement blue-green and canary (since K8s only natively supports rolling updates), and what factors determine which strategy is appropriate for a given service?</summary>

<!-- Answer will be added later -->

</details>

## Practical — Single Resource Configuration

<details>
<summary>15. Two microservices need to communicate within the same cluster — show how to set up the Deployments and ClusterIP Services so Service A can call Service B reliably, explain the DNS naming convention (service.namespace.svc.cluster.local), when to use the short name vs the FQDN, and what best practices ensure this communication is resilient (retries, timeouts, readiness gates)</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>16. Configure a Deployment that reads configuration from both a ConfigMap (as environment variables) and a Secret (as a mounted volume) — show the YAML for all three resources, explain when you'd choose env vars vs volume mounts, how you handle multiple environments, and what happens if the ConfigMap changes while pods are running</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>17. Configure resource requests and limits for a Deployment, then set up a LimitRange and ResourceQuota for a namespace — show the YAML for all three, explain the difference between requests and limits, what happens when a container exceeds its memory limit vs CPU limit, how requests and limits determine a pod's QoS class (Guaranteed, Burstable, BestEffort) and its eviction priority, and why getting requests wrong causes scheduling failures and noisy-neighbor issues</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>18. Configure RBAC so a CI/CD pipeline's service account can deploy to a specific namespace but nothing else — show the ServiceAccount, Role, and RoleBinding YAML, explain the difference between Role and ClusterRole, and identify the common over-permissioning mistakes teams make (wildcard verbs, cluster-admin for CI, overly broad ServiceAccounts)</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>19. Configure pod scheduling using taints, tolerations, and node affinity — show the YAML for a scenario where certain pods must run on GPU nodes while others must avoid them, explain the difference between required and preferred affinity rules, and how pod priorities and preemption work when the cluster is resource-constrained</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>20. Set up persistent storage for a StatefulSet using a PersistentVolumeClaim and a StorageClass with dynamic provisioning — show the YAML and explain what happens when a pod with a PVC is rescheduled to a different node, and when you'd use ReadWriteOnce vs ReadWriteMany access modes</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>21. Configure a pod that uses an init container to run database migrations before the main application starts, and a sidecar container for log forwarding — show the pod spec YAML, explain the container ordering guarantees, what happens if the init container fails, and how the sidecar lifecycle interacts with the main container's lifecycle</summary>

<!-- Answer will be added later -->

</details>

## Practical — Multi-Resource Composition

<details>
<summary>22. Set up an Ingress with automatic TLS using cert-manager and Let's Encrypt — show the YAML for the Ingress, ClusterIssuer, and any Certificate resources, explain the ACME challenge flow, and what happens when certificate renewal fails</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>23. Configure rate limiting and authentication at the Ingress layer — show the annotations or configuration for an NGINX Ingress Controller, explain why handling these at the ingress level is better than in application code for certain use cases, and what the tradeoffs are</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>24. Why does Kubernetes default to allow-all traffic between pods and what does that mean for security? Write a default-deny NetworkPolicy and then add rules that allow specific traffic paths (e.g., frontend → API on port 3000, API → database on port 5432) — show the YAML and explain the common mistakes teams make when first adopting Network Policies</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>25. Set up a HorizontalPodAutoscaler that scales on both CPU utilization and a custom metric (e.g., queue depth) — show the YAML, explain how HPA calculates the desired replica count using the scaling formula, and what happens when multiple metrics disagree on the target replica count</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>26. Compare HPA and VPA — what problem does each solve, when would you use them together, and what conflicts arise when both try to adjust the same workload?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>27. Set up namespace-level isolation for a multi-tenant cluster — show the YAML for ResourceQuotas (CPU, memory, pod count limits), LimitRanges (default requests/limits for pods), and a default-deny NetworkPolicy. Explain how these three resources work together to prevent one team from affecting another</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>28. Configure a zero-downtime deployment — show the Deployment spec with rolling update strategy (maxUnavailable/maxSurge), a PodDisruptionBudget, and a pod spec with a preStop hook and proper connection draining. Explain why you need each piece, how to set terminationGracePeriodSeconds correctly, and what happens to in-flight requests without these configurations</summary>

<!-- Answer will be added later -->

</details>

## Practical — Helm & Tooling

<details>
<summary>29. Walk through the structure of a Helm chart — show the directory layout, explain what Chart.yaml, values.yaml, and templates/ contain, demonstrate how values overrides work (--set vs -f values files), and how you manage different configurations across environments (dev, staging, production)</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>30. How do Helm hooks work for running tasks like database migrations during a deployment — show the hook annotations, explain the hook lifecycle (pre-install, pre-upgrade, post-upgrade), what happens when a hook fails, and compare Helm vs Kustomize for managing Kubernetes manifests — when would you choose one over the other?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>31. Walk through the kubectl commands you use daily for deploying, inspecting, and troubleshooting — show how you roll out a deployment, check pod logs and events, exec into a running container, tail logs across multiple pods, and quickly find why a pod isn't starting. What productivity shortcuts (aliases, autocompletion, -o formatting) make kubectl efficient?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>32. How do you manage multiple Kubernetes clusters and contexts — show how kubeconfig is structured, walk through the exact commands for switching contexts safely (avoiding accidental production commands), demonstrate how you'd set up aliases or kubectx to prevent mistakes, and when does k9s give you an advantage over raw kubectl for navigating and inspecting resources in real time?</summary>

<!-- Answer will be added later -->

</details>

## Practical — Debugging & Troubleshooting

<details>
<summary>33. A pod is stuck in CrashLoopBackOff — walk through the exact kubectl commands to diagnose the root cause step by step: checking pod status, reading events, pulling logs from the previous crashed container, interpreting exit codes, and using ephemeral containers (kubectl debug) when the container has no shell or debugging tools. What are the most common causes and how do you fix each?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>34. A pod has been Pending for 10 minutes and won't schedule — walk through the exact steps to diagnose why: checking scheduler events, identifying resource constraints vs node selector mismatches vs taint issues vs PVC binding failures. What are the common fixes for each cause?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>35. A container keeps getting OOMKilled — walk through how you detect this (pod describe, events, exit code 137), how you determine whether the memory limit is too low or the application has a genuine memory leak, and what the fix looks like for each case</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>36. Service A cannot reach Service B even though both pods are running — walk through the systematic debugging process: checking DNS resolution from within a pod, verifying service selectors match pod labels, inspecting endpoints, testing connectivity with curl/wget, and checking whether Network Policies are blocking traffic. Show the exact commands at each step</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>37. HPA has scaled your deployment to max replicas but pods are stuck Pending because the cluster has no node capacity — show the exact kubectl commands to diagnose why the Cluster Autoscaler isn't adding nodes (checking CA logs, node pool status, pending pod events), explain what common misconfigurations prevent scale-up (resource fragmentation, max node limits, pod anti-affinity), and what you'd configure to prevent HPA and Cluster Autoscaler conflicts</summary>

<!-- Answer will be added later -->

</details>

---

## Experience-Based Questions

These questions test real-world experience. Prepare by mapping them to your own projects and situations.

<details>
<summary>38. Tell me about a time you migrated a workload to Kubernetes or set up Kubernetes infrastructure from scratch — what drove the decision, what challenges did you face during the migration, and what would you do differently?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>39. Describe a time you debugged a critical Kubernetes issue in production — what were the symptoms, how did you diagnose it, and what was the root cause?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>40. Tell me about a time you had to optimize Kubernetes resource usage or cost — what was the situation, what changes did you make (right-sizing, autoscaling, spot instances), and what was the measurable impact?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>41. Describe a time you designed the deployment strategy or CI/CD pipeline for a Kubernetes-based application — what decisions did you make about rollout strategy, environment promotion, and developer experience?</summary>

<!-- Answer framework will be added later -->

</details>
