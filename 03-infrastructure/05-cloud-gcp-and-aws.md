# Cloud — GCP & AWS

> **20 questions** — 9 theory, 8 practical, 3 experience

- Cloud organization: regions, zones, projects/accounts — GCP vs AWS structural differences
- Shared responsibility model across IaaS, PaaS, and serverless
- IAM: GCP resource hierarchy + roles vs AWS policy documents + principals — least-privilege patterns, Workload Identity/IRSA for K8s-to-cloud auth
- VPC architecture: subnets (public/private), NAT gateways, route tables, firewall rules vs security groups vs NACLs, VPC flow logs for troubleshooting
- Cloud load balancing: L4 vs L7 tradeoffs, application vs network load balancers, Cloud Armor/WAF, SSL termination, health checks
- Compute tradeoffs: VMs vs containers (GKE/EKS) vs serverless (Cloud Run/Fargate, Functions/Lambda)
- Managed databases: Cloud SQL/RDS, Spanner/Aurora, Firestore/DynamoDB, Memorystore/ElastiCache
- Managed messaging and eventing: Pub/Sub, SNS/SQS, EventBridge — when to use managed vs self-hosted queues
- Container registries: Artifact Registry/ECR, image lifecycle policies, vulnerability scanning, authentication from CI/CD and K8s
- GKE vs EKS: VPC-native vs overlay networking, node autoscaling (NAP vs Karpenter), control plane upgrades, managed add-ons — what each abstracts vs leaves to you
- Cross-region and multi-zone high availability: regional resources, global load balancing, replication

---

## Foundational

<details>
<summary>1. How are GCP and AWS organized structurally — what are the differences between GCP's organization/folder/project hierarchy and AWS's account/OU model, why do regions and zones exist as separate concepts, and how do these structural choices affect how you isolate environments, manage billing, and set security boundaries?</summary>

**GCP hierarchy**: Organization → Folders → Projects. The organization is the root (tied to a Google Workspace or Cloud Identity domain). Folders are logical groupings (by team, environment, product). Projects are the fundamental unit — every resource lives in a project, and billing, IAM, and API enablement are scoped to the project level. IAM policies set at higher levels inherit downward, so a role granted at the folder level applies to all projects within it.

**AWS hierarchy**: Organization → Organizational Units (OUs) → Accounts. AWS accounts are the blast-radius boundary — each account has its own IAM, billing, and resource namespace. OUs group accounts for applying Service Control Policies (SCPs) that restrict what accounts can do. Unlike GCP, IAM policies in AWS don't inherit across accounts by default — cross-account access requires explicit role assumption (sts:AssumeRole).

**Regions and zones**: A region is a geographic area (e.g., us-east1, eu-west-1) with independent infrastructure. Zones are isolated data centers within a region, each with independent power, cooling, and networking. Regions provide data residency and latency control. Zones provide fault isolation — if one zone goes down, others in the same region stay up. You deploy across multiple zones for HA within a region, and across multiple regions for disaster recovery.

**How this affects real decisions:**

- **Environment isolation**: GCP teams typically use separate projects per environment (dev, staging, prod) grouped under folders. AWS teams use separate accounts per environment — stronger isolation since accounts are hard boundaries. AWS's approach gives you tighter blast radius but more operational overhead managing cross-account access.
- **Billing**: GCP billing is per-project and can be aggregated via billing accounts linked to folders/org. AWS billing rolls up through the organization, with consolidated billing across accounts. Both support cost allocation tags, but GCP's project-level billing is simpler for small teams.
- **Security boundaries**: GCP's inheritance model is convenient but dangerous — an overly broad role at the org level grants access everywhere. AWS's account-based isolation is stricter by default but requires more plumbing (cross-account roles, resource policies) for legitimate access patterns. SCPs in AWS act as guardrails that even account admins cannot override.

</details>

<details>
<summary>2. What is the shared responsibility model and how does it shift as you move from IaaS (VMs) to PaaS (managed databases, App Engine/Elastic Beanstalk) to serverless (Cloud Functions/Lambda) — what specific responsibilities move from you to the provider at each level, and what mistakes do teams make by assuming the provider handles something they actually don't?</summary>

The shared responsibility model divides security and operational duties between you and the cloud provider. The provider always owns the physical infrastructure (data centers, hardware, hypervisor). What shifts is everything above that.

**IaaS (VMs — Compute Engine/EC2):**
- **Provider handles**: Physical hardware, hypervisor, network fabric, storage infrastructure
- **You handle**: OS patching, firewall configuration, application code, data encryption, identity management, monitoring, backups

**PaaS (Cloud SQL/RDS, App Engine/Elastic Beanstalk):**
- **Provider handles**: Everything above plus OS patching, runtime updates, HA configuration, automated backups (if enabled)
- **You handle**: Application code, data, IAM/access control, network configuration (VPC placement, firewall rules), encryption key management, query performance

**Serverless (Cloud Functions/Lambda, Cloud Run/Fargate):**
- **Provider handles**: Everything above plus scaling, server provisioning, runtime patching, infrastructure monitoring
- **You handle**: Application code, IAM permissions, data, secrets management, input validation, function configuration (memory, timeout, concurrency)

**Common mistakes teams make:**

- **"Managed means secure"**: RDS/Cloud SQL gives you automated patching, but you still own network access rules. Teams leave databases with public IPs or overly permissive security groups, assuming the managed service is locked down by default.
- **"Serverless means no IAM"**: Lambda functions still need least-privilege execution roles. Teams often give functions `AdministratorAccess` or `*` permissions because "it's just a function."
- **"Backups are automatic"**: RDS has automated backups, but you need to configure retention and test restores. Cloud SQL requires you to enable backups explicitly. Neither provider tests your restore process for you. Similarly, encryption at rest may be on by default, but in-transit encryption and key management remain your responsibility.

</details>

<details>
<summary>3. Why does the cloud compute spectrum exist — from VMs (Compute Engine/EC2) to managed containers (GKE/EKS, Cloud Run/Fargate) to functions (Cloud Functions/Lambda) — what are the real tradeoffs in control, cost model, cold start behavior, and operational burden, and how do you decide which layer fits a given workload instead of defaulting to one?</summary>

The spectrum exists because workloads have fundamentally different requirements. A long-running stateful service has different needs than an event-driven image resizer. Each layer trades control for operational simplicity.

| | VMs | Managed K8s (GKE/EKS) | Serverless Containers (Cloud Run/Fargate) | Functions (Lambda/Cloud Functions) |
|---|---|---|---|---|
| **Control** | Full — OS, runtime, networking | High — container runtime, networking, scheduling | Medium — container image, concurrency settings | Low — just the handler code |
| **Cost model** | Per-hour, always running | Per-node-hour + management fee | Per-request + CPU/memory-seconds | Per-invocation + duration |
| **Cold start** | None (always on) | None for running pods; new pod spin-up ~seconds | 0-few seconds (instance kept warm on traffic) | 100ms-several seconds depending on runtime/VPC |
| **Operational burden** | Highest — patching, scaling, monitoring all manual | High — cluster upgrades, node management, RBAC | Low — no cluster to manage | Lowest — deploy and forget |
| **Scaling** | Manual or ASG rules | HPA/VPA + node autoscaler | Automatic, per-request | Automatic, per-event |

**Decision framework:**

- **VMs**: Legacy apps that can't be containerized, workloads requiring specific OS configurations, GPU workloads, or when you need full network stack control. Also when sustained-use discounts make them cheaper than serverless at steady high load.
- **Managed K8s (GKE/EKS)**: Multiple long-running services that need service mesh, complex networking, or tight inter-service communication. When you already have K8s expertise and need fine-grained control over scheduling, resource limits, and deployment strategies.
- **Serverless containers (Cloud Run/Fargate)**: Stateless HTTP services or event processors where you want container flexibility without cluster management. Good for variable traffic — scales to zero, no cost when idle. Cloud Run is simpler (just deploy a container); Fargate still requires ECS/EKS task definitions.
- **Functions (Lambda/Cloud Functions)**: Event-driven glue — webhook handlers, file processing triggers, scheduled jobs, lightweight API endpoints. Best when execution time is short (<15 min) and you want zero infrastructure management.

**The real-world trap**: Teams default to one layer for everything. Running a low-traffic webhook handler on a full K8s cluster wastes money and ops time. Running a high-throughput, latency-sensitive API on Lambda fights cold starts and concurrency limits. Match the workload to the abstraction, not the other way around.

</details>

## Conceptual Depth — Identity & Networking

<details>
<summary>4. How do IAM models differ between GCP and AWS — why does GCP use a resource hierarchy with inherited roles while AWS uses policy documents attached to principals, what does least-privilege look like in each model, and what are the most common over-permissioning mistakes teams make on each platform?</summary>

