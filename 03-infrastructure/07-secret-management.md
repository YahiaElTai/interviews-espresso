# Secret Management

> **34 questions** — 11 theory, 23 practical

- Why secrets need dedicated management — risks of hardcoded values, env files, and config files
- Secret lifecycle: creation, storage, distribution, rotation, revocation
- Categories of solutions: cloud-native managers (AWS/GCP), HashiCorp Vault, K8s Secrets, Sealed Secrets — tradeoffs between simplicity, features, and operational overhead
- HashiCorp Vault: architecture (seal/unseal, storage backends, auth methods, secret engines), dynamic secrets (lease-and-revoke vs static credentials)
- Kubernetes Secrets limitations: base64 encoding, etcd storage, RBAC gaps
- Encryption at rest: etcd encryption, KMS envelope encryption, cloud-managed encryption defaults
- External Secrets Operator and Sealed Secrets: bridging K8s with external stores, GitOps-safe secret storage
- Workload Identity pattern: eliminating static credentials by binding K8s ServiceAccounts to cloud IAM (GCP Workload Identity, AWS IRSA) — why this is the default for cloud-native apps
- GCP Secret Manager: creation, IAM access, Workload Identity integration, rotation patterns
- AWS Secrets Manager: automatic rotation with Lambda, IAM policies, ECS/EKS access
- Secret delivery and access control: env vars vs volume mounts vs sidecar/init containers, per-service least-privilege scoping
- Secret versioning: pinning to specific versions vs latest, rollback strategy when rotation breaks consumers
- Zero-downtime secret rotation: dual-credential strategy, database vs API key rotation
- Audit logging and leak detection: access pattern monitoring, log redaction, alert triggers for anomalous secret reads, secret inventory and ownership tracking
- CI/CD secret injection: secure flow from secret manager through pipeline to deployment
- Secret scanning and prevention: gitleaks, detect-secrets, GitHub secret scanning — pre-commit hooks, CI integration, and false positive management
- Incident response: accidental Git commits, containment, history cleanup, pre-commit hooks
- Troubleshooting secret delivery: ESO sync status, IAM permission errors, rotation propagation lag, pod restart requirements

---

## Foundational

<details>
<summary>1. Why do secrets need dedicated management infrastructure instead of being stored in environment files, config files, or hardcoded in source code — what specific risks does each of these common shortcuts introduce, and what properties must a proper secret management solution have to address them?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>2. What is the secret lifecycle — walk through each phase (creation, storage, distribution, rotation, revocation) and explain why skipping any single phase creates a security gap, how the phases depend on each other, and what goes wrong in organizations that treat secret management as "just storage"?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>3. What are the major categories of secret management solutions (cloud-native managers, HashiCorp Vault, Kubernetes-native secrets, sealed secrets) — what problem does each category solve best, what are the tradeoffs between them, and how do you decide which combination fits a given team's infrastructure and maturity level?</summary>

<!-- Answer will be added later -->

</details>

## HashiCorp Vault — Concepts

<details>
<summary>4. How does HashiCorp Vault's seal/unseal mechanism work — why does Vault start sealed, what does unsealing actually do cryptographically, what are the risks of manual unsealing with Shamir's keys, and why do most production deployments use auto-unseal with a cloud KMS instead?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>5. How do Vault's auth methods, secret engines, and storage backends work together to form a layered architecture — why does Vault separate these concerns instead of being a simple encrypted key-value store, and how does this separation affect security and extensibility?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>6. Why do dynamic secrets exist in Vault and what problem do they solve that static credentials cannot — explain the lease-and-revoke model, how Vault generates short-lived database credentials on demand, what happens when a lease expires, and when dynamic secrets are worth the operational complexity vs when static secrets with rotation are good enough?</summary>

<!-- Answer will be added later -->

</details>

## Kubernetes & Secret Integration — Concepts

<details>
<summary>7. What are the real limitations of Kubernetes Secrets — why is base64 encoding not encryption, what are the risks of secrets stored unencrypted in etcd, how can RBAC gaps expose secrets to unauthorized pods or users, and what does it take to actually secure K8s secrets (etcd encryption, KMS envelope encryption, cloud-managed encryption defaults)?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>8. Why do tools like External Secrets Operator and Sealed Secrets exist when Kubernetes already has a Secrets resource — what gap does each one fill, how does ESO bridge the divide between external secret stores and K8s, how do Sealed Secrets make secret storage GitOps-safe, and when would you choose one over the other?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>9. Why is the Workload Identity pattern (GCP Workload Identity, AWS IRSA) the default approach for cloud-native apps accessing cloud services — what security problems does it solve that service account key files and static credentials create, how does binding K8s ServiceAccounts to cloud IAM work at a high level, and when might you still need static credentials despite Workload Identity being available?</summary>

