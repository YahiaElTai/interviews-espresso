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

<!-- Answer will be added later -->

</details>

<details>
<summary>2. What is the shared responsibility model and how does it shift as you move from IaaS (VMs) to PaaS (managed databases, App Engine/Elastic Beanstalk) to serverless (Cloud Functions/Lambda) — what specific responsibilities move from you to the provider at each level, and what mistakes do teams make by assuming the provider handles something they actually don't?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>3. Why does the cloud compute spectrum exist — from VMs (Compute Engine/EC2) to managed containers (GKE/EKS, Cloud Run/Fargate) to functions (Cloud Functions/Lambda) — what are the real tradeoffs in control, cost model, cold start behavior, and operational burden, and how do you decide which layer fits a given workload instead of defaulting to one?</summary>

<!-- Answer will be added later -->

</details>

## Conceptual Depth — Identity & Networking

<details>
<summary>4. How do IAM models differ between GCP and AWS — why does GCP use a resource hierarchy with inherited roles while AWS uses policy documents attached to principals, what does least-privilege look like in each model, and what are the most common over-permissioning mistakes teams make on each platform?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>5. Why does cloud networking require VPCs with public and private subnets instead of just putting everything on the internet — what problems do NAT gateways solve, and what does a well-designed network topology look like for a typical web application with a public frontend and private backend?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>6. How do cloud load balancers work at L4 vs L7 — what are the tradeoffs between network load balancers and application load balancers, when would you choose each, what role do Cloud Armor (GCP) and WAF (AWS) play in protecting your application at the load balancer layer, how does SSL termination work at the load balancer layer, and how do health check configurations affect traffic routing?</summary>

<!-- Answer will be added later -->

</details>

## Conceptual Depth — Data, Storage & Messaging

<details>
<summary>7. How do you choose between managed database options on GCP and AWS — what are the tradeoffs between relational (Cloud SQL/RDS), globally distributed (Spanner/Aurora), document (Firestore/DynamoDB), and in-memory (Memorystore/ElastiCache), and when does each become the right or wrong choice for a given workload?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>8. When should you use managed messaging services (Pub/Sub, SNS/SQS, EventBridge) vs self-hosted alternatives (Kafka, RabbitMQ) — what are the delivery guarantee differences, how do their pricing models compare at scale, and what capabilities does EventBridge add that simple pub/sub doesn't cover?</summary>

<!-- Answer will be added later -->

</details>

## Conceptual Depth — Compute & Containers

<details>
<summary>9. What are the meaningful differences between GKE and EKS — how do they differ in networking models, node autoscaling, control plane management, and cluster upgrades, and what does each platform manage for you vs leave to you that affects day-to-day operations?</summary>

<!-- Answer will be added later -->

</details>

## Practical — Networking & Security

<details>
<summary>10. Design a VPC architecture for a web application with a public-facing load balancer, private application servers, and a private database tier — show the subnet layout, NAT gateway placement, and firewall rules (or security groups) using either gcloud or AWS CLI commands, and explain what breaks if you skip the NAT gateway or misconfigure the firewall rules.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>11. Configure IAM for a team of developers who need read-only access to production and full access to staging — show the CLI commands or policy JSON in both GCP (using roles and resource hierarchy) and AWS (using IAM policies and groups), demonstrate the least-privilege principle, and identify what a dangerous but common shortcut looks like in each platform.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>12. A service in a private subnet cannot reach an external API and you suspect a networking issue — walk through the exact steps to diagnose the problem using VPC flow logs, checking route tables, NAT gateway status, and firewall rules/security groups, and explain how to read flow log entries to pinpoint whether traffic is being dropped and where.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>13. Configure Workload Identity (GCP) or IRSA (AWS) so a Kubernetes pod can access cloud services without storing service account keys — show the setup steps, explain why this is preferred over mounting key files, and what breaks if the binding between the K8s service account and the cloud IAM role is misconfigured.</summary>

<!-- Answer will be added later -->

</details>

## Practical — Compute & Serverless

<details>
<summary>14. Deploy a containerized application to a managed Kubernetes cluster (GKE or EKS) and to a serverless container platform (Cloud Run or Fargate) — show the key CLI commands for each and explain the operational differences in scaling, logging, cost model, and deployment speed.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>15. Set up a container registry (Artifact Registry or ECR) for a production workload — show how to configure image lifecycle policies to automatically clean up untagged and old images, enable vulnerability scanning and act on the results, authenticate from a CI/CD pipeline (using OIDC, not stored keys), and authenticate from a Kubernetes cluster (Workload Identity / IRSA vs imagePullSecrets).</summary>

<!-- Answer will be added later -->

</details>

## Practical — Data & Messaging

<details>
<summary>16. Set up a managed PostgreSQL database (Cloud SQL or RDS) with high availability — show the CLI commands for creating the instance with a read replica, configuring automated backups, and setting up failover, and explain the difference between multi-AZ (AWS) and regional (GCP) HA configurations and what happens during a failover event.</summary>

<!-- Answer will be added later -->

</details>

## Practical — High Availability & Operations

<details>
<summary>17. Design a multi-zone architecture for a stateless web application with a database — show how to configure regional instance groups or auto-scaling groups across zones, set up a global or regional load balancer, and configure database replication so a single zone failure doesn't cause downtime, then walk through what happens during a zone outage.</summary>

<!-- Answer will be added later -->

</details>

---

## Experience-Based Questions

These questions test real-world experience. Prepare by mapping them to your own projects and situations.

<details>
<summary>18. Tell me about a time you migrated an application between cloud services or redesigned a cloud architecture — what drove the decision, what tradeoffs did you evaluate, what went wrong during the migration, and what would you do differently?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>19. Describe a time you debugged a significant cloud infrastructure issue in production — what were the symptoms, how did you narrow down whether it was a networking, compute, or configuration problem, and what was the root cause?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>20. Describe a time you designed a cloud architecture for high availability or disaster recovery — what availability target did you set, what architecture decisions did you make to meet it, and how did you test that failover actually worked?</summary>

<!-- Answer framework will be added later -->

</details>
