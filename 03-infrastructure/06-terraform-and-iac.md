# Terraform & Infrastructure as Code

> **34 questions** — 10 theory, 24 practical

- Why Infrastructure as Code exists — problems it solves over manual provisioning (ClickOps)
- Terraform's declarative model: init, plan, apply — and how it differs from imperative approaches
- State: why it exists, drift detection (manual changes, external automation), and consequences of out-of-sync state
- Remote backends (GCS/S3): configuration, state locking, race conditions, and force-unlock
- State splitting: monolithic vs per-service/per-environment state, sharing data between states
- Provider versioning, pinning, and handling breaking changes across upgrades
- HCL: count vs for_each, dynamic blocks, locals, variables with validation, outputs, data sources
- Lifecycle meta-arguments: create_before_destroy, prevent_destroy, ignore_changes, replace_triggered_by
- Dependency management: implicit graph, depends_on for hidden dependencies, -target for partial operations and its risks
- Provisioners (local-exec, remote-exec): why they're a last resort, alternatives (cloud-init, Packer, user_data)
- Modules: reusable design, input/output contracts, versioning, when NOT to create a module
- Repo structure for multiple environments: workspaces vs directories vs Terragrunt
- Secrets in Terraform: state file risks, sensitive flag, Vault provider, SOPS
- Terraform import: CLI command, import blocks (1.5+), adopting Terraform for existing (brownfield) infrastructure, pitfalls with complex resources
- State operations: state mv, state rm, state pull/push, moved/removed blocks — refactoring and recovery scenarios
- CI/CD workflows: plan on PR, apply on merge, guardrails, provider credentials (workload identity federation)
- Policy as code: Sentinel (Terraform Cloud), OPA/Conftest for open-source, pre-plan and post-plan validation
- Debugging: unexpected destroy/recreate, partial apply failures, tainted resources
- Comparison with Pulumi, CloudFormation, and CDK: language model, state management, cloud lock-in, ecosystem maturity

---

## Foundational

<details>
<summary>1. Why does Infrastructure as Code exist — what specific problems does it solve over manual provisioning (ClickOps), why is the declarative approach (describing desired state) fundamentally different from imperative scripting (step-by-step commands), and why has the declarative model won for infrastructure management?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>2. Walk through Terraform's core workflow — what happens during each phase of init, plan, and apply, why are they separated into distinct steps instead of a single "deploy" command, and what role does the dependency graph play in determining the order of operations?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>3. Why does Terraform need a state file at all — what information does it store that can't be derived from the HCL configuration or the cloud provider's API, how does Terraform use state for drift detection when someone changes infrastructure manually or via external automation, and what are the consequences when state gets out of sync with reality?</summary>

<!-- Answer will be added later -->

</details>

## Conceptual Depth

<details>
<summary>4. Why must Terraform state be stored in a remote backend (GCS, S3) for any team larger than one person — how does state locking prevent race conditions when two engineers run apply simultaneously, what happens if a lock gets stuck (e.g., apply crashed midway), and what are the risks of using force-unlock?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>5. What are the tradeoffs between monolithic state (everything in one state file), per-service state, and per-environment state — what problems does a monolithic state cause as infrastructure grows, how do you share data between split states (e.g., a networking state needs to expose VPC IDs to a compute state), and what's the blast radius argument for splitting?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>6. Why would you create a Terraform module and when is it the wrong choice -- what makes a module genuinely reusable vs just adding abstraction for abstraction's sake, what are the signs that a module is doing too much, and when should you just use flat resources instead?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>7. What are the tradeoffs between Terraform workspaces, directory-based structure, and Terragrunt for managing multiple environments (dev/staging/prod) — why do most teams avoid workspaces for environment separation despite it being a built-in feature, what does a directory-based approach look like, and when does Terragrunt's overhead become worth it?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>8. Why are secrets in Terraform particularly dangerous — how does the state file expose sensitive values even when you use the `sensitive` flag, what are the limitations of `sensitive` (plan output vs state storage), and how do approaches like the Vault provider and SOPS address this problem differently?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>9. How does Terraform compare to Pulumi, AWS CloudFormation, and AWS CDK — what are the fundamental architectural differences (HCL vs general-purpose languages, state management, multi-cloud support), what are the real tradeoffs in team adoption and ecosystem maturity, and when would you choose each one over Terraform?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>10. Why are Terraform provisioners (local-exec, remote-exec) considered a last resort -- what problems do they introduce (non-idempotent operations, error handling, dependency on network access), what are the recommended alternatives (cloud-init, Packer images, user_data scripts), and when is a provisioner genuinely the only option?</summary>