<!-- Answer will be added later -->

</details>

## Delivery & Access Control — Concepts

<details>
<summary>10. What are the tradeoffs between the different secret delivery methods in Kubernetes — environment variables vs volume mounts vs sidecar/init containers — how does each one work, what are the security implications of each (visibility in process listings, persistence on disk, rotation behavior), and how do you implement per-service least-privilege scoping so each service can only access the secrets it needs?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>11. Why should you pin secret references to specific versions instead of always using 'latest' — what risks does 'latest' introduce during rotation, how do you implement a rollback strategy when a rotated secret breaks consumers, and how do cloud secret managers (GCP Secret Manager, AWS Secrets Manager) handle versioning differently?</summary>

<!-- Answer will be added later -->

</details>

## Practical — Secret Manager Configuration

<details>
<summary>12. Configure Vault's Kubernetes auth method so pods can authenticate using their ServiceAccount tokens, then set up a KV v2 secret engine with a policy that restricts a specific service to reading only its own secrets — show the Vault CLI commands or API calls, explain how the auth flow works end to end (pod → K8s token review → Vault token), and what breaks if the Vault-K8s trust is misconfigured?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>13. Set up a secret in GCP Secret Manager and grant access using IAM with Workload Identity so a GKE pod can read it without a service account key file — show the gcloud commands for creating the secret, binding IAM roles, and configuring Workload Identity on the K8s ServiceAccount. What are the common IAM misconfigurations that cause access failures?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>14. Configure a rotation schedule for a GCP Secret Manager secret — show how to set up automatic rotation using a Cloud Function, explain how consuming applications detect and pick up the new version, and what happens if rotation succeeds but consumers are still pinned to the old version.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>15. Configure AWS Secrets Manager with automatic rotation using a Lambda function for a database credential — show the AWS CLI commands or Terraform for creating the secret, attaching a rotation Lambda, and setting the rotation schedule. Then show how an ECS task or EKS pod accesses the secret via IAM roles (task role / IRSA), and explain what happens during the rotation window if the application caches the old credential.</summary>

<!-- Answer will be added later -->

</details>

## Practical — Kubernetes Secret Configuration

<details>
<summary>16. Create a Kubernetes Secret and consume it two ways — as environment variables and as a mounted volume in a pod — show the YAML for both approaches, explain when you'd choose one over the other (rotation behavior, multi-line values, secret size), and demonstrate what happens when the underlying secret is updated (which method picks up changes automatically and which doesn't).</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>17. Configure etcd encryption for Kubernetes Secrets using an EncryptionConfiguration, then show how to upgrade to KMS envelope encryption with a cloud KMS provider — show the configuration files, explain the difference between identity (no encryption), aescbc, and KMS providers, how envelope encryption adds a layer of protection, and what the migration process looks like for a cluster that currently stores secrets unencrypted.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>18. Set up External Secrets Operator to sync secrets from a cloud secret manager (GCP Secret Manager or AWS Secrets Manager) into Kubernetes Secrets — show the YAML for the SecretStore, ExternalSecret, and the resulting K8s Secret, explain the sync interval and refresh behavior, and what happens when the external secret is rotated — does the K8s Secret update automatically and do pods pick up the change?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>19. Set up Sealed Secrets (Bitnami) for a GitOps workflow — show how to install the controller, encrypt a secret using kubeseal so it's safe to commit to Git, and how the controller decrypts it in-cluster. Explain the certificate management (what happens when the sealing key rotates), scope modes (strict, namespace-wide, cluster-wide), and when Sealed Secrets falls short compared to ESO.</summary>

<!-- Answer will be added later -->

</details>

## Practical — Rotation, CI/CD & Incident Response

