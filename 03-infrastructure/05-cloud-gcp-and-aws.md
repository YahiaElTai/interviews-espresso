# Cloud — GCP & AWS

> **36 questions** — 12 theory, 24 practical

- Cloud organization: regions, zones, projects/accounts — GCP vs AWS structural differences
- Shared responsibility model across IaaS, PaaS, and serverless
- IAM: GCP resource hierarchy + roles vs AWS policy documents + principals — least-privilege patterns, Workload Identity/IRSA for K8s-to-cloud auth
- VPC architecture: subnets (public/private), NAT gateways, route tables, firewall rules vs security groups vs NACLs, VPC flow logs for troubleshooting
- Cloud load balancing: L4 vs L7 tradeoffs, application vs network load balancers, Cloud Armor/WAF, SSL termination, health checks
- DNS management: Cloud DNS/Route 53, record types (A, CNAME, ALIAS), TTL tradeoffs, domain verification, private DNS zones
- Private connectivity: VPC peering, Private Service Connect/PrivateLink, shared VPCs — connecting services without public internet
- Compute tradeoffs: VMs vs containers (GKE/EKS) vs serverless (Cloud Run/Fargate, Functions/Lambda)
- Managed databases: Cloud SQL/RDS, Spanner/Aurora, Firestore/DynamoDB, Memorystore/ElastiCache
- Storage tiers: GCS/S3, block storage, file storage — lifecycle policies and cost implications
- Managed messaging and eventing: Pub/Sub, SNS/SQS, EventBridge — when to use managed vs self-hosted queues
- GKE vs EKS: VPC-native vs overlay networking, node autoscaling (NAP vs Karpenter), control plane upgrades, managed add-ons — what each abstracts vs leaves to you
- Container registries: Artifact Registry/ECR, image lifecycle policies, vulnerability scanning, authentication from CI/CD and K8s
- Serverless event-driven pipelines: triggers, retry behavior, dead-letter handling, idempotency
- Cost optimization: committed use discounts, savings plans, spot/preemptible instances, right-sizing
- CDN setup: Cloud CDN/CloudFront, cache rules, origin shielding, cache invalidation
- Cross-region and multi-zone high availability: regional resources, global load balancing, replication
- Cloud billing diagnosis, budget alerts, and anomaly detection
- Serverless debugging: cold starts, timeouts, Cloud Logging/CloudWatch, Cloud Trace/X-Ray
- Cloud security posture: detecting public resources, service account hygiene, organization policies basics

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
<summary>6. How do GCP firewall rules differ from AWS security groups and NACLs in their evaluation model — why does GCP use a single firewall rule system while AWS separates stateful security groups from stateless NACLs, and what are the practical implications for debugging connectivity issues?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>7. How do cloud load balancers work at L4 vs L7 — what are the tradeoffs between network load balancers and application load balancers, when would you choose each, what role do Cloud Armor (GCP) and WAF (AWS) play in protecting your application at the load balancer layer, how does SSL termination work at the load balancer layer, and how do health check configurations affect traffic routing?</summary>

<!-- Answer will be added later -->

</details>

## Conceptual Depth — Data, Storage & Messaging

<details>
<summary>8. How do you choose between managed database options on GCP and AWS — what are the tradeoffs between relational (Cloud SQL/RDS), globally distributed (Spanner/Aurora), document (Firestore/DynamoDB), and in-memory (Memorystore/ElastiCache), and when does each become the right or wrong choice for a given workload?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>9. How do cloud storage tiers work across GCS and S3 — what are the differences between object storage, block storage (Persistent Disk/EBS), and file storage (Filestore/EFS), how do lifecycle policies automate cost savings by transitioning data between tiers, and what cost traps do teams fall into with storage?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>10. When should you use managed messaging services (Pub/Sub, SNS/SQS, EventBridge) vs self-hosted alternatives (Kafka, RabbitMQ) — what are the delivery guarantee differences, how do their pricing models compare at scale, and what capabilities does EventBridge add that simple pub/sub doesn't cover?</summary>