**GCP IAM model**: Role-based, attached to the resource hierarchy. You bind a principal (user, service account, group) to a role at a specific level (organization, folder, project, or individual resource). Roles are collections of permissions (e.g., `roles/storage.objectViewer` grants `storage.objects.get` and `storage.objects.list`). Policies inherit downward — a role granted at the folder level applies to every project in that folder. There are three role types: basic (Owner/Editor/Viewer — overly broad), predefined (curated by Google for each service), and custom (you pick exact permissions).

**AWS IAM model**: Policy-based, attached to principals. You write JSON policy documents that specify Effect (Allow/Deny), Actions, Resources, and optional Conditions, then attach them to users, groups, or roles. There's no automatic inheritance across accounts — cross-account access requires explicit role assumption. AWS also has resource-based policies (e.g., S3 bucket policies) that grant access from the resource side, and SCPs that set permission ceilings for entire accounts.

**Why the different approaches**: GCP's hierarchy model was designed for organizations that manage many projects — inherited roles reduce repetition. AWS's model came from a single-account world where explicit policy attachment gives fine-grained control. AWS later added Organizations and SCPs to bolt on hierarchy-like governance.

**Least-privilege in each:**

- **GCP**: Use predefined roles instead of basic roles. Bind at the lowest possible hierarchy level (project or resource, not folder or org). Use conditions to restrict by IP, time, or resource attribute. Audit with IAM Recommender, which analyzes actual usage and suggests tighter roles.
- **AWS**: Write policies with specific actions and resource ARNs — never use `"Action": "*"` or `"Resource": "*"`. Use IAM Access Analyzer to find unused permissions. Prefer managed policies over inline policies for reusability. Use permission boundaries to set a maximum scope even if someone attaches broader policies.

**Common over-permissioning mistakes:**

- **GCP**: Using basic roles (`roles/editor`) instead of predefined ones — Editor grants write access to almost every service. Granting roles at the org or folder level "for convenience" when only one project needs it. Creating service accounts with `roles/owner` because a deployment script needed access to two services.
- **AWS**: Attaching `AdministratorAccess` or `PowerUserAccess` to developers "temporarily." Using wildcard resources (`"Resource": "*"`) in policies because writing specific ARNs takes effort. Not using SCPs to prevent account admins from doing dangerous things (disabling CloudTrail, creating public S3 buckets). Storing long-lived access keys instead of using roles with temporary credentials.

</details>

<details>
<summary>5. Why does cloud networking require VPCs with public and private subnets instead of just putting everything on the internet — what problems do NAT gateways solve, and what does a well-designed network topology look like for a typical web application with a public frontend and private backend?</summary>

**Why not put everything on the internet**: Defense in depth. If every service has a public IP, every service is an attack surface. A compromised application server could expose your database directly. Private subnets ensure that internal services (databases, caches, background workers) are unreachable from the internet — the only entry point is through a controlled layer (load balancer, bastion host, VPN). This also simplifies firewall rules: instead of locking down every individual service, you control traffic at the subnet boundary.

**What NAT gateways solve**: Services in private subnets often need outbound internet access — pulling package updates, calling external APIs, sending webhooks. But they shouldn't have public IPs (which would make them directly reachable). A NAT gateway sits in a public subnet and translates outbound traffic from private instances to its own public IP, then routes responses back. Traffic flows out but nothing can initiate a connection in. Without a NAT gateway, private instances have zero internet connectivity — `apt update`, external API calls, and container image pulls all fail silently.

**Well-designed topology for a typical web app:**

```
Internet
    │
    ▼
┌─────────────────────────────────────────┐
│ Public Subnet (per AZ/zone)             │
│  - Load Balancer (ALB/Cloud LB)         │
│  - NAT Gateway                          │
│  - Bastion host (if needed, or use IAP) │
└─────────────────────────────────────────┘
    │ (internal traffic only)
    ▼
┌─────────────────────────────────────────┐
│ Private Subnet — Application Tier       │
│  - Application servers (containers/VMs) │
│  - No public IPs                        │
│  - Outbound via NAT gateway             │
└─────────────────────────────────────────┘
    │ (application → database only)
    ▼
┌─────────────────────────────────────────┐
│ Private Subnet — Data Tier              │
│  - Database (RDS/Cloud SQL)             │
│  - Cache (ElastiCache/Memorystore)      │
│  - No internet access at all            │
└─────────────────────────────────────────┘
```

**Key design points:**

- Each tier gets its own subnet in each availability zone for HA
- The load balancer in the public subnet is the only internet-facing resource
- Application servers in the private subnet accept traffic only from the load balancer's security group/firewall rule
- The database subnet allows connections only from the application subnet on the database port (e.g., 5432 for PostgreSQL)
- NAT gateways go in public subnets (one per AZ for HA in AWS; GCP uses Cloud NAT which is regional)
- Route tables for private subnets point `0.0.0.0/0` to the NAT gateway; route tables for public subnets point `0.0.0.0/0` to the internet gateway

</details>

<details>
<summary>6. How do cloud load balancers work at L4 vs L7 — what are the tradeoffs between network load balancers and application load balancers, when would you choose each, what role do Cloud Armor (GCP) and WAF (AWS) play in protecting your application at the load balancer layer, how does SSL termination work at the load balancer layer, and how do health check configurations affect traffic routing?</summary>

**L4 (Network Load Balancer)**: Operates at the transport layer (TCP/UDP). Routes based on IP and port — it sees packets, not HTTP requests. Passes traffic through with minimal processing, so it's extremely fast and can handle millions of requests per second. Preserves the client's source IP. Cannot inspect HTTP headers, cookies, or paths.

**L7 (Application Load Balancer)**: Operates at the HTTP layer. Can route based on URL path, host header, HTTP method, query strings, and headers. Supports features like path-based routing (`/api/*` → backend, `/static/*` → CDN origin), sticky sessions, WebSocket upgrades, and gRPC. Higher latency than L4 because it terminates the TCP connection and inspects HTTP.

**When to choose each:**

- **NLB/L4**: Raw TCP/UDP workloads (game servers, IoT, custom protocols), extreme throughput requirements, when you need to preserve source IP without proxy protocol, or when you need static IP addresses (useful for allowlisting by clients).
- **ALB/L7**: HTTP/HTTPS services (most web applications), when you need path-based or host-based routing, WebSocket support, or want to integrate WAF/Cloud Armor. This is the default for almost all web applications.

**Cloud Armor (GCP) and WAF (AWS)**: Web application firewalls that attach to L7 load balancers, inspecting HTTP requests and blocking malicious traffic before it reaches your application. Cloud Armor attaches to GCP's external HTTP(S) LB with IP filtering, geo-blocking, rate limiting, OWASP Top 10 rules, and ML-based DDoS detection. AWS WAF attaches to ALB, CloudFront, or API Gateway with similar capabilities via custom rules or AWS Managed Rules. Critical for any internet-facing service — without them, your app burns compute on malicious requests.

**SSL termination**: The load balancer decrypts TLS using a certificate you provision (ACM on AWS, Google-managed certs on GCP). Backends receive plain HTTP, centralizing certificate management and avoiding per-server TLS overhead. For compliance requiring end-to-end encryption, you can re-encrypt between LB and backends, but most teams terminate at the LB since VPC traffic is already on a private network.

**Health checks**: The load balancer periodically sends requests to backends (HTTP GET on a path like `/health`, or TCP connection checks). If a backend fails the health check threshold (e.g., 3 consecutive failures), the LB stops routing traffic to it. Key configuration decisions:

- **Interval and threshold**: Too aggressive (1s interval, 1 failure) causes flapping during minor GC pauses. Too lenient (30s interval, 5 failures) means serving errors for minutes before unhealthy instances are removed.
- **Health check path**: Should verify the app is genuinely ready (can reach the database, has loaded configs), not just return 200 from a static route — but keep it lightweight to avoid adding load. During deployments, ensure old instances deregister and drain connections before shutting down.

</details>

## Conceptual Depth — Data, Storage & Messaging

<details>
<summary>7. How do you choose between managed database options on GCP and AWS — what are the tradeoffs between relational (Cloud SQL/RDS), globally distributed (Spanner/Aurora), document (Firestore/DynamoDB), and in-memory (Memorystore/ElastiCache), and when does each become the right or wrong choice for a given workload?</summary>

**Relational — Cloud SQL / RDS (PostgreSQL, MySQL)**

Best for: Structured data with relationships, ACID transactions, complex queries with JOINs. The default choice for most applications.

Tradeoffs: Single-region by default. Vertical scaling has limits. Read replicas help read-heavy workloads but writes go to a single primary. HA is within a region (multi-AZ in AWS, regional in GCP).

Wrong choice when: You need global distribution, sub-millisecond latency, or horizontal write scaling beyond what a single primary can handle.

**Globally distributed — Spanner / Aurora**

These aren't equivalent. Spanner is a globally distributed, strongly consistent relational database — it scales writes horizontally across regions with external consistency. Aurora is a MySQL/PostgreSQL-compatible engine with faster replication and storage that auto-scales, but is still fundamentally single-region for writes (Aurora Global Database adds read replicas in other regions with ~1s lag).