<!-- Answer will be added later -->

</details>

## Practical — HCL & Configuration

<details>
<summary>11. Show how to pin provider versions in Terraform using the `required_providers` block with version constraints — explain the difference between `~>`, `>=`, and exact pinning, what the `.terraform.lock.hcl` file does, and walk through the process of upgrading a provider when a new major version introduces breaking changes</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>12. Show examples of `count` vs `for_each` in HCL and explain when each one breaks — demonstrate a scenario where using `count` with a list causes resources to be unnecessarily destroyed and recreated when an item is removed from the middle of the list, and show how `for_each` with a map or set avoids this problem</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>13. Show how to use HCL variables with custom validation blocks and locals -- demonstrate a variable with a validation rule that gives a clear error message on bad input, show a local that computes a derived value used in multiple resources, and explain when to use a variable vs a local vs hardcoding a value</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>14. Show how Terraform data sources and outputs work together to connect separate states and modules -- demonstrate a data source that looks up an existing resource (e.g., a VPC or service account not managed by this config), outputs that expose values for other states or modules to consume, and explain when a data source is the right choice vs managing the resource directly</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>15. Show HCL examples for create_before_destroy and prevent_destroy lifecycle meta-arguments -- demonstrate create_before_destroy for a zero-downtime resource replacement (e.g., a TLS certificate or load balancer) and prevent_destroy for protecting a production database, explain what specific problem each solves, and what breaks without them</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>16. Show HCL examples for ignore_changes and replace_triggered_by lifecycle meta-arguments -- demonstrate ignore_changes for a resource whose tags or annotations are managed externally (e.g., by a Kubernetes operator or another team's automation), and replace_triggered_by for forcing a replacement when a dependency changes. Explain when ignore_changes masks a real problem vs solves a legitimate one</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>17. Write a reusable Terraform module for a common infrastructure pattern (e.g., a GCS bucket with IAM bindings or a VPC with subnets) -- show the directory structure, variables.tf with sensible defaults and validation, outputs.tf exposing necessary values, and how the calling root module consumes it. Explain what makes this module easy vs hard to reuse across teams and how you'd version it using git tags</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>18. Show how to use dynamic blocks in HCL to generate repeated nested blocks programmatically -- demonstrate a scenario where a security group or IAM policy needs a variable number of rules based on input, explain how dynamic blocks reduce duplication compared to writing each block manually, and when using a dynamic block hurts readability more than it helps</summary>

<!-- Answer will be added later -->

</details>

## Practical — State Operations & Import

<details>
<summary>19. You have existing cloud infrastructure that was created manually and you need to bring it under Terraform management — show how to do this using both the `terraform import` CLI command and the newer `import` blocks (Terraform 1.5+), explain what pitfalls you'll hit with complex resources (e.g., the imported state doesn't match your HCL and plan shows changes), and walk through the process of writing the HCL to match the imported resource</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>20. You're refactoring Terraform code — a resource needs to move to a different module or be renamed. Show how to use `terraform state mv` to move it without destroying and recreating the real infrastructure, how `terraform state rm` removes a resource from state without deleting it, and when you'd use `terraform state pull` and `state push` for manual state repairs. What are the risks of each operation?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>21. Show how `moved` blocks and `removed` blocks work as declarative alternatives to `terraform state mv` and `terraform state rm` — write examples of each, explain why they're safer than imperative state commands (version-controlled, team-visible, idempotent), and what limitations they have compared to the CLI commands</summary>

<!-- Answer will be added later -->

</details>

## Practical — CI/CD & Production Workflows