<!-- Answer will be added later -->

</details>

## Conceptual Depth — Compute & Containers

<details>
<summary>11. What are the meaningful differences between GKE and EKS — how do they differ in networking models, node autoscaling, control plane management, and cluster upgrades, and what does each platform manage for you vs leave to you that affects day-to-day operations?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>12. Why do serverless event-driven pipelines need special attention to retry behavior, dead-letter handling, and idempotency — what happens when a Cloud Function or Lambda is triggered multiple times for the same event, how do you design pipelines that handle partial failures gracefully, and what are the common pitfalls teams hit when chaining serverless functions together?</summary>

<!-- Answer will be added later -->

</details>

## Practical — Networking & Security

<details>
<summary>13. Design a VPC architecture for a web application with a public-facing load balancer, private application servers, and a private database tier — show the subnet layout, NAT gateway placement, and firewall rules (or security groups) using either gcloud or AWS CLI commands, and explain what breaks if you skip the NAT gateway or misconfigure the firewall rules.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>14. Configure IAM for a team of developers who need read-only access to production and full access to staging — show the CLI commands or policy JSON in both GCP (using roles and resource hierarchy) and AWS (using IAM policies and groups), demonstrate the least-privilege principle, and identify what a dangerous but common shortcut looks like in each platform.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>15. A service in a private subnet cannot reach an external API and you suspect a networking issue — walk through the exact steps to diagnose the problem using VPC flow logs, checking route tables, NAT gateway status, and firewall rules/security groups, and explain how to read flow log entries to pinpoint whether traffic is being dropped and where.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>16. Set up DNS for a web application using Cloud DNS or Route 53 — show how to create A records and CNAME records, explain when to use ALIAS records instead of CNAMEs, what TTL values to set for different record types and why, how to configure a private DNS zone for internal service discovery, and what breaks if TTL is set too high during a migration.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>17. Connect two services running in separate VPCs without routing traffic over the public internet — show how to set up VPC peering and explain when you'd use Private Service Connect (GCP) or PrivateLink (AWS) instead, what the tradeoffs are between peering and private endpoints, and when shared VPCs (GCP) or Transit Gateway (AWS) become necessary.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>18. Configure Workload Identity (GCP) or IRSA (AWS) so a Kubernetes pod can access cloud services without storing service account keys — show the setup steps, explain why this is preferred over mounting key files, and what breaks if the binding between the K8s service account and the cloud IAM role is misconfigured.</summary>

<!-- Answer will be added later -->

</details>

## Practical — Compute & Serverless

<details>
<summary>19. Deploy a containerized application to a managed Kubernetes cluster (GKE or EKS) and to a serverless container platform (Cloud Run or Fargate) — show the key CLI commands for each and explain the operational differences in scaling, logging, cost model, and deployment speed.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>20. When would you choose deploying containers on a VM with Docker (Compute Engine/EC2) over managed options like GKE/EKS or Cloud Run/Fargate — what are the tradeoffs in control, cost, and operational burden, and show the setup commands for running a container on a VM?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>21. Build a serverless event-driven pipeline where a file upload to GCS/S3 triggers a function that processes the file, writes results to a database, and sends a notification — show the configuration for the trigger, retry policy, and dead-letter queue, and explain how you ensure idempotency so reprocessing the same event doesn't corrupt data.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>22. A serverless function is experiencing cold starts of 5+ seconds and intermittent timeouts — walk through how you diagnose and fix this using Cloud Logging/CloudWatch and Cloud Trace/X-Ray, explain what causes cold starts (runtime size, dependencies, VPC attachment), what the key differences in observability are between serverless and container-based workloads, and what mitigation strategies exist (provisioned concurrency, min instances, reducing bundle size).</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>23. Set up a container registry (Artifact Registry or ECR) with image lifecycle policies that automatically clean up untagged images older than 30 days — show the CLI commands, explain how to enable vulnerability scanning, and configure authentication so both your CI/CD pipeline and Kubernetes cluster can pull images securely.</summary>