Best for: Spanner — global applications needing strong consistency across regions (financial systems, inventory). Aurora — applications that outgrow RDS and need better replication performance but don't need true global distribution.

Wrong choice when: Spanner — you have a simple single-region app (it's expensive and complex). Aurora — you need strongly consistent multi-region writes (Aurora Global is async).

**Document — Firestore / DynamoDB**

Best for: High-throughput key-value or document access patterns, known query patterns at design time, massive scale with predictable latency. DynamoDB is a key-value/document store with single-digit-ms latency at any scale. Firestore (in Datastore mode) is similar but with stronger consistency and automatic indexing.

Tradeoffs: You must design your data model around access patterns upfront. No JOINs — denormalize everything. DynamoDB pricing can surprise you (read/write capacity units, or on-demand which is more expensive per operation). Firestore has lower write throughput limits per document/collection group.

Wrong choice when: You have complex relational queries, ad-hoc analytics, or don't know your access patterns yet. Migrating away from a poorly designed DynamoDB schema is painful.

**In-memory — Memorystore / ElastiCache (Redis, Memcached)**

Best for: Caching (session data, API responses, database query results), real-time leaderboards, rate limiting, pub/sub within a region. Sub-millisecond latency.

Tradeoffs: Data is volatile (persistence options exist in Redis but it's not a primary database). Memory is expensive compared to disk. Dataset must fit in memory.

Wrong choice when: You need durable storage, complex queries, or your dataset exceeds available memory. Also overkill if your database is already fast enough — adding a cache layer adds complexity (invalidation, consistency).

**Decision framework:**

1. Start with your data model — relational or not? If relational, start with Cloud SQL/RDS.
2. Do you need global distribution with strong consistency? Spanner. Just better MySQL/PostgreSQL performance? Aurora.
3. Are your queries all key-value lookups with known access patterns? DynamoDB/Firestore.
4. Do you need sub-millisecond reads for hot data? Add Redis as a caching layer.

Most applications start with a relational database + Redis cache and never need anything else.

</details>

<details>
<summary>8. When should you use managed messaging services (Pub/Sub, SNS/SQS, EventBridge) vs self-hosted alternatives (Kafka, RabbitMQ) — what are the delivery guarantee differences, how do their pricing models compare at scale, and what capabilities does EventBridge add that simple pub/sub doesn't cover?</summary>

**When to use managed vs self-hosted:**

Use **managed services** when: You want zero operational burden (no broker patching, no disk management, no replication monitoring), your messaging patterns are straightforward (fan-out, work queues, event routing), and your team doesn't have dedicated infrastructure expertise.

Use **self-hosted** (Kafka, RabbitMQ) when: You need features the managed services don't provide well — Kafka's log-based retention for event replay, consumer group semantics, exactly-once processing within streams, or very high throughput at predictable cost. RabbitMQ when you need complex routing topologies (exchanges, routing keys, dead letter queues with fine control). Also when you're already running Kubernetes and have the operational maturity to manage stateful workloads.

**Note**: AWS offers managed Kafka (MSK) and both clouds have managed RabbitMQ options, which give you the feature set with reduced ops burden — but at higher cost than self-hosted.

**Delivery guarantees:**

| Service | Guarantee | Ordering |
|---|---|---|
| **GCP Pub/Sub** | At-least-once | Ordering keys for ordered delivery within a key |
| **SQS Standard** | At-least-once | Best-effort (no guarantee) |
| **SQS FIFO** | Exactly-once processing | Strict within message group |
| **SNS** | At-least-once (fan-out to subscribers) | No ordering guarantee |
| **Kafka** | At-least-once by default, exactly-once with idempotent producers + transactions | Strict within partition |
| **RabbitMQ** | At-least-once with publisher confirms + consumer acks | Strict within a queue |

**Pricing comparison at scale:**

- **Pub/Sub**: Per-message + data volume. ~$40/TiB ingested + $40/TiB delivered (approximate base-tier prices — actual pricing includes a free tier of 10 GiB/month and volume discounts at higher tiers). At very high volumes (billions of messages/day), this gets expensive but includes all infrastructure management.
- **SQS**: $0.40 per million requests (standard). Cheap for moderate volumes, but costs scale linearly with throughput.
- **Kafka (self-hosted)**: Fixed infrastructure cost (VMs/disks) regardless of message volume. At high throughput, dramatically cheaper per message — but you pay for engineers to operate it. MSK charges per broker-hour + storage, which adds up.
- **RabbitMQ**: Similar to Kafka — fixed infra cost. Lower throughput ceiling than Kafka but simpler to operate for moderate workloads.

The crossover point is typically around millions of messages per day — below that, managed is almost always cheaper when you factor in operations cost. Above that, self-hosted Kafka starts winning on per-message cost.

**What EventBridge adds:**

EventBridge is not just a message bus — it's an event router with built-in schema awareness and integrations.

- **Content-based routing**: Filter events by any field in the JSON payload using pattern matching rules, routing different events to different targets without consumer-side filtering. Pub/Sub and SNS filter on attributes/headers, not payload content.
- **Schema registry**: Automatically discovers event schemas from connected sources. You can browse, version, and generate code bindings for event types.
- **First-party integrations**: Receives events directly from 100+ AWS services and 30+ SaaS partners (Shopify, Zendesk, Auth0) without you writing integration code.
- **Archive and replay**: Store events and replay them to a target — useful for debugging, reprocessing after a bug fix, or populating new consumers.
- **Scheduled events**: Cron-like scheduling built in — no need for a separate scheduler service.

EventBridge is the right choice when you're building event-driven architectures within AWS and want routing, transformation, and integration without glue code. It's the wrong choice for high-throughput streaming (it has lower throughput limits than SQS/Kafka) or cross-cloud architectures.

</details>

## Conceptual Depth — Compute & Containers

<details>
<summary>9. What are the meaningful differences between GKE and EKS — how do they differ in networking models, node autoscaling, control plane management, and cluster upgrades, and what does each platform manage for you vs leave to you that affects day-to-day operations?</summary>

**Networking:**

- **GKE (VPC-native)**: Pods get IP addresses from the VPC's secondary CIDR ranges (alias IPs). Every pod is directly routable within the VPC — no overlay network needed. This means VPC firewall rules, routes, and network policies work natively with pod IPs. Cloud Load Balancers can target pods directly (container-native load balancing) instead of going through NodePort.
- **EKS**: Uses the VPC CNI plugin by default — pods also get VPC IPs, so it's similar to GKE's VPC-native mode. But the number of pods per node is limited by the number of ENIs (elastic network interfaces) the instance type supports. A `t3.medium` can only run ~17 pods. This is a capacity planning constraint GKE doesn't have.

**Node autoscaling:**

- **GKE**: Cluster Autoscaler is built in and enabled with a flag. GKE also offers **Node Auto-Provisioning (NAP)**, which automatically creates and deletes node pools with optimal machine types based on pending pod requirements — you don't pre-define node pools. GKE Autopilot takes this further: Google manages all nodes, you only define pods.
- **EKS**: Cluster Autoscaler exists but is a self-managed add-on. The modern replacement is **Karpenter** — an open-source, AWS-native autoscaler that provisions right-sized EC2 instances directly (no node groups needed), consolidates underutilized nodes, and responds faster than Cluster Autoscaler. Karpenter is more flexible than GKE's NAP but requires you to install and configure it.

**Control plane management:**

- **GKE**: Fully managed control plane — Google handles etcd, API server, scheduler, controller manager. No visibility into control plane VMs. GKE SLA covers control plane availability. You don't pay separately for the control plane in Autopilot; Standard mode charges a management fee.
- **EKS**: Also managed control plane, but you're responsible for more. EKS charges $0.10/hour for the control plane. You manage add-ons (CoreDNS, kube-proxy, VPC CNI) — EKS offers managed add-ons but you still choose versions and trigger updates. IAM integration (aws-auth ConfigMap or EKS access entries) is a common source of misconfigurations.

**Cluster upgrades:**

- **GKE**: Supports automatic control plane upgrades on release channels (Rapid, Regular, Stable). Node auto-upgrades can be enabled per node pool. GKE handles surge upgrades (creates new nodes, drains old ones). Maintenance windows let you control timing. Overall, upgrades are more hands-off.
- **EKS**: Control plane upgrades are initiated by you (or automated via Terraform/CDK). Node group upgrades require a separate step — you update the launch template and trigger a rolling update, or use managed node groups which handle it semi-automatically. Add-on compatibility must be checked manually against the new K8s version. More steps, more room for error.

**Day-to-day operational impact summary:**

| Area | GKE manages | EKS leaves to you |
|---|---|---|
| Logging/monitoring | Cloud Logging/Monitoring integrated by default | Install FluentBit/CloudWatch agent yourself |
| Ingress | GKE Ingress controller built in | Install and manage ALB Ingress Controller or Nginx |
| Service mesh | Managed Anthos Service Mesh available | Install and manage Istio or App Mesh yourself |
| DNS | kube-dns managed | CoreDNS managed as add-on (you manage version) |
| GPU support | Built-in GPU node pools | Install NVIDIA device plugin yourself |


</details>

## Practical — Networking & Security

<details>
<summary>10. Design a VPC architecture for a web application with a public-facing load balancer, private application servers, and a private database tier — show the subnet layout, NAT gateway placement, and firewall rules (or security groups) using either gcloud or AWS CLI commands, and explain what breaks if you skip the NAT gateway or misconfigure the firewall rules.</summary>

Here's a complete three-tier VPC architecture using AWS CLI (the concepts map directly to GCP — noted below).

**Subnet layout** (across 2 AZs for HA):

```bash
# Create VPC
aws ec2 create-vpc --cidr-block 10.0.0.0/16 --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=app-vpc}]'

# Public subnets (load balancer + NAT gateways)
aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.1.0/24 --availability-zone us-east-1a --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=public-1a}]'
aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.2.0/24 --availability-zone us-east-1b --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=public-1b}]'

# Private subnets — application tier
aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.10.0/24 --availability-zone us-east-1a --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=app-1a}]'
aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.11.0/24 --availability-zone us-east-1b --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=app-1b}]'

# Private subnets — database tier
aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.20.0/24 --availability-zone us-east-1a --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=db-1a}]'
aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.21.0/24 --availability-zone us-east-1b --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=db-1b}]'
```

**Internet gateway + NAT gateway:**

```bash
# Internet gateway for public subnets
aws ec2 create-internet-gateway --tag-specifications 'ResourceType=internet-gateway,Tags=[{Key=Name,Value=app-igw}]'
aws ec2 attach-internet-gateway --internet-gateway-id $IGW_ID --vpc-id $VPC_ID

# Elastic IP + NAT gateway in each public subnet
aws ec2 allocate-address --domain vpc  # gives $EIP_1
aws ec2 create-nat-gateway --subnet-id $PUBLIC_1A --allocation-id $EIP_1

# Route tables
# Public route table: 0.0.0.0/0 → internet gateway
aws ec2 create-route --route-table-id $PUBLIC_RT --destination-cidr-block 0.0.0.0/0 --gateway-id $IGW_ID

# Private route table (per AZ): 0.0.0.0/0 → NAT gateway in same AZ
aws ec2 create-route --route-table-id $PRIVATE_RT_1A --destination-cidr-block 0.0.0.0/0 --nat-gateway-id $NAT_1A
```

**Security groups (firewall rules):**

```bash
# ALB security group — accepts HTTP/HTTPS from internet
aws ec2 create-security-group --group-name alb-sg --description "ALB" --vpc-id $VPC_ID
aws ec2 authorize-security-group-ingress --group-id $ALB_SG --protocol tcp --port 443 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id $ALB_SG --protocol tcp --port 80 --cidr 0.0.0.0/0

# App security group — accepts traffic ONLY from ALB
aws ec2 create-security-group --group-name app-sg --description "App tier" --vpc-id $VPC_ID
aws ec2 authorize-security-group-ingress --group-id $APP_SG --protocol tcp --port 3000 --source-group $ALB_SG

# DB security group — accepts traffic ONLY from app tier
aws ec2 create-security-group --group-name db-sg --description "Database tier" --vpc-id $VPC_ID
aws ec2 authorize-security-group-ingress --group-id $DB_SG --protocol tcp --port 5432 --source-group $APP_SG
```

**GCP equivalent approach** (this implements the three-tier topology described in question 5): GCP uses VPC firewall rules (applied by network tags) instead of security groups, and Cloud NAT (regional, no separate gateway instance) instead of NAT gateways. Subnets in GCP are regional (span all zones), so you create fewer subnets but use tags and firewall rules for tier isolation.

```bash
# Allow HTTPS from internet to LB-tagged instances
gcloud compute firewall-rules create allow-lb-https \
  --network=app-vpc --direction=INGRESS \
  --action=ALLOW --rules=tcp:443 \
  --source-ranges=0.0.0.0/0 --target-tags=lb-tier

# Allow app traffic only from LB tier
gcloud compute firewall-rules create allow-app-from-lb \
  --network=app-vpc --direction=INGRESS \
  --action=ALLOW --rules=tcp:3000 \
  --source-tags=lb-tier --target-tags=app-tier

# Allow database traffic only from app tier
gcloud compute firewall-rules create allow-db-from-app \
  --network=app-vpc --direction=INGRESS \
  --action=ALLOW --rules=tcp:5432 \
  --source-tags=app-tier --target-tags=db-tier
```

**What breaks without a NAT gateway:**

- Application servers can't reach the internet — `npm install` in CI/CD fails, external API calls time out, container images can't be pulled
- Health check traffic from the LB still works (that's internal), so the app appears healthy but can't do anything useful
- DNS resolution may still work (if using VPC DNS) but connections to resolved IPs fail silently with timeouts

**What breaks with misconfigured security groups:**

- **App SG allows 0.0.0.0/0 on port 3000**: Application servers are directly accessible from the internet, bypassing the load balancer — attackers can hit your app without going through WAF/Cloud Armor
- **DB SG allows the app subnet CIDR instead of the app security group**: If an attacker compromises any instance in the app subnet (not just your app), they can reach the database. Security group references are tighter than CIDR blocks.
- **Missing egress rules**: AWS security groups allow all outbound by default, but if someone restricts egress (or uses NACLs), the app can't reach the database or NAT gateway
- **Forgetting to allow the LB health check**: The LB marks all targets unhealthy, returns 502/503 to users, but the application is actually running fine — a confusing failure mode

</details>

<details>
<summary>11. Configure IAM for a team of developers who need read-only access to production and full access to staging — show the CLI commands or policy JSON in both GCP (using roles and resource hierarchy) and AWS (using IAM policies and groups), demonstrate the least-privilege principle, and identify what a dangerous but common shortcut looks like in each platform.</summary>

**GCP approach — roles bound at the project level:**

```bash
# Create a Google Group for the dev team (managed in Google Workspace/Cloud Identity)
# Group: devs@company.com

# Read-only on production project — use predefined Viewer role
gcloud projects add-iam-policy-binding prod-project-id \
  --member="group:devs@company.com" \
  --role="roles/viewer"

# For specific services, use narrower roles instead of project-wide Viewer
gcloud projects add-iam-policy-binding prod-project-id \
  --member="group:devs@company.com" \
  --role="roles/logging.viewer"

gcloud projects add-iam-policy-binding prod-project-id \
  --member="group:devs@company.com" \
  --role="roles/monitoring.viewer"

# Full access on staging — use Editor role (or better, specific roles)
gcloud projects add-iam-policy-binding staging-project-id \
  --member="group:devs@company.com" \
  --role="roles/editor"
```

**Least-privilege refinement for GCP**: Instead of `roles/editor` on staging, grant the specific roles developers actually need — `roles/container.developer` for GKE, `roles/cloudsql.editor` for databases, `roles/run.developer` for Cloud Run. This prevents staging from becoming a playground where developers accidentally enable APIs or create resources they shouldn't.

**AWS approach — IAM groups with policies:**

StagingFullAccess policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "*",
      "Resource": "*"
    }
  ]
}
```

ProdReadOnly policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:Describe*",
        "rds:Describe*",
        "ecs:Describe*",
        "ecs:List*",
        "logs:GetLogEvents",
        "logs:FilterLogEvents",
        "logs:DescribeLogGroups",
        "cloudwatch:GetMetricData",
        "cloudwatch:DescribeAlarms"
      ],
      "Resource": "*"
    }
  ]
}
```