<details>
<summary>22. Design a CI/CD workflow for Terraform where `plan` runs on pull requests and `apply` runs on merge to main — show the pipeline configuration (e.g., CircleCI or GitHub Actions YAML), explain what guardrails you'd add (plan output as PR comment, manual approval for production, policy checks), and how you prevent two merges from racing to apply conflicting changes</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>23. How do you securely provide cloud provider credentials to Terraform in a CI/CD pipeline — compare workload identity federation (OIDC-based, no static keys) vs static service account keys, show how workload identity federation is configured for GCP or AWS, and explain why static credentials are a security risk even when stored as CI/CD secrets</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>24. Configure a remote backend for team collaboration — show the HCL for a GCS or S3 backend with state locking enabled, explain the initialization process, what happens when you switch from local to remote state, and how you handle the chicken-and-egg problem of the backend bucket itself (do you Terraform the bucket that holds your Terraform state?)</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>25. How do you enforce policy as code for Terraform -- compare Sentinel (Terraform Cloud/Enterprise) with OPA and Conftest for open-source workflows, show how a policy check integrates into the plan/apply pipeline (pre-plan vs post-plan validation), and give an example of a policy rule you'd enforce (e.g., "all S3 buckets must have encryption enabled" or "no public IPs on compute instances")</summary>

<!-- Answer will be added later -->

</details>

## Practical — Debugging & Troubleshooting

<details>
<summary>26. Terraform plan shows it wants to destroy and recreate a resource you didn't change — walk through the systematic debugging process: how to read the plan output to identify what's forcing the replacement (immutable field change, `replace_triggered_by`, provider upgrade), how to use `terraform show` and `terraform state show` to compare current state with the plan, and what your options are to avoid the destructive operation</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>27. A terraform apply fails halfway through -- some resources were created, others were not, and one resource is marked as tainted. Walk through the recovery process: what does Terraform's state look like after a partial apply, how do you identify which resources succeeded vs failed vs are tainted, what happens when you run apply again (how does Terraform treat tainted resources differently from healthy ones), how do you untaint a resource you know is actually fine, and when would you intentionally taint a resource to force its recreation?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>28. Someone modified infrastructure directly in the cloud console (ClickOps) and now Terraform state is out of sync — walk through how you detect this drift with `terraform plan`, how you decide whether to update the HCL to match reality or revert the manual change by applying the original config, and what patterns prevent this from happening repeatedly in a team</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>29. Your team's Terraform state file is corrupted or accidentally deleted — walk through the recovery options: restoring from the remote backend's versioning (GCS/S3 object versioning), using `terraform state pull` and `terraform state push` for repair, re-importing resources as a last resort, and what preventive measures (backend versioning, state snapshots) you should have in place before disaster strikes</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>30. A `terraform plan` is failing with confusing dependency or cycle errors — explain how Terraform builds its dependency graph, the difference between implicit dependencies (resource references) and explicit `depends_on`, when `depends_on` is necessary vs when it hides a design problem, how you debug circular dependency errors by inspecting the graph with `terraform graph`, and when engineers use terraform apply -target to apply a subset of resources, what risks does this introduce (state drift, skipped dependencies) and why is it considered a last resort?</summary>

<!-- Answer will be added later -->

</details>

---

## Experience-Based Questions

These questions test real-world experience. Prepare by mapping them to your own projects and situations.

<details>
<summary>31. Tell me about a time you migrated existing infrastructure (manually provisioned or managed with another tool) to Terraform — what was the scope, what challenges did you face with import and state, and how did you validate that the migration was correct without downtime?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>32. Describe a time you had to recover from a Terraform state issue in production — what happened (state corruption, accidental deletion, state lock stuck), how did you diagnose and fix it, and what safeguards did you put in place afterward?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>33. Tell me about a time you designed the Terraform repository structure, module strategy, or CI/CD workflow for a team — what decisions did you make about state splitting, environment management, and code reuse, and what tradeoffs did you encounter as the team and infrastructure grew?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>34. Describe a time you debugged a tricky Terraform issue — unexpected resource destruction, plan/apply mismatch, provider bugs, or mysterious diffs that wouldn't go away. How did you systematically diagnose the root cause and what was the fix?</summary>

<!-- Answer framework will be added later -->

</details>