<!-- Answer will be added later -->

</details>

## Practical — Data, CDN & Messaging

<details>
<summary>24. Set up a managed PostgreSQL database (Cloud SQL or RDS) with high availability — show the CLI commands for creating the instance with a read replica, configuring automated backups, and setting up failover, and explain the difference between multi-AZ (AWS) and regional (GCP) HA configurations and what happens during a failover event.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>25. Configure a GCS or S3 bucket with lifecycle policies that automatically transition objects from standard storage to nearline/infrequent access after 30 days and to coldline/glacier after 90 days — show the lifecycle rule configuration, explain how retrieval costs change per tier, and identify the common cost trap of storing many small files in cold tiers.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>26. Set up a CDN (Cloud CDN or CloudFront) in front of an application load balancer — explain why origin shielding matters and the tradeoffs of aggressive vs conservative caching, show the configuration for cache rules, custom cache keys, and origin shielding, demonstrate how to invalidate specific paths when content changes, and explain why wildcard invalidation is expensive and what alternatives exist.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>27. Configure a managed messaging fanout pattern where a single event is delivered to multiple consumers — show the setup using either GCP Pub/Sub (topic with multiple subscriptions) or AWS SNS+SQS (topic fanning out to multiple queues), and explain how dead-letter handling and retry policies differ between the two platforms.</summary>

<!-- Answer will be added later -->

</details>

## Practical — High Availability & Operations

<details>
<summary>28. Design a multi-zone architecture for a stateless web application with a database — show how to configure regional instance groups or auto-scaling groups across zones, set up a global or regional load balancer, and configure database replication so a single zone failure doesn't cause downtime, then walk through what happens during a zone outage.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>29. Configure a GKE or EKS cluster and highlight the key setup decisions that differ between the two — show the CLI commands for cluster creation, explain the differences in networking mode (VPC-native vs overlay), node auto-provisioning vs Karpenter, and control plane upgrade strategy, and identify what each platform handles automatically that the other leaves to you.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>30. Set up budget alerts and billing anomaly detection — show how to create a budget with alert thresholds in either GCP or AWS, configure notifications to Slack or email, and explain how to use billing exports and cost dashboards to diagnose an unexpected cost spike (identifying which service, region, or resource caused it).</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>31. Your cloud bill jumped 40% month-over-month and leadership wants an explanation — walk through the exact steps to diagnose the cause using billing dashboards and cost breakdowns, show how to identify the top cost drivers by service and resource, and demonstrate how you'd implement right-sizing recommendations and committed use discounts to bring costs back down.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>32. Audit the security posture of a cloud project/account — walk through how you detect publicly accessible resources (open storage buckets, VMs with public IPs, overly permissive firewall rules), review service account usage for default or over-privileged accounts, and show what organization policies or SCPs you'd enable to prevent these issues from recurring.</summary>

<!-- Answer will be added later -->

</details>

---

## Experience-Based Questions

These questions test real-world experience. Prepare by mapping them to your own projects and situations.

<details>
<summary>33. Tell me about a time you migrated an application between cloud services or redesigned a cloud architecture — what drove the decision, what tradeoffs did you evaluate, what went wrong during the migration, and what would you do differently?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>34. Describe a time you debugged a significant cloud infrastructure issue in production — what were the symptoms, how did you narrow down whether it was a networking, compute, or configuration problem, and what was the root cause?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>35. Tell me about a time you significantly reduced cloud costs — what analysis did you do to identify waste, what changes did you implement (right-sizing, reserved capacity, architecture changes), and what was the measurable impact?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>36. Describe a time you designed a cloud architecture for high availability or disaster recovery — what availability target did you set, what architecture decisions did you make to meet it, and how did you test that failover actually worked?</summary>

<!-- Answer framework will be added later -->

</details>