```bash
# Create group and attach policies
aws iam create-group --group-name Developers

# In a multi-account setup (recommended), use separate accounts for prod/staging
# Developers assume a read-only role in the prod account
aws iam create-role --role-name ProdReadOnlyRole \
  --assume-role-policy-document file://trust-policy.json

aws iam attach-role-policy --role-name ProdReadOnlyRole \
  --policy-arn arn:aws:iam::policy/ProdReadOnly

# Developers assume a full-access role in the staging account
aws iam create-role --role-name StagingDevRole \
  --assume-role-policy-document file://trust-policy.json

aws iam attach-role-policy --role-name StagingDevRole \
  --policy-arn arn:aws:iam::policy/StagingFullAccess
```

**Least-privilege refinement for AWS**: The `StagingFullAccess` policy above is still too broad — `"Action": "*"` grants everything including IAM changes. In a multi-account setup, the staging account itself provides the blast-radius boundary, but you should still use AWS managed policies like `PowerUserAccess` (everything except IAM) or build custom policies scoped to the services your team actually uses. Add a permission boundary to prevent privilege escalation (a developer creating a new role with more permissions than their own).

**Dangerous shortcuts:**

- **GCP**: Granting `roles/owner` at the folder level because "devs need access to multiple projects." Owner includes IAM admin — developers can grant themselves any permission, disable audit logs, or delete projects. The fix: use `roles/editor` at most, bound per-project, and use custom roles for production.
- **AWS**: Attaching `AdministratorAccess` to the Developers group in a single-account setup (no account-level isolation). Every developer can modify IAM, delete CloudTrail, make S3 buckets public, or spin up expensive resources. The fix: use separate accounts for prod/staging, with cross-account roles that are scoped appropriately. Even in staging, use permission boundaries.