<details>
<summary>20. Walk through a zero-downtime database credential rotation using the dual-credential strategy — explain why you need two valid credentials simultaneously, show the step-by-step process (create new credential → deploy to secret manager → wait for propagation → revoke old credential), what coordination is needed between the secret manager and the application, and what breaks if you rotate-then-revoke without the overlap window.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>21. You need to rotate a third-party API key where the provider only allows one active key at a time — design the rotation procedure step by step, show how you'd implement a brief maintenance window or proxy-based switchover, explain what coordination is needed with the consuming services, and what breaks if you revoke the old key before all consumers have the new one.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>22. Design the secret injection flow for a CI/CD pipeline — show how secrets move securely from a secret manager through the CI/CD system (e.g., CircleCI, GitHub Actions) to the deployment target without being exposed in logs, environment dumps, or build artifacts. What are the common mistakes (echoing secrets in debug output, storing in intermediate files, overly broad pipeline permissions), and how do you scope pipeline access so each job only sees the secrets it needs?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>23. How do you set up audit logging and leak prevention for secrets — show how to detect secret exfiltration attempts (access patterns, anomalous reads), implement redaction in application logs and error messages so secrets never appear in log aggregators, configure alert triggers for suspicious secret access, and maintain a secret inventory so you know what secrets exist, who owns them, and when they expire? What tools and patterns make this practical at scale?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>24. A developer accidentally commits a database password to a Git repository — walk through the complete incident response: immediate containment (rotate the credential NOW, not after cleanup), removing the secret from Git history using git filter-branch or BFG Repo-Cleaner, force-pushing the cleaned history, and then setting up pre-commit hooks (e.g., detect-secrets, gitleaks) to prevent future occurrences. Why is removing from history insufficient without rotation?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>25. How do you integrate secret scanning tools (gitleaks, detect-secrets, GitHub secret scanning) into your CI/CD pipeline — where in the pipeline should scanning run, how do you handle false positives so they don't block legitimate deployments, and what's the difference between pre-commit scanning (developer-side) vs CI scanning (server-side) in terms of coverage and enforcement?</summary>

<!-- Answer will be added later -->

</details>

## Practical — Debugging & Troubleshooting

<details>
<summary>26. External Secrets Operator is deployed but the K8s Secret it should create is missing or stale — walk through the debugging process: checking the ExternalSecret status and sync conditions, verifying the SecretStore connectivity, diagnosing IAM permission errors in the operator logs, and fixing common misconfigurations (wrong secret path, incorrect property extraction, SecretStore auth failures). Show the exact kubectl commands at each step.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>27. A pod can't read a Kubernetes Secret it should have access to — walk through diagnosing whether the issue is RBAC (ServiceAccount lacks permission), secret not existing in the namespace, a typo in the secret name reference, or the secret key not matching what the pod spec expects. Show the kubectl commands to check each, and explain how RBAC for secrets differs between reading via the API and mounting as a volume.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>28. You rotated a secret in your secret manager but the application is still using the old value — diagnose the propagation chain: did the secret manager update succeed, did ESO pick up the change (check sync interval and status), did the K8s Secret update, and is the pod actually seeing the new value? Explain why environment variable-based secrets require a pod restart, how volume-mounted secrets eventually update via kubelet sync, and what application-level patterns (file watchers, periodic reload) ensure rotation actually propagates.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>29. Vault is returning 503 errors and pods can't authenticate — walk through diagnosing whether Vault is sealed (and why auto-unseal might have failed), whether the Kubernetes auth method's token reviewer binding is broken, whether the Vault policy denies the requested path, or whether network connectivity between the pod and Vault is the issue. Show the Vault CLI and kubectl commands for each diagnostic step.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>30. A new microservice is deployed but its secrets are completely missing at runtime — the pod starts but the application crashes because it can't find its credentials. Walk through a systematic end-to-end diagnosis: checking the secret manager for the secret's existence, verifying ESO or Sealed Secrets created the K8s Secret, confirming the pod spec references the correct secret name and keys, checking the namespace, and inspecting pod events. What order do you check these layers and why?</summary>

<!-- Answer will be added later -->

</details>

---

## Experience-Based Questions

These questions test real-world experience. Prepare by mapping them to your own projects and situations.

<details>
<summary>31. Tell me about a time you set up or significantly improved secret management infrastructure for a team or organization — what was the existing state, what solution did you implement, what tradeoffs did you weigh, and what impact did it have on security and developer experience?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>32. Describe a time you dealt with a secret leak or exposure — how did you discover it, what was your immediate response, how did you contain the blast radius, and what preventive measures did you put in place afterward?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>33. Tell me about a time you implemented zero-downtime secret rotation — what type of credential was it, what strategy did you use, what challenges did you encounter during the rotation window, and how did you verify the rotation succeeded without breaking any consumers?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>34. Describe a time you debugged a secret delivery failure in production — what were the symptoms, how did you trace the issue through the layers (secret manager → sync mechanism → K8s Secret → pod), and what was the root cause?</summary>

<!-- Answer framework will be added later -->

</details>