</details>

<details>
<summary>12. A service in a private subnet cannot reach an external API and you suspect a networking issue — walk through the exact steps to diagnose the problem using VPC flow logs, checking route tables, NAT gateway status, and firewall rules/security groups, and explain how to read flow log entries to pinpoint whether traffic is being dropped and where.</summary>

Walk through this systematically, layer by layer, from the instance outward.

**Step 1: Verify the problem is networking, not application-level.**

SSH into the instance (or use SSM/IAP) and test basic connectivity:

```bash
# DNS resolution working?
nslookup api.external-service.com

# Can you reach the internet at all?
curl -v https://api.external-service.com/health
ping 8.8.8.8

# Is it just this API or all outbound traffic?
curl https://ifconfig.me  # shows your public IP (NAT gateway's IP)
```

If DNS fails, check VPC DNS settings (DHCP options set). If DNS resolves but connections time out, it's a routing or firewall issue.

**Step 2: Check the route table for the private subnet.**

```bash
# AWS
aws ec2 describe-route-tables --filters "Name=association.subnet-id,Values=$SUBNET_ID"
```

Look for a route `0.0.0.0/0` pointing to a NAT gateway (`nat-xxxxx`). If this route is missing, the subnet has no path to the internet. Common mistakes:
- Route table was never associated with the subnet (AWS uses the VPC's main route table by default, which may not have the NAT route)
- Route points to a NAT gateway in a different AZ that was deleted
- Route points to an internet gateway instead of NAT gateway (only works if instance has a public IP)

**Step 3: Check the NAT gateway status.**

```bash
aws ec2 describe-nat-gateways --filter "Name=state,Values=available"
```

Verify the NAT gateway is in `available` state (not `failed` or `deleted`). Check that it's in a public subnet with an Elastic IP. If the NAT gateway's subnet doesn't have a route to an internet gateway, the NAT gateway itself can't reach the internet — a common circular misconfiguration.

Also check NAT gateway CloudWatch metrics: `ErrorPortAllocation` means the NAT gateway ran out of simultaneous connections (default ~55,000 per destination). `PacketsDropCount` indicates traffic being dropped.

**Step 4: Check security groups and NACLs.**

```bash
# Security group on the instance — check outbound rules
aws ec2 describe-security-groups --group-ids $SG_ID

# NACLs on the subnet — check both inbound AND outbound rules
aws ec2 describe-network-acls --filters "Name=association.subnet-id,Values=$SUBNET_ID"
```

Security groups are stateful (if outbound is allowed, the response is automatically allowed). NACLs are stateless — you need explicit rules for both directions. Common NACL mistake: allowing outbound on port 443 but forgetting to allow inbound on ephemeral ports (1024-65535) for the response traffic.

**Step 5: Enable and read VPC flow logs.**

```bash
# Enable flow logs on the subnet's ENI or the entire VPC
aws ec2 create-flow-logs \
  --resource-type VPC \
  --resource-ids $VPC_ID \
  --traffic-type ALL \
  --log-destination-type cloud-watch-logs \
  --log-group-name /vpc/flow-logs
```

**Reading flow log entries:**

```
# Format: version account-id interface-id srcaddr dstaddr srcport dstport protocol packets bytes start end action log-status

2 123456789012 eni-abc123 10.0.10.15 203.0.113.50 49152 443 6 10 5000 1609459200 1609459260 ACCEPT OK
2 123456789012 eni-abc123 203.0.113.50 10.0.10.15 443 49152 6 0 0 1609459200 1609459260 REJECT OK
```

Key fields to look at:
- **srcaddr/dstaddr**: The source and destination IPs
- **action**: `ACCEPT` means the security group/NACL allowed it; `REJECT` means it was blocked
- **packets/bytes**: If both are 0, the traffic never actually flowed

In the example above: outbound traffic to the external API (port 443) was ACCEPTED, but the response was REJECTED. This tells you the outbound rules are fine but something is blocking the inbound return traffic — almost certainly a NACL missing an inbound rule for ephemeral ports.

**Step 6: GCP-specific differences.**

On GCP, use VPC Flow Logs (enabled per subnet) and check:
- Cloud NAT status and logs (Logging → Cloud NAT)
- Firewall rules (check both the instance's network tags and the rule priorities — a higher-priority DENY rule might override your ALLOW)
- Routes in the VPC (verify the default route `0.0.0.0/0` points to the default internet gateway and Cloud NAT is configured for the subnet's region)

**Common root causes in order of likelihood:**
1. Missing route to NAT gateway in the subnet's route table
2. Security group missing outbound rule (if default outbound-all was removed)
3. NACL blocking return traffic on ephemeral ports
4. NAT gateway in a failed state or in a subnet without internet gateway access
5. DNS resolution failure (VPC DNS disabled or custom DHCP options misconfigured)

</details>

<details>
<summary>13. Configure Workload Identity (GCP) or IRSA (AWS) so a Kubernetes pod can access cloud services without storing service account keys — show the setup steps, explain why this is preferred over mounting key files, and what breaks if the binding between the K8s service account and the cloud IAM role is misconfigured.</summary>

**GCP Workload Identity setup:**

```bash
# 1. Enable Workload Identity on the GKE cluster
gcloud container clusters update my-cluster \
  --workload-pool=my-project.svc.id.goog

# 2. Create a GCP service account with the permissions your app needs
gcloud iam service-accounts create my-app-sa \
  --display-name="My App Service Account"

gcloud projects add-iam-policy-binding my-project \
  --member="serviceAccount:my-app-sa@my-project.iam.gserviceaccount.com" \
  --role="roles/storage.objectViewer"

# 3. Create a Kubernetes service account
kubectl create serviceaccount my-app-ksa --namespace my-namespace

# 4. Bind the K8s SA to the GCP SA
gcloud iam service-accounts add-iam-policy-binding \
  my-app-sa@my-project.iam.gserviceaccount.com \
  --role="roles/iam.workloadIdentityUser" \
  --member="serviceAccount:my-project.svc.id.goog[my-namespace/my-app-ksa]"

# 5. Annotate the K8s SA to point to the GCP SA
kubectl annotate serviceaccount my-app-ksa \
  --namespace my-namespace \
  iam.gke.io/gcp-service-account=my-app-sa@my-project.iam.gserviceaccount.com
```

```yaml
# 6. Use the K8s SA in your pod spec
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  template:
    spec:
      serviceAccountName: my-app-ksa
      containers:
        - name: my-app
          image: gcr.io/my-project/my-app:latest
```

**AWS IRSA (IAM Roles for Service Accounts) setup:**

```bash
# 1. Associate an OIDC provider with the EKS cluster (one-time)
eksctl utils associate-iam-oidc-provider --cluster my-cluster --approve

# 2. Create an IAM role with a trust policy that allows the K8s SA
# The trust policy restricts which K8s namespace/SA can assume this role
cat > trust-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::ACCOUNT_ID:oidc-provider/oidc.eks.REGION.amazonaws.com/id/CLUSTER_ID"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "oidc.eks.REGION.amazonaws.com/id/CLUSTER_ID:sub": "system:serviceaccount:my-namespace:my-app-ksa",
          "oidc.eks.REGION.amazonaws.com/id/CLUSTER_ID:aud": "sts.amazonaws.com"
        }
      }
    }
  ]
}
EOF

aws iam create-role --role-name my-app-role --assume-role-policy-document file://trust-policy.json
aws iam attach-role-policy --role-name my-app-role --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

# 3. Create and annotate the K8s service account
kubectl create serviceaccount my-app-ksa --namespace my-namespace
kubectl annotate serviceaccount my-app-ksa \
  --namespace my-namespace \
  eks.amazonaws.com/role-arn=arn:aws:iam::ACCOUNT_ID:role/my-app-role
```

**Why this is preferred over mounting key files:**

- **No long-lived credentials**: Workload Identity and IRSA use short-lived tokens (automatically rotated) obtained via OIDC federation. Mounted key files are static — if leaked, they grant access until manually revoked.
- **No secret management burden**: Key files need to be created, stored securely (in a Secret, Vault, etc.), rotated periodically, and cleaned up. With WI/IRSA, there are no secrets to manage.
- **Blast radius containment**: Each pod gets exactly the permissions of its bound role. With a shared key file, every pod using that file gets the same permissions, and the key could be exfiltrated from any of them.
- **Auditability**: Cloud audit logs show which K8s service account (namespace + name) made each API call, giving you per-pod identity in cloud logs.

**What breaks if the binding is misconfigured:**

- **Wrong namespace or SA name in the trust policy/binding**: The pod gets a token, but when it presents the token to the cloud provider, the identity doesn't match the trust policy. The cloud API returns a 403 Forbidden or "access denied." The pod's application logs show authentication failures, not authorization failures — which is confusing because the pod "has" a service account.
- **Missing annotation on the K8s SA**: The pod runs with the node's default identity (the GKE node SA or the EKS node instance role), which may have broader permissions than intended — a silent privilege escalation.
- **OIDC provider not configured (EKS)**: The EKS cluster can't issue tokens that AWS recognizes. Pods fall back to the node's IAM role.
- **Workload Identity pool not enabled (GKE)**: The GKE metadata server doesn't intercept token requests. The pod either gets no credentials or falls back to the node's default SA.

The most dangerous failure mode is the **silent fallback** — the pod works because it inherits the node's broader permissions, so no one notices the misconfiguration until an audit reveals pods have more access than intended.

</details>

## Practical — Compute & Serverless

<details>
<summary>14. Deploy a containerized application to a managed Kubernetes cluster (GKE or EKS) and to a serverless container platform (Cloud Run or Fargate) — show the key CLI commands for each and explain the operational differences in scaling, logging, cost model, and deployment speed.</summary>

**Deploy to GKE:**

```bash
# Authenticate and get cluster credentials
gcloud container clusters get-credentials my-cluster --zone us-central1-a

# Deploy using a K8s manifest
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: my-app
          image: us-docker.pkg.dev/my-project/my-repo/my-app:v1.0
          ports:
            - containerPort: 3000
          resources:
            requests:
              cpu: 250m
              memory: 256Mi
            limits:
              cpu: 500m
              memory: 512Mi
---
apiVersion: v1
kind: Service
metadata:
  name: my-app
spec:
  type: LoadBalancer
  selector:
    app: my-app
  ports:
    - port: 80
      targetPort: 3000
EOF

# Set up autoscaling
kubectl autoscale deployment my-app --min=2 --max=10 --cpu-percent=70
```

**Deploy to Cloud Run:**

```bash
# One command — no cluster, no manifests
gcloud run deploy my-app \
  --image=us-docker.pkg.dev/my-project/my-repo/my-app:v1.0 \
  --port=3000 \
  --memory=512Mi \
  --cpu=1 \
  --min-instances=0 \
  --max-instances=100 \
  --allow-unauthenticated \
  --region=us-central1
```

**Operational differences** (for the full compute-layer tradeoff comparison including VMs and functions, see question 3):

| Aspect | GKE | Cloud Run |
|---|---|---|
| **Scaling** | HPA + node autoscaler. You configure thresholds and limits. Scaling takes seconds to minutes. | Automatic per-request, concurrency-based. Scales to zero when idle. No config beyond min/max instances. |
| **Logging** | Stdout/stderr to Cloud Logging automatically (agent pre-installed). You manage structured logging and alerting. | Stdout/stderr to Cloud Logging with zero setup. Request logs auto-captured with latency, status, and trace IDs. |
| **Cost model** | Pay per node-hour + management fee. Predictable but inefficient at low utilization. Committed-use discounts help. | Pay per CPU-seconds + memory-seconds + request count. $0 when idle. Expensive at sustained high load vs reserved nodes. |
| **Deployment speed** | Rolling updates: 30s-5min depending on readiness probes and replica count. | ~30s deploy. Instant rollback by routing to a previous revision. |

**When to choose each:**

- **GKE**: Long-running services with steady traffic, complex networking needs (service mesh, custom DNS), multiple interdependent services, or workloads requiring GPUs, persistent volumes, or fine-grained scheduling.
- **Cloud Run**: Stateless HTTP services with variable traffic, low-traffic microservices (cost-efficient at low volume), rapid prototyping, or when you want zero infrastructure management. Also good for running existing container images without Kubernetes knowledge.

</details>

<details>
<summary>15. Set up a container registry (Artifact Registry or ECR) for a production workload — show how to configure image lifecycle policies to automatically clean up untagged and old images, enable vulnerability scanning and act on the results, authenticate from a CI/CD pipeline (using OIDC, not stored keys), and authenticate from a Kubernetes cluster (Workload Identity / IRSA vs imagePullSecrets).</summary>

**Setting up Artifact Registry (GCP):**

```bash
# Create a Docker repository
gcloud artifacts repositories create my-app \
  --repository-format=docker \
  --location=us-central1 \
  --description="Production container images"
```

**Setting up ECR (AWS):**

```bash
aws ecr create-repository \
  --repository-name my-app \
  --image-scanning-configuration scanOnPush=true \
  --encryption-configuration encryptionType=KMS
```

**Image lifecycle policies:**

ECR has built-in lifecycle policies:

```bash
aws ecr put-lifecycle-policy --repository-name my-app --lifecycle-policy-text '{
  "rules": [
    {
      "rulePriority": 1,
      "description": "Remove untagged images after 1 day",
      "selection": {
        "tagStatus": "untagged",
        "countType": "sinceImagePushed",
        "countUnit": "days",
        "countNumber": 1
      },
      "action": { "type": "expire" }
    },
    {
      "rulePriority": 2,
      "description": "Keep only last 10 tagged images",
      "selection": {
        "tagStatus": "tagged",
        "tagPatternList": ["v*"],
        "countType": "imageCountMoreThan",
        "countNumber": 10
      },
      "action": { "type": "expire" }
    }
  ]
}'
```

Artifact Registry has a cleanup policy feature:

```bash
gcloud artifacts repositories set-cleanup-policies my-app \
  --location=us-central1 \
  --policy=cleanup-policy.json
```

```json
// cleanup-policy.json
[
  {
    "name": "delete-untagged",
    "action": { "type": "Delete" },
    "condition": {
      "tagState": "untagged",
      "olderThan": "86400s"
    }
  },
  {
    "name": "keep-recent-tagged",
    "action": { "type": "Keep" },
    "mostRecentVersions": {
      "keepCount": 10
    }
  }
]
```

**Vulnerability scanning:**

ECR scans on push (enabled above). Query results:

```bash
aws ecr describe-image-scan-findings \
  --repository-name my-app \
  --image-id imageTag=v1.0

# In CI/CD, fail the pipeline if critical vulnerabilities are found
CRITICAL=$(aws ecr describe-image-scan-findings \
  --repository-name my-app \
  --image-id imageTag=v1.0 \
  --query 'imageScanFindings.findingSeverityCounts.CRITICAL' \
  --output text)

if [ "$CRITICAL" != "None" ] && [ "$CRITICAL" != "0" ]; then
  echo "Critical vulnerabilities found — blocking deployment"
  exit 1
fi
```

Artifact Registry uses Artifact Analysis (formerly Container Analysis):

```bash
# Enable scanning
gcloud services enable containerscanning.googleapis.com

# Images are scanned automatically. Query results:
gcloud artifacts docker images list us-central1-docker.pkg.dev/my-project/my-app \
  --show-occurrences --occurrence-filter='kind="VULNERABILITY"'
```

**CI/CD authentication with OIDC (no stored keys):**

For GitHub Actions to GCP Artifact Registry:

```yaml
# .github/workflows/build.yml
jobs:
  build:
    permissions:
      contents: read
      id-token: write  # Required for OIDC
    steps:
      - uses: google-github-actions/auth@v2
        with:
          workload_identity_provider: 'projects/PROJECT_NUM/locations/global/workloadIdentityPools/github-pool/providers/github-provider'
          service_account: 'ci-sa@my-project.iam.gserviceaccount.com'

      - name: Configure Docker
        run: gcloud auth configure-docker us-central1-docker.pkg.dev

      - name: Push image
        run: docker push us-central1-docker.pkg.dev/my-project/my-app/my-app:${{ github.sha }}
```

For GitHub Actions to ECR:

```yaml
jobs:
  build:
    permissions:
      id-token: write
      contents: read
    steps:
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::ACCOUNT_ID:role/GitHubActionsRole
          aws-region: us-east-1

      - uses: aws-actions/amazon-ecr-login@v2

      - name: Push image
        run: docker push ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/my-app:${{ github.sha }}
```

**K8s cluster authentication — Workload Identity / IRSA vs imagePullSecrets:**

With **Workload Identity (GKE)**: If the GKE cluster and Artifact Registry are in the same project, GKE nodes can pull images by default (the default compute SA has read access). For cross-project access, grant `roles/artifactregistry.reader` to the node SA or the Workload Identity SA.

With **IRSA (EKS)**: EKS nodes need an instance profile with `ecr:GetAuthorizationToken` and `ecr:BatchGetImage` permissions. For cross-account ECR access, add a repository policy allowing the node role.

**imagePullSecrets** (the old way): You create a Kubernetes Secret with registry credentials and reference it in pod specs. This requires rotating the credentials manually, storing them securely, and distributing them to every namespace. Workload Identity/IRSA eliminates all of this — the kubelet gets short-lived tokens automatically. Use imagePullSecrets only for third-party registries that don't support OIDC federation.

</details>

## Practical — Data & Messaging

<details>
<summary>16. Set up a managed PostgreSQL database (Cloud SQL or RDS) with high availability — show the CLI commands for creating the instance with a read replica, configuring automated backups, and setting up failover, and explain the difference between multi-AZ (AWS) and regional (GCP) HA configurations and what happens during a failover event.</summary>

**AWS RDS setup:**

```bash
# Create primary instance with Multi-AZ and automated backups
aws rds create-db-instance \
  --db-instance-identifier my-app-db \
  --db-instance-class db.r6g.large \
  --engine postgres \
  --engine-version 16.4 \
  --master-username admin \
  --master-user-password "$DB_PASSWORD" \
  --allocated-storage 100 \
  --storage-type gp3 \
  --multi-az \
  --db-subnet-group-name my-private-subnet-group \
  --vpc-security-group-ids $DB_SG \
  --backup-retention-period 14 \
  --preferred-backup-window "03:00-04:00" \
  --storage-encrypted \
  --kms-key-id $KMS_KEY

# Create a read replica (for read scaling — separate from Multi-AZ HA)
aws rds create-db-instance-read-replica \
  --db-instance-identifier my-app-db-read \
  --source-db-instance-identifier my-app-db \
  --db-instance-class db.r6g.large \
  --availability-zone us-east-1b
```

**GCP Cloud SQL setup:**

```bash
# Create primary with HA (regional) and automated backups
gcloud sql instances create my-app-db \
  --database-version=POSTGRES_16 \
  --tier=db-custom-4-16384 \
  --region=us-central1 \
  --availability-type=REGIONAL \
  --storage-type=SSD \
  --storage-size=100GB \
  --backup-start-time=03:00 \
  --retained-backups-count=14 \
  --enable-point-in-time-recovery \
  --network=projects/my-project/global/networks/my-vpc \
  --no-assign-ip  # private IP only

# Create a read replica
gcloud sql instances create my-app-db-read \
  --master-instance-name=my-app-db \
  --tier=db-custom-4-16384 \
  --region=us-central1
```

**Multi-AZ (AWS) vs Regional (GCP) HA:**

Both achieve the same goal (automatic failover within a region) but work differently:

**AWS Multi-AZ**: RDS maintains a synchronous standby replica in a different AZ. The standby is not accessible for reads — it exists purely for failover. Replication is at the storage level (synchronous block-level replication). The primary and standby share a single DNS endpoint; during failover, AWS flips the DNS to the standby.

**GCP Regional**: Cloud SQL maintains a standby instance in a different zone within the region. Like AWS, the standby is not accessible for reads. Replication uses synchronous disk-level replication. During failover, the instance keeps the same IP address (no DNS change needed).

**Key difference**: Read replicas are separate from HA in both platforms. Multi-AZ/Regional gives you failover. Read replicas give you read scaling. They serve different purposes and you typically want both.

**What happens during a failover event:**

1. **Detection**: The platform detects the primary is unhealthy (hardware failure, zone outage, unresponsive instance, OS patching). Detection takes 30-120 seconds depending on the failure type.
2. **Promotion**: The standby is promoted to primary. This takes:
   - AWS: Typically 60-120 seconds. DNS TTL is 5 seconds, but application connection pools may cache the old IP longer.
   - GCP: Typically 30-120 seconds. IP stays the same, so reconnection is faster.
3. **Application impact**: Any in-flight transactions on the old primary are lost. Connections are dropped. Applications must handle reconnection — use a connection pool that retries on disconnect. Expect 1-3 minutes of downtime.
4. **After failover**: The old primary is replaced with a new standby in the original zone. Read replicas continue replicating from the new primary (may need to catch up on any replication lag).

**Gotchas:**
- Failover is NOT instant — plan for 1-3 minutes of database unavailability. Your application needs retry logic and circuit breakers.
- Failover can be triggered by maintenance (patching), not just failures. Schedule maintenance windows during low-traffic periods.
- Test failover before you need it: `aws rds reboot-db-instance --db-instance-identifier my-app-db --force-failover` or `gcloud sql instances failover my-app-db`.

</details>

## Practical — High Availability & Operations

<details>
<summary>17. Design a multi-zone architecture for a stateless web application with a database — show how to configure regional instance groups or auto-scaling groups across zones, set up a global or regional load balancer, and configure database replication so a single zone failure doesn't cause downtime, then walk through what happens during a zone outage.</summary>

**Architecture overview:**

```
                    Global/Regional Load Balancer
                    ┌─────────┴─────────┐
                Zone A                Zone B
            ┌──────────┐          ┌──────────┐
            │ App (x2) │          │ App (x2) │
            └────┬─────┘          └────┬─────┘
                 │                     │
            ┌────┴─────────────────────┴────┐
            │   Regional Database (HA)       │
            │   Primary (Zone A)             │
            │   Standby (Zone B)             │
            └───────────────────────────────┘
```

**GCP — Regional Managed Instance Group + Global LB:**

```bash
# 1. Create an instance template
gcloud compute instance-templates create my-app-template \
  --machine-type=e2-medium \
  --image-family=cos-stable \
  --image-project=cos-cloud \
  --tags=app-server \
  --metadata=startup-script='docker run -p 3000:3000 us-docker.pkg.dev/my-project/my-repo/my-app:latest'

# 2. Create a REGIONAL managed instance group (spans multiple zones automatically)
gcloud compute instance-groups managed create my-app-mig \
  --template=my-app-template \
  --size=4 \
  --region=us-central1 \
  --zones=us-central1-a,us-central1-b

# 3. Configure autoscaling
gcloud compute instance-groups managed set-autoscaling my-app-mig \
  --region=us-central1 \
  --min-num-replicas=2 \
  --max-num-replicas=10 \
  --target-cpu-utilization=0.7

# 4. Create a global HTTP(S) load balancer
gcloud compute health-checks create http my-app-hc \
  --port=3000 --request-path=/health

gcloud compute backend-services create my-app-backend \
  --protocol=HTTP --port-name=http --health-checks=my-app-hc --global

gcloud compute backend-services add-backend my-app-backend \
  --instance-group=my-app-mig --instance-group-region=us-central1 --global

gcloud compute url-maps create my-app-urlmap \
  --default-service=my-app-backend

gcloud compute target-http-proxies create my-app-proxy \
  --url-map=my-app-urlmap

gcloud compute forwarding-rules create my-app-fr \
  --global --target-http-proxy=my-app-proxy --ports=80
```

**AWS — Auto Scaling Group across AZs + ALB:**

```bash
# 1. Create a launch template
aws ec2 create-launch-template \
  --launch-template-name my-app-lt \
  --launch-template-data '{
    "ImageId": "ami-xxxxx",
    "InstanceType": "t3.medium",
    "SecurityGroupIds": ["'$APP_SG'"],
    "UserData": "'$(base64 <<< '#!/bin/bash\ndocker run -p 3000:3000 ACCOUNT.dkr.ecr.us-east-1.amazonaws.com/my-app:latest')'"  # UserData must be base64-encoded for the EC2 API
  }'

# 2. Create ASG spanning multiple AZs
aws autoscaling create-auto-scaling-group \
  --auto-scaling-group-name my-app-asg \
  --launch-template LaunchTemplateName=my-app-lt,Version='$Latest' \
  --min-size 2 --max-size 10 --desired-capacity 4 \
  --vpc-zone-identifier "$APP_SUBNET_1A,$APP_SUBNET_1B" \
  --target-group-arns $TARGET_GROUP_ARN \
  --health-check-type ELB \
  --health-check-grace-period 120

# 3. Create ALB
aws elbv2 create-load-balancer \
  --name my-app-alb \
  --subnets $PUBLIC_1A $PUBLIC_1B \
  --security-groups $ALB_SG

aws elbv2 create-target-group \
  --name my-app-tg \
  --protocol HTTP --port 3000 \
  --vpc-id $VPC_ID \
  --health-check-path /health \
  --health-check-interval-seconds 15 \
  --healthy-threshold-count 2

aws elbv2 create-listener \
  --load-balancer-arn $ALB_ARN \
  --protocol HTTP --port 80 \
  --default-actions Type=forward,TargetGroupArn=$TG_ARN
```

**Database HA**: As covered in question 16 — use Multi-AZ (AWS) or Regional (GCP) for automatic failover across zones.

**What happens during a zone outage:**

1. **Immediate (0-30s)**: Instances in the failed zone become unreachable. The load balancer's health checks start failing for those instances.
2. **Health check failure (30-90s)**: After the configured unhealthy threshold (e.g., 3 consecutive failures at 10s intervals), the LB stops routing traffic to failed-zone instances. Remaining instances in the healthy zone handle all traffic — this is why you run enough capacity in each zone to absorb the other zone's load.
3. **ASG/MIG response (1-5min)**: The autoscaler detects instances as unhealthy and launches replacements. In AWS, the ASG tries to rebalance across AZs — if the failed AZ is unavailable, new instances launch in the healthy AZ. GCP's regional MIG does the same.
4. **Database failover (1-3min)**: If the primary database was in the failed zone, automatic failover promotes the standby (as covered in question 16). Applications experience connection drops and need to reconnect.
5. **Recovery (5-15min)**: New application instances pass health checks and start receiving traffic. The system runs at full capacity in the surviving zone(s). When the failed zone recovers, instances are rebalanced back.

**Critical design consideration**: Each zone must be able to handle the full load independently during a zone failure. If you normally run 4 instances across 2 zones, each zone needs enough headroom to serve 100% of traffic with just 2 instances. Size your autoscaler's max capacity and your instance sizes with this in mind.

</details>

---

## Experience-Based Questions

These questions test real-world experience. Prepare by mapping them to your own projects and situations.

<details>
<summary>18. Tell me about a time you migrated an application between cloud services or redesigned a cloud architecture — what drove the decision, what tradeoffs did you evaluate, what went wrong during the migration, and what would you do differently?</summary>

**What the interviewer is looking for:**
- Ability to evaluate tradeoffs systematically (not just "the new thing was better")
- Risk management during migration (how you avoided downtime)
- Honest reflection on what went wrong and what you learned
- Decision-making process, not just the technical steps

**Suggested structure (STAR+L):**

1. **Situation**: What was the original architecture and what problem emerged? (cost, scale limits, operational burden, compliance)
2. **Task**: What was your role in evaluating and executing the migration?
3. **Action**: What alternatives did you consider? What tradeoffs did you weigh? How did you execute — big bang or incremental? How did you handle data migration and the cutover?
4. **Result**: What improved? What metrics changed? (cost, latency, operational incidents, deployment speed)
5. **Learnings**: What went wrong? What would you do differently?

**Example outline to personalize:**

- *Situation*: "We ran a Node.js API on self-managed VMs with manual scaling. Deployments took 30 minutes, scaling was reactive (alerts → manual intervention), and we spent 2+ hours/week on OS patching and security updates."
- *Task*: "I led the evaluation and migration to a container-based platform — we considered GKE, Cloud Run, and staying on VMs with better automation."
- *Action*: "We evaluated based on: team expertise (limited K8s experience), traffic pattern (variable, with spikes), cost at our scale, and migration complexity. Chose Cloud Run because it eliminated infrastructure management entirely and our service was stateless. Migrated incrementally — ran both platforms in parallel behind a load balancer, shifting traffic 10% at a time over 2 weeks."
- *Result*: "Deployments went from 30 minutes to 2 minutes. Scaling became automatic. Infrastructure costs dropped 40% (scale-to-zero during off-hours). Eliminated all patching work."
- *Learnings*: "We underestimated connection pooling differences — Cloud Run instances are ephemeral, so our database connection pool settings caused connection storms during scale-up. We should have load-tested the new platform under realistic traffic before shifting production traffic."

**Key points to hit:**
- Show you evaluated multiple options, not just picked the trendy one
- Demonstrate you planned for risk (parallel run, incremental cutover, rollback plan)
- Be specific about what went wrong — interviewers distrust "everything went smoothly"
- Connect the migration to a business outcome (cost, reliability, velocity), not just technical preference

</details>

<details>
<summary>19. Describe a time you debugged a significant cloud infrastructure issue in production — what were the symptoms, how did you narrow down whether it was a networking, compute, or configuration problem, and what was the root cause?</summary>

**What the interviewer is looking for:**
- Systematic debugging methodology (not guessing randomly)
- Ability to triage across infrastructure layers (network, compute, config, application)
- Calm under pressure — how you handled the incident, not just the technical fix
- Understanding of observability tools and what signals to look for

**Suggested structure:**

1. **Symptoms**: What alerted you? (monitoring alerts, user reports, error rate spike, latency increase)
2. **Triage**: How did you determine the scope? (all users or some? one region or all? one service or many?)
3. **Investigation layers**: Walk through how you systematically narrowed down the cause — checking metrics, logs, network, and config in a logical order
4. **Root cause**: What was actually broken and why?
5. **Resolution and prevention**: How did you fix it and what did you change to prevent recurrence?

**Example outline to personalize:**

- *Symptoms*: "Our API started returning 504 Gateway Timeout errors for ~30% of requests. The error rate spiked from 0.1% to 30% in 5 minutes. CloudWatch alarms fired for elevated 5xx rates and increased p99 latency."
- *Triage*: "Checked if it was deployment-related (no recent deploys). Checked all regions (only us-east-1 affected). Checked all services (only the order service, not auth or inventory)."
- *Investigation*: "Application logs showed database connection timeouts. Database metrics showed the primary was healthy (CPU 20%, connections 50/200). Checked security groups — no recent changes. Checked VPC flow logs for traffic between app subnet and database subnet — saw REJECT entries for traffic from two of our four app instances. Traced the issue to a Terraform apply that had modified a security group rule — it replaced the app security group reference with a CIDR block that didn't include the new subnet we'd recently added for scaling."
- *Root cause*: "A Terraform change to the database security group used a hardcoded CIDR instead of referencing the app security group ID. When autoscaling launched instances in a new subnet (different CIDR), those instances couldn't reach the database."
- *Resolution*: "Immediate fix: updated the security group to reference the app SG instead of CIDR. Prevention: added a policy requiring SG-to-SG references instead of CIDR blocks in Terraform reviews. Added a connectivity health check that verifies database reachability from every app instance."

**Key points to hit:**
- Show a systematic approach: metrics first, then narrow the scope, then dive deep
- Name specific tools you used (CloudWatch, VPC flow logs, `kubectl logs`, etc.)
- Explain your reasoning at each step — why you checked X before Y
- Include the prevention step — fixing the root cause matters more than fixing the symptom

</details>

<details>
<summary>20. Describe a time you designed a cloud architecture for high availability or disaster recovery — what availability target did you set, what architecture decisions did you make to meet it, and how did you test that failover actually worked?</summary>

**What the interviewer is looking for:**
- Understanding of availability targets and what they mean operationally (99.9% vs 99.99% — the budget of allowed downtime)
- Architecture decisions driven by requirements, not over-engineering
- Practical testing of failover — not just theoretical design
- Cost-awareness: HA has a price, and you should know the tradeoff

**Suggested structure:**

1. **Context**: What was the application, why did it need HA, and what was the availability target?
2. **Design decisions**: What architecture did you choose and why? What did you trade off?
3. **Implementation**: Key components and how they work together for HA
4. **Testing**: How did you validate that failover actually works?
5. **Outcome**: Did it work in practice? Any real incidents that tested it?

**Example outline to personalize:**

- *Context*: "We ran a payment processing service that needed 99.95% availability (~22 min downtime/month max). Previous architecture was single-AZ — one zone outage caused a 45-minute outage that breached our SLA."
- *Design decisions*: "We chose multi-AZ within a single region (not multi-region) because: our latency requirements didn't need global distribution, our users were concentrated in one region, and multi-region would have doubled our database costs and added cross-region consistency complexity. The tradeoff: a full regional outage (rare but possible) would cause downtime."
- *Implementation*: "Stateless app tier across 3 AZs behind an ALB with health checks (as covered in question 17). RDS Multi-AZ for the database with automated failover. Redis ElastiCache with Multi-AZ and automatic failover. All infrastructure defined in Terraform so we could recreate the stack in a different region if needed (DR, not HA)."
- *Testing*: "We ran quarterly 'game days' where we simulated failures:
  1. Terminated instances in one AZ during peak traffic — verified the ALB routed around them within 30 seconds
  2. Forced an RDS failover (`--force-failover`) — measured 90 seconds of database unavailability, verified application reconnected automatically
  3. Blocked traffic between app subnet and database subnet via security group — verified circuit breakers activated and returned graceful errors instead of hanging
  4. Ran chaos engineering with AWS Fault Injection Service to inject random AZ failures monthly"
- *Outcome*: "Over 12 months, we experienced two AZ-level issues. Both times, traffic shifted automatically with no user-visible impact. Actual availability was 99.98%. The game days were critical — our first test revealed that our connection pool didn't retry on failover, which would have caused a real outage."

**Key points to hit:**
- Tie the availability target to a business reason (SLA, revenue impact, user trust)
- Show you made deliberate tradeoffs (multi-AZ vs multi-region, cost vs resilience)
- Emphasize testing — an untested failover plan is not a plan
- Mention something the testing revealed that you fixed before a real incident

</details>
