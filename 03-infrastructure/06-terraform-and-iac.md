# Terraform & Infrastructure as Code

> **20 questions** — 7 theory, 11 practical, 2 experience

- Why Infrastructure as Code exists — problems it solves over manual provisioning (ClickOps)
- Terraform's declarative model: init, plan, apply — and how it differs from imperative approaches
- State: why it exists, drift detection (manual changes, external automation), and consequences of out-of-sync state
- Remote backends (GCS/S3): configuration, state locking, race conditions, and force-unlock
- State splitting: monolithic vs per-service/per-environment state, sharing data between states
- HCL: count vs for_each, dynamic blocks, locals, variables with validation, outputs, data sources
- Lifecycle meta-arguments: create_before_destroy, prevent_destroy
- Dependency management: implicit graph, depends_on for hidden dependencies, -target for partial operations and its risks
- Modules: reusable design, input/output contracts, versioning, when NOT to create a module
- Repo structure for multiple environments: workspaces vs directories vs Terragrunt
- Secrets in Terraform: state file risks, sensitive flag, Vault provider, SOPS
- CI/CD workflows: plan on PR, apply on merge, guardrails, provider credentials (workload identity federation)

---

## Foundational

<details>
<summary>1. Why does Infrastructure as Code exist — what specific problems does it solve over manual provisioning (ClickOps), why is the declarative approach (describing desired state) fundamentally different from imperative scripting (step-by-step commands), and why has the declarative model won for infrastructure management?</summary>

**Problems IaC solves over ClickOps:**

- **Reproducibility**: Manual provisioning is impossible to repeat exactly. IaC lets you spin up identical environments (dev/staging/prod) from the same code.
- **Auditability**: Infrastructure changes go through pull requests with review, approval, and git history. ClickOps leaves no trace of who changed what or why.
- **Speed and scale**: Creating 50 identical resources manually is error-prone and slow. IaC handles it in one apply.
- **Disaster recovery**: If an environment is destroyed, IaC rebuilds it from code. Manual setups require someone to remember every setting.
- **Drift prevention**: Without IaC, environments diverge over time as different people make ad-hoc changes.

**Declarative vs imperative:**

Imperative scripting (bash scripts, SDK calls) says "do these steps in this order" — create VPC, then create subnet, then create firewall rule. You manage ordering, error handling, idempotency, and the "what if step 3 fails but steps 1-2 succeeded?" problem yourself.

Declarative IaC says "here's what I want to exist" — Terraform figures out what currently exists, computes the diff, determines the order via dependency graph, and executes only the needed changes.

**Why declarative won:**

The key insight is that infrastructure management is mostly about convergence — getting from current state to desired state. Declarative tools handle this naturally: run it once, run it again, nothing changes (idempotent). Imperative scripts need explicit checks at every step to be idempotent. Declarative also handles deletions automatically — remove a resource from config and it gets destroyed. In imperative scripts, you'd need separate teardown logic.


</details>

<details>
<summary>2. Walk through Terraform's core workflow — what happens during each phase of init, plan, and apply, why are they separated into distinct steps instead of a single "deploy" command, and what role does the dependency graph play in determining the order of operations?</summary>

**`terraform init`:**
- Downloads provider plugins (AWS, GCP, etc.) and modules referenced in the config
- Configures the backend (local or remote) for state storage
- Creates `.terraform/` directory with provider binaries and module cache
- Must be re-run when you add a new provider, change backend config, or add a module

**`terraform plan`:**
- Reads current state (from backend) and refreshes it against real infrastructure (API calls to cloud provider)
- Parses all `.tf` files and builds the dependency graph
- Computes the diff: what needs to be created, updated, or destroyed to reach desired state
- Outputs a human-readable execution plan showing every change with `+`, `~`, `-` markers
- Can be saved to a plan file (`-out=plan.tfplan`) for exact reproducibility

**`terraform apply`:**
- Executes the changes from the plan (or computes a new plan if no plan file is provided)
- Walks the dependency graph, creating/updating/destroying resources in the correct order
- Parallelizes independent operations (resources with no dependency relationship)
- Updates state file after each successful resource operation
- If a resource fails, already-completed changes persist — Terraform doesn't roll back

**Why separate steps:**

Separation gives you a review gate. `plan` is safe — it's read-only and changes nothing. This lets you put `plan` in CI on every PR so reviewers see exactly what will change before approving. A single "deploy" command would be too dangerous for production infrastructure — you'd be approving changes blind.

**Dependency graph:**

Terraform builds a directed acyclic graph (DAG) from resource references. If `google_compute_instance` references `google_compute_network.main.id`, Terraform knows the network must be created first. Resources with no dependency on each other are created in parallel. This is why declarative works — you don't specify order, Terraform infers it.


</details>

<details>
<summary>3. Why does Terraform need a state file at all — what information does it store that can't be derived from the HCL configuration or the cloud provider's API, how does Terraform use state for drift detection when someone changes infrastructure manually or via external automation, and what are the consequences when state gets out of sync with reality?</summary>

**Why state exists — what it stores that HCL and APIs can't provide:**

- **Mapping between HCL names and real resource IDs**: Your config says `google_compute_instance.web`, but GCP knows it as `projects/my-proj/zones/us-east1-b/instances/abc123`. State holds this mapping. Without it, Terraform can't know which real resource corresponds to which config block.
- **Metadata and dependency ordering**: State tracks which resources depend on which, including implicit dependencies from resource references. This is needed for correct destruction order.
- **Performance cache**: For large infrastructure (hundreds of resources), querying every cloud API on every plan is slow. State acts as a cache — Terraform can skip refresh for resources it already knows about (though default behavior is to refresh).
- **Computed/derived values**: Some resource attributes are only known after creation (IP addresses, generated IDs). State stores these so outputs and other resources can reference them.

**Drift detection:**

When you run `terraform plan`, Terraform:
1. Reads the state file for the last known configuration
2. Calls the cloud provider API to get the actual current state of each resource (the "refresh" step)
3. Compares actual state vs desired state (HCL config)

If someone changed a firewall rule in the console, the refresh shows the real value differs from what's in state AND what's in the HCL. Terraform reports this as drift and proposes changes to bring reality back in line with the config.

**Consequences of out-of-sync state:**

- **Terraform tries to recreate existing resources** if state says they don't exist but they do — this can fail or, worse, create duplicates
- **Terraform ignores resources** it doesn't know about — if someone manually created a resource, Terraform won't manage or destroy it
- **Incorrect dependency ordering** during destroy — if state has stale dependency info, Terraform might try to destroy a VPC before its subnets
- **Orphaned resources** — if state loses track of a resource, it'll never be cleaned up by Terraform, leading to cost leaks


</details>

## Conceptual Depth

<details>
<summary>4. Why must Terraform state be stored in a remote backend (GCS, S3) for any team larger than one person — how does state locking prevent race conditions when two engineers run apply simultaneously, what happens if a lock gets stuck (e.g., apply crashed midway), and what are the risks of using force-unlock?</summary>

**Why remote state is required for teams:**

With local state, the state file lives on one engineer's machine. Two people can't see each other's changes — engineer A creates a VPC, engineer B's state doesn't know it exists and tries to create it again. There's no single source of truth. Remote backends (GCS, S3, Terraform Cloud) put state in a shared location everyone reads from and writes to.

Remote state also enables CI/CD — your pipeline needs access to state, and it can't read a file from someone's laptop.

**How state locking works:**

When `terraform apply` starts, it acquires a lock on the state file (GCS supports locking natively without additional configuration; S3 traditionally uses a DynamoDB table, though as of Terraform v1.10+ S3 also supports native locking via `use_lockfile = true`, making DynamoDB the legacy approach). If another operation is already running, Terraform refuses to proceed:

```
Error: Error locking state: Error acquiring the state lock
Lock Info:
  ID:        abc-123
  Path:      gs://my-bucket/terraform.tfstate
  Operation: OperationTypeApply
  Who:       engineer-a@machine
  Created:   2024-01-15 10:30:00 UTC
```

This prevents two simultaneous `apply` operations from writing conflicting changes to state. Without locking, two applies could both read the same state, each make different changes, and the last one to write wins — silently overwriting the other's state entries, leaving orphaned resources.

**Stuck locks:**

If an `apply` crashes (killed process, network failure, CI timeout), the lock may not be released. Terraform shows the lock info (who, when, operation). The fix:

1. **First**: Verify no operation is actually running (check CI pipelines, ask the team)
2. **Then**: Run `terraform force-unlock <LOCK_ID>`

**Risks of force-unlock:**

- If you force-unlock while an operation IS still running (e.g., slow apply on someone else's machine), both operations write to state simultaneously — corrupting it
- There's no confirmation that the original operation actually stopped
- Force-unlock should be a rare, manual operation — never automated in CI. If locks are getting stuck regularly, fix the root cause (timeouts, crash-prone providers)


</details>

<details>
<summary>5. What are the tradeoffs between monolithic state (everything in one state file), per-service state, and per-environment state — what problems does a monolithic state cause as infrastructure grows, how do you share data between split states (e.g., a networking state needs to expose VPC IDs to a compute state), and what's the blast radius argument for splitting?</summary>

**Monolithic state problems:**

- **Blast radius**: Every `apply` touches every resource. A typo in a variable could destroy your database alongside an unrelated DNS change. The blast radius is "everything."
- **Slow operations**: `plan` and `apply` refresh every resource's state via API calls. At hundreds of resources, this takes minutes.
- **Lock contention**: Only one person can run `apply` at a time. With a monolithic state, networking changes block application deployments.
- **Risk accumulation**: The state file itself becomes a critical single point of failure. Corruption affects all infrastructure.

**Per-service state:**

Split by logical service/component: `networking/`, `database/`, `compute/`, `dns/`. Each has its own state file, backend config, and lock.

- **Pros**: Small blast radius, fast plans, independent deploys, team ownership boundaries align with state boundaries
- **Cons**: Need explicit data sharing between states, more backend configuration to manage

**Per-environment state:**

Split by environment: `dev/`, `staging/`, `prod/`. Each environment is a separate state.

- **Pros**: Prod changes can't affect dev, independent apply cadence per environment
- **Cons**: Still monolithic within each environment

**In practice, combine both**: `prod/networking/`, `prod/compute/`, `dev/networking/`, etc.

**Sharing data between split states:**

Use `terraform_remote_state` data source or (better) output values read via data sources:

```hcl
# In networking state — outputs.tf
output "vpc_id" {
  value = google_compute_network.main.id
}

# In compute state — data.tf
data "terraform_remote_state" "networking" {
  backend = "gcs"
  config = {
    bucket = "my-tf-state"
    prefix = "prod/networking"
  }
}

resource "google_compute_instance" "app" {
  network_interface {
    network = data.terraform_remote_state.networking.outputs.vpc_id
  }
}
```

**The blast radius argument**: If a bad `apply` can only affect one service's compute instances and not the database or networking layer, the worst case is a service restart, not a full infrastructure outage. This is the single strongest reason to split state.


</details>

<details>
<summary>6. Why would you create a Terraform module and when is it the wrong choice -- what makes a module genuinely reusable vs just adding abstraction for abstraction's sake, what are the signs that a module is doing too much, and when should you just use flat resources instead?</summary>

**When to create a module:**

- **Genuine reuse**: The same pattern (e.g., GCS bucket with standard IAM, logging, and lifecycle rules) is deployed 5+ times across services or environments. The module encodes organizational standards.
- **Enforcing conventions**: A module can enforce that every database gets backups enabled, every bucket gets versioning, every service account follows a naming convention. This is governance through code.
- **Reducing cognitive load**: A well-designed module hides provider-specific complexity behind a clean interface (`environment`, `service_name`, `team`) so consumers don't need to know the 15 arguments on a raw resource.

**When it's the wrong choice:**

- **One-off infrastructure**: If you're only creating one VPC, wrapping it in a module adds indirection with no benefit. Just use flat resources.
- **Thin wrappers**: A module that takes the same arguments as the underlying resource and just passes them through adds no value — it's abstraction for abstraction's sake.
- **Rapidly changing requirements**: Modules add coupling. Every consumer depends on the module's interface. If the underlying pattern is still being figured out, a module forces premature standardization.
- **Team of one**: If nobody else will ever use it, a module's input/output contract is overhead for no audience.

**Signs a module is doing too much:**

- It has 20+ input variables with most being pass-through to underlying resources
- It manages unrelated resources (networking AND compute AND DNS in one module)
- You constantly need to add conditional logic (`count` based on feature flags) because different consumers need different subsets
- Changes to the module require coordinated updates across many consumers

**Rule of thumb**: Start with flat resources. Extract a module when you've copy-pasted the same pattern three times and the pattern has stabilized. Premature modularization is worse than duplication.


</details>

<details>
<summary>7. What are the tradeoffs between Terraform workspaces, directory-based structure, and Terragrunt for managing multiple environments (dev/staging/prod) — why do most teams avoid workspaces for environment separation despite it being a built-in feature, what does a directory-based approach look like, and when does Terragrunt's overhead become worth it?</summary>

**Terraform workspaces:**

Workspaces let you maintain multiple state files for the same configuration. You switch with `terraform workspace select prod` and use `terraform.workspace` in your HCL to vary behavior.

**Why most teams avoid them for environments:**

- All environments share the exact same code — you can't have prod use a larger instance type without conditional logic like `var.instance_type[terraform.workspace]` everywhere. This gets messy fast.
- The state files are stored under the same backend prefix, making it easy to accidentally apply to the wrong workspace (one typo away from destroying prod).
- No visual separation — `ls` shows one directory, not separate environments. Code review can't tell which environment a change targets.
- Workspaces were designed for feature branches and testing, not long-lived environment separation.

**Directory-based approach (most common):**

```
infrastructure/
  modules/
    compute/
    networking/
  environments/
    dev/
      main.tf          # calls modules with dev-specific values
      terraform.tfvars  # instance_type = "e2-small"
      backend.tf        # bucket prefix = "dev/"
    staging/
      main.tf
      terraform.tfvars
      backend.tf
    prod/
      main.tf
      terraform.tfvars  # instance_type = "e2-standard-4"
      backend.tf
```

Each environment is a separate Terraform root module with its own state. You `cd` into the environment and run `terraform apply`. Differences between environments are explicit in the tfvars. Pros: simple, clear, auditable. Cons: duplication of `main.tf` across environments (mostly the same module calls with different variables).

**Terragrunt:**

Terragrunt wraps Terraform to reduce the duplication in directory-based structures. You define modules once, and each environment has a small `terragrunt.hcl` file specifying just the variable overrides:

```hcl
# environments/prod/compute/terragrunt.hcl
terraform {
  source = "../../../modules/compute"
}

inputs = {
  instance_type = "e2-standard-4"
  min_replicas  = 3
}
```

**When Terragrunt is worth it:**

- 5+ environments or 10+ state roots with mostly identical module calls
- You need dependency orchestration between states (Terragrunt's `dependency` blocks)
- You want DRY backend configuration (defined once, inherited by all environments)
- The team is comfortable with the extra tooling layer

**When it's not worth it**: Small teams with 2-3 environments. The directory-based approach with some copy-paste is simpler and doesn't require learning Terragrunt's own DSL on top of HCL.


</details>

<details>
<summary>8. Why are secrets in Terraform particularly dangerous — how does the state file expose sensitive values even when you use the `sensitive` flag, what are the limitations of `sensitive` (plan output vs state storage), and how do approaches like the Vault provider and SOPS address this problem differently?</summary>

**Why Terraform state is dangerous for secrets:**

The state file is plain JSON. Every attribute of every resource is stored in cleartext — including database passwords, API keys, TLS private keys, and any other sensitive value Terraform manages. If you create a `google_sql_user` with a password, that password is sitting in the state file, readable by anyone with access to the backend bucket.

**The `sensitive` flag and its limitations:**

```hcl
variable "db_password" {
  type      = string
  sensitive = true
}
```

The `sensitive` flag does one thing: it redacts the value from `plan` and `apply` output (shows `(sensitive value)` instead). This prevents secrets from leaking into CI logs and PR comments.

**What `sensitive` does NOT do:**
- It does NOT encrypt the value in the state file — it's still plain text in `terraform.tfstate`
- It does NOT prevent the value from being in the state at all
- It does NOT protect against someone reading the state with `terraform state show`

So `sensitive` is a log-redaction feature, not a security feature. The state file itself must be protected (encrypted bucket, restricted IAM access, no local copies).

**Vault provider approach:**

Instead of storing secrets in Terraform variables, read them from Vault at apply time:

```hcl
data "vault_generic_secret" "db" {
  path = "secret/prod/database"
}

resource "google_sql_user" "app" {
  password = data.vault_generic_secret.db.data["password"]
}
```

The secret still ends up in state (Terraform tracks resource attributes), but it's never in your HCL, git, or tfvars files. Vault handles rotation, access control, and audit logging. The remaining risk is state file exposure.

**SOPS approach:**

SOPS encrypts variable files at rest. You store `secrets.enc.yaml` in git (encrypted), and CI decrypts it at apply time using a KMS key. This solves the "secrets in git" problem but doesn't address the state file issue.

**Defense in depth**: Combine approaches — encrypted backend bucket, strict IAM on the bucket, `sensitive` flags on all secret variables, secrets sourced from Vault rather than tfvars, and state file access auditing. No single approach covers everything.


</details>

## Practical — HCL & Configuration

<details>
<summary>9. Show examples of `count` vs `for_each` in HCL and explain when each one breaks — demonstrate a scenario where using `count` with a list causes resources to be unnecessarily destroyed and recreated when an item is removed from the middle of the list, and show how `for_each` with a map or set avoids this problem</summary>

**`count` — index-based identification:**

```hcl
variable "bucket_names" {
  default = ["logs", "data", "backups"]
}

resource "google_storage_bucket" "buckets" {
  count = length(var.bucket_names)
  name  = "${var.bucket_names[count.index]}-bucket"
}
```

This creates:
- `google_storage_bucket.buckets[0]` -> "logs-bucket"
- `google_storage_bucket.buckets[1]` -> "data-bucket"
- `google_storage_bucket.buckets[2]` -> "backups-bucket"

**The problem — remove "data" from the middle:**

```hcl
# Changed to: ["logs", "backups"]
```

Now Terraform sees:
- `buckets[0]` = "logs-bucket" (unchanged)
- `buckets[1]` = "backups-bucket" (was "data-bucket" -> **destroy and recreate**)
- `buckets[2]` = gone -> **destroy**

Result: "data-bucket" is destroyed (correct), but "backups-bucket" is ALSO destroyed and recreated because its index shifted from 2 to 1. If that bucket had data in it, the data is gone.

**`for_each` — key-based identification:**

```hcl
variable "bucket_names" {
  default = toset(["logs", "data", "backups"])
}

resource "google_storage_bucket" "buckets" {
  for_each = var.bucket_names
  name     = "${each.key}-bucket"
}
```

This creates:
- `google_storage_bucket.buckets["logs"]` -> "logs-bucket"
- `google_storage_bucket.buckets["data"]` -> "data-bucket"
- `google_storage_bucket.buckets["backups"]` -> "backups-bucket"

**Remove "data" — only "data" is destroyed:**

```hcl
# Changed to: toset(["logs", "backups"])
```

Terraform sees:
- `buckets["logs"]` -> unchanged
- `buckets["data"]` -> **destroy** (correct)
- `buckets["backups"]` -> unchanged

No shifting, no unnecessary destruction. Each resource is identified by its key, not its position.

**When to use `count`:**

- Simple boolean toggles: `count = var.enable_monitoring ? 1 : 0`
- Creating N identical resources where order doesn't matter and items are never removed from the middle

**When to use `for_each`:**

- Any time you're iterating over a list of distinct named things (buckets, IAM bindings, DNS records). This is the default choice for almost everything. Use a map for richer data:

```hcl
variable "buckets" {
  default = {
    logs    = { location = "US", retention_days = 30 }
    backups = { location = "EU", retention_days = 90 }
  }
}

resource "google_storage_bucket" "buckets" {
  for_each = var.buckets
  name     = "${each.key}-bucket"
  location = each.value.location

  lifecycle_rule {
    action { type = "Delete" }
    condition { age = each.value.retention_days }
  }
}
```


</details>

<details>
<summary>10. Show how to use HCL variables with custom validation blocks and locals -- demonstrate a variable with a validation rule that gives a clear error message on bad input, show a local that computes a derived value used in multiple resources, and explain when to use a variable vs a local vs hardcoding a value</summary>

**Variables with validation:**

```hcl
variable "environment" {
  type        = string
  description = "Deployment environment"

  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be one of: dev, staging, prod. Got: ${var.environment}"
  }
}

variable "instance_count" {
  type        = number
  description = "Number of compute instances"
  default     = 1

  validation {
    condition     = var.instance_count >= 1 && var.instance_count <= 20
    error_message = "Instance count must be between 1 and 20."
  }
}

variable "project_id" {
  type        = string

  validation {
    condition     = can(regex("^[a-z][a-z0-9-]{4,28}[a-z0-9]$", var.project_id))
    error_message = "Project ID must be 6-30 characters, lowercase letters, digits, and hyphens."
  }
}
```

Validation runs at `plan` time before any API calls, giving fast, clear feedback. Much better than waiting for a cloud API to return a cryptic error.

**Locals for derived values:**

```hcl
locals {
  # Used by multiple resources — compute it once
  resource_prefix = "${var.project_id}-${var.environment}"

  # Common labels applied to every resource
  common_labels = {
    environment = var.environment
    managed_by  = "terraform"
    team        = var.team_name
  }

  # Conditional logic computed once, referenced many times
  is_production = var.environment == "prod"
}

resource "google_compute_instance" "app" {
  name   = "${local.resource_prefix}-app"
  labels = local.common_labels

  # Production gets more resources
  machine_type = local.is_production ? "e2-standard-4" : "e2-small"
}

resource "google_storage_bucket" "data" {
  name   = "${local.resource_prefix}-data"
  labels = local.common_labels
}
```

**When to use which:**

| | Use when |
|---|---|
| **Variable** | The value changes between environments, consumers, or deployments. It's an input to the configuration. |
| **Local** | The value is derived from variables or other locals. It avoids repeating the same expression in multiple places. Not settable by the caller. |
| **Hardcoded** | The value is a true constant that will never change (e.g., a cloud region for a single-region setup, a fixed CIDR block for a known network design). If in doubt, make it a variable — the cost is low. |


</details>

<details>
<summary>11. Show how Terraform data sources and outputs work together to connect separate states and modules -- demonstrate a data source that looks up an existing resource (e.g., a VPC or service account not managed by this config), outputs that expose values for other states or modules to consume, and explain when a data source is the right choice vs managing the resource directly</summary>

**Data source — look up an existing resource:**

Data sources read information about infrastructure that exists outside your current Terraform config. They don't create anything.

```hcl
# Look up a VPC that was created by the networking team's Terraform config
data "google_compute_network" "shared_vpc" {
  name    = "shared-vpc-prod"
  project = "networking-project"
}

# Look up a service account created by a different state
data "google_service_account" "app_sa" {
  account_id = "my-app-sa"
  project    = var.project_id
}

# Use the looked-up values in your resources
resource "google_compute_instance" "app" {
  name         = "app-server"
  machine_type = "e2-standard-2"

  network_interface {
    network = data.google_compute_network.shared_vpc.id
  }

  service_account {
    email  = data.google_service_account.app_sa.email
    scopes = ["cloud-platform"]
  }
}
```

**Outputs — expose values for consumers:**

```hcl
# outputs.tf — in the networking state
output "vpc_id" {
  value       = google_compute_network.main.id
  description = "VPC ID for other states to reference"
}

output "subnet_ids" {
  value = {
    for k, v in google_compute_subnetwork.subnets : k => v.id
  }
  description = "Map of subnet name to subnet ID"
}
```

Other states consume these via `terraform_remote_state` (as shown in question 5) or via data sources that look up the resource directly.

**Connecting modules within the same state:**

```hcl
module "networking" {
  source      = "./modules/networking"
  environment = var.environment
}

module "compute" {
  source     = "./modules/compute"
  vpc_id     = module.networking.vpc_id      # output from networking module
  subnet_ids = module.networking.subnet_ids
}
```

**When to use a data source vs managing the resource:**

| Scenario | Approach |
|---|---|
| Resource owned by another team/state | Data source — you read it, they manage it |
| Resource that predates your Terraform (manually created) | Data source to reference it, then consider `terraform import` to bring it under management |
| Resource shared across many states (VPC, DNS zone) | Data source in consuming states; one state owns and manages it |
| Resource only your config uses | Manage it directly — no data source needed |

**Key principle**: Data sources create a read-only dependency. If the upstream resource changes or is deleted, your `plan` will fail at the data source lookup. This is actually a feature — it surfaces breaking changes early instead of silently using stale values.

</details>

<details>
<summary>12. Show HCL examples for create_before_destroy and prevent_destroy lifecycle meta-arguments -- demonstrate create_before_destroy for a zero-downtime resource replacement (e.g., a TLS certificate or load balancer) and prevent_destroy for protecting a production database, explain what specific problem each solves, and what breaks without them</summary>

**`create_before_destroy` — zero-downtime replacements:**

Terraform's default behavior when a resource must be replaced (not just updated) is: destroy old, then create new. This causes downtime.

```hcl
resource "google_compute_ssl_certificate" "main" {
  name        = "app-cert-${formatdate("YYYYMMDDhhmmss", timestamp())}"
  private_key = file("private.key")
  certificate = file("certificate.crt")

  lifecycle {
    create_before_destroy = true
  }
}

resource "google_compute_target_https_proxy" "main" {
  name             = "app-proxy"
  ssl_certificates = [google_compute_ssl_certificate.main.id]
}
```

When the certificate is renewed, Terraform:
1. Creates the new certificate (with a new name)
2. Updates the HTTPS proxy to point to the new certificate
3. Destroys the old certificate

Without `create_before_destroy`, there's a window where no valid certificate exists, causing HTTPS errors for all traffic.

**Common use cases**: TLS certificates, instance templates used by managed instance groups, DNS records during migration, any resource where a gap in availability is unacceptable.

**Gotcha**: The resource must support having two instances coexist temporarily. That's why the name includes a timestamp — GCP requires unique names, so two certificates can't both be called `app-cert`. This is a common pattern with `create_before_destroy`.

**`prevent_destroy` — protecting critical resources:**

```hcl
resource "google_sql_database_instance" "prod" {
  name             = "prod-database"
  database_version = "POSTGRES_15"
  region           = "us-central1"

  settings {
    tier = "db-custom-4-16384"

    backup_configuration {
      enabled = true
    }
  }

  lifecycle {
    prevent_destroy = true
  }
}
```

If any Terraform operation would destroy this resource (explicit removal from config, a change that forces replacement, `terraform destroy`), Terraform exits with an error:

```
Error: Instance cannot be destroyed
  Resource google_sql_database_instance.prod has
  lifecycle.prevent_destroy set, but the plan calls
  for this resource to be destroyed.
```

This is a safety net against accidental database deletion — the most catastrophic class of infrastructure mistake.

**What it does NOT do**: It doesn't prevent modification. You can still change settings on the database. It only blocks destruction. To actually remove the resource, you must first remove the `prevent_destroy` flag from the config, apply that change, and then remove the resource.

**When to use it**: Production databases, persistent storage buckets with important data, IAM resources, anything where destruction means data loss. Don't use it on ephemeral or easily recreatable resources — it just adds friction for no safety benefit.

</details>

<details>
<summary>13. Write a reusable Terraform module for a common infrastructure pattern (e.g., a GCS bucket with IAM bindings or a VPC with subnets) -- show the directory structure, variables.tf with sensible defaults and validation, outputs.tf exposing necessary values, and how the calling root module consumes it. Explain what makes this module easy vs hard to reuse across teams and how you'd version it using git tags</summary>

**Directory structure:**

```
modules/
  gcs-bucket/
    main.tf
    variables.tf
    outputs.tf
    versions.tf
```

**`versions.tf`:**

```hcl
terraform {
  required_version = ">= 1.5"
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = ">= 5.0"
    }
  }
}
```

**`variables.tf`:**

```hcl
variable "name" {
  type        = string
  description = "Bucket name (must be globally unique)"

  validation {
    condition     = can(regex("^[a-z0-9][a-z0-9._-]{1,61}[a-z0-9]$", var.name))
    error_message = "Bucket name must be 3-63 chars, lowercase, digits, hyphens, dots."
  }
}

variable "project_id" {
  type        = string
  description = "GCP project ID"
}

variable "location" {
  type        = string
  default     = "US"
  description = "Bucket location (region or multi-region)"
}

variable "storage_class" {
  type    = string
  default = "STANDARD"

  validation {
    condition     = contains(["STANDARD", "NEARLINE", "COLDLINE", "ARCHIVE"], var.storage_class)
    error_message = "Must be one of: STANDARD, NEARLINE, COLDLINE, ARCHIVE."
  }
}

variable "versioning_enabled" {
  type    = bool
  default = true
}

variable "retention_days" {
  type        = number
  default     = null
  description = "Object retention in days. Null means no lifecycle rule."
}

variable "readers" {
  type        = list(string)
  default     = []
  description = "List of IAM members with objectViewer access (e.g., 'serviceAccount:sa@proj.iam.gserviceaccount.com')"
}

variable "writers" {
  type        = list(string)
  default     = []
  description = "List of IAM members with objectAdmin access"
}

variable "labels" {
  type    = map(string)
  default = {}
}
```

**`main.tf`:**

```hcl
resource "google_storage_bucket" "this" {
  name          = var.name
  project       = var.project_id
  location      = var.location
  storage_class = var.storage_class
  labels        = var.labels

  uniform_bucket_level_access = true

  versioning {
    enabled = var.versioning_enabled
  }

  dynamic "lifecycle_rule" {
    for_each = var.retention_days != null ? [1] : []
    content {
      action { type = "Delete" }
      condition { age = var.retention_days }
    }
  }
}

resource "google_storage_bucket_iam_member" "readers" {
  for_each = toset(var.readers)
  bucket   = google_storage_bucket.this.name
  role     = "roles/storage.objectViewer"
  member   = each.value
}

resource "google_storage_bucket_iam_member" "writers" {
  for_each = toset(var.writers)
  bucket   = google_storage_bucket.this.name
  role     = "roles/storage.objectAdmin"
  member   = each.value
}
```

**`outputs.tf`:**

```hcl
output "bucket_name" {
  value = google_storage_bucket.this.name
}

output "bucket_url" {
  value = google_storage_bucket.this.url
}

output "bucket_self_link" {
  value = google_storage_bucket.this.self_link
}
```

**Calling from a root module:**

```hcl
module "app_data_bucket" {
  source  = "git::https://github.com/my-org/terraform-modules.git//gcs-bucket?ref=v1.2.0"

  name       = "my-app-data-prod"
  project_id = var.project_id
  location   = "EU"
  readers    = ["serviceAccount:${google_service_account.app.email}"]
  labels     = local.common_labels
}

# Use the output
resource "google_cloud_run_service" "app" {
  # ...
  template {
    spec {
      containers {
        env {
          name  = "BUCKET_NAME"
          value = module.app_data_bucket.bucket_name
        }
      }
    }
  }
}
```

**What makes this easy to reuse:**

- Sensible defaults — you only need `name` and `project_id` to get a working bucket with versioning and uniform access
- Validation catches errors early with clear messages
- IAM is optional (empty lists by default) but built in when needed
- Outputs expose what consumers actually need (name, URL, self_link)
- No hardcoded organizational assumptions baked into the module

**What kills reusability:**

- Coupling unrelated concerns into one module (a bucket AND a Cloud Function AND app-specific IAM — now every consumer gets all or nothing)
- Hardcoded values that bake in one team's assumptions (specific project IDs, naming conventions)

**Versioning with git tags:**

```bash
git tag -a v1.2.0 -m "Add retention lifecycle rule support"
git push origin v1.2.0
```

Consumers pin to a version with `?ref=v1.2.0`. Breaking changes (removed variables, changed behavior) get a major version bump. This lets teams upgrade on their own schedule instead of being forced by upstream changes.

</details>

---

## Practical — CI/CD & Production Workflows

<details>
<summary>14. Design a CI/CD workflow for Terraform where `plan` runs on pull requests and `apply` runs on merge to main — show the pipeline configuration (e.g., CircleCI or GitHub Actions YAML), explain what guardrails you'd add (plan output as PR comment, manual approval for production, policy checks), and how you prevent two merges from racing to apply conflicting changes</summary>

**GitHub Actions workflow:**

```yaml
name: Terraform
on:
  pull_request:
    paths: ['infrastructure/**']
  push:
    branches: [main]
    paths: ['infrastructure/**']

permissions:
  contents: read
  pull-requests: write
  id-token: write  # for workload identity federation

env:
  TF_DIR: infrastructure/environments/prod

jobs:
  plan:
    if: github.event_name == 'pull_request'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: 1.9.0

      - name: Authenticate to GCP
        uses: google-github-actions/auth@v2
        with:
          workload_identity_provider: projects/123/locations/global/workloadIdentityPools/ci-pool/providers/github
          service_account: terraform-ci@my-project.iam.gserviceaccount.com

      - name: Terraform Init
        run: terraform -chdir=$TF_DIR init

      - name: Terraform Format Check
        run: terraform -chdir=$TF_DIR fmt -check

      - name: Terraform Validate
        run: terraform -chdir=$TF_DIR validate

      - name: Terraform Plan
        id: plan
        # -out=tfplan writes inside $TF_DIR; tee writes plan_output.txt to workspace root
        run: terraform -chdir=$TF_DIR plan -no-color -out=tfplan 2>&1 | tee plan_output.txt
        continue-on-error: true

      - name: Post Plan to PR
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const plan = fs.readFileSync('plan_output.txt', 'utf8');
            const truncated = plan.length > 60000 ? plan.substring(0, 60000) + '\n... truncated' : plan;
            github.rest.issues.createComment({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: context.issue.number,
              body: `### Terraform Plan\n\`\`\`\n${truncated}\n\`\`\``
            });

      - name: Fail if Plan Failed
        if: steps.plan.outcome == 'failure'
        run: exit 1

  apply:
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    concurrency:
      group: terraform-apply-prod  # prevents parallel applies
      cancel-in-progress: false    # queue, don't cancel
    steps:
      - uses: actions/checkout@v4

      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: 1.9.0

      - name: Authenticate to GCP
        uses: google-github-actions/auth@v2
        with:
          workload_identity_provider: projects/123/locations/global/workloadIdentityPools/ci-pool/providers/github
          service_account: terraform-ci@my-project.iam.gserviceaccount.com

      - name: Terraform Init
        run: terraform -chdir=$TF_DIR init

      - name: Terraform Apply
        run: terraform -chdir=$TF_DIR apply -auto-approve
```

**Key guardrails explained:**

1. **Plan as PR comment**: Every reviewer sees exactly what infrastructure will change. No "trust me, it's fine" merges. Truncate long plans to stay within GitHub API limits.

2. **`concurrency` group on apply**: This is how you prevent two merges from racing. GitHub Actions queues the second apply until the first finishes. `cancel-in-progress: false` ensures both changes eventually apply in order rather than dropping one.

3. **Terraform state locking** (as covered in question 4) is the second layer — even if two applies somehow start simultaneously, the remote backend lock prevents concurrent state writes.

4. **Format and validate checks**: Catch syntax errors and style issues before plan even runs.

**Additional guardrails for production teams:**

- **Manual approval gate**: Add an `environment: production` with required reviewers in GitHub so the apply step needs explicit approval after merge
- **Policy checks**: Run OPA/Conftest or Sentinel policies against the plan output to enforce rules like "no public buckets" or "databases must have backups enabled"
- **Saved plan files**: In stricter setups, save the plan file from the PR job, store it as an artifact, and have the apply job use that exact plan file — guaranteeing what was reviewed is exactly what's applied
- **Drift detection**: Schedule a nightly `plan` run (no apply) that alerts if drift is detected outside of PRs

</details>

<details>
<summary>15. How do you securely provide cloud provider credentials to Terraform in a CI/CD pipeline — compare workload identity federation (OIDC-based, no static keys) vs static service account keys, show how workload identity federation is configured for GCP or AWS, and explain why static credentials are a security risk even when stored as CI/CD secrets</summary>

**The problem with static service account keys:**

The traditional approach: create a service account, download a JSON key, store it as a CI/CD secret, and export it as `GOOGLE_APPLICATION_CREDENTIALS`.

Even stored as a CI/CD secret, this is risky:
- **Keys don't expire** — a leaked key works until manually revoked. If your CI provider is breached, the key works forever.
- **Keys are portable** — anyone with the JSON file can authenticate from anywhere. There's no binding to the CI environment.
- **Key rotation is manual** — most teams never rotate them. The key from 2 years ago is still active.
- **Blast radius** — the key has whatever permissions the service account has, usable from any IP, any context.
- **Audit gap** — it's hard to distinguish "CI ran this" from "someone copied the key and ran it from their laptop."

**Workload Identity Federation (WIF) — the modern approach:**

WIF lets your CI pipeline authenticate to GCP using an OIDC token issued by the CI provider (GitHub, GitLab, etc.). No static keys exist.

**How it works:**
1. GitHub Actions issues a short-lived OIDC token for each workflow run (contains repo name, branch, workflow, etc.)
2. GCP Workload Identity Pool is configured to trust tokens from GitHub's OIDC provider
3. A Workload Identity Provider maps claims from the token (repo, branch) to a GCP service account
4. Terraform gets a short-lived access token — valid for ~1 hour, scoped to that specific workflow run

**GCP configuration (Terraform, naturally):**

```hcl
# Workload Identity Pool
resource "google_iam_workload_identity_pool" "ci" {
  project                   = var.project_id
  workload_identity_pool_id = "ci-pool"
  display_name              = "CI/CD Pool"
}

# Provider — trust GitHub's OIDC tokens
resource "google_iam_workload_identity_pool_provider" "github" {
  project                            = var.project_id
  workload_identity_pool_id          = google_iam_workload_identity_pool.ci.workload_identity_pool_id
  workload_identity_pool_provider_id = "github"
  display_name                       = "GitHub Actions"

  attribute_mapping = {
    "google.subject"       = "assertion.sub"
    "attribute.repository" = "assertion.repository"
  }

  # Only trust tokens from your specific org
  attribute_condition = "assertion.repository_owner == 'my-org'"

  oidc {
    issuer_uri = "https://token.actions.githubusercontent.com"
  }
}

# Allow the pool to impersonate the Terraform service account
resource "google_service_account_iam_member" "wif_binding" {
  service_account_id = google_service_account.terraform_ci.name
  role               = "roles/iam.workloadIdentityUser"
  member             = "principalSet://iam.googleapis.com/${google_iam_workload_identity_pool.ci.name}/attribute.repository/my-org/infrastructure"
}
```

**GitHub Actions usage** (shown in question 14):

```yaml
- uses: google-github-actions/auth@v2
  with:
    workload_identity_provider: projects/123/locations/global/workloadIdentityPools/ci-pool/providers/github
    service_account: terraform-ci@my-project.iam.gserviceaccount.com
```

**Why WIF wins:**

| | Static Keys | Workload Identity Federation |
|---|---|---|
| Key material to manage | Yes (JSON file) | None |
| Expiration | Never (until revoked) | ~1 hour |
| Portable/replayable | Yes | No (bound to CI run) |
| Rotation needed | Yes (manual) | N/A (ephemeral) |
| Audit trail | "Service account was used" | "GitHub repo X, branch Y, workflow Z" |
| Setup complexity | Low | Medium (one-time) |

</details>

<details>
<summary>16. Configure a remote backend for team collaboration — show the HCL for a GCS or S3 backend with state locking enabled, explain the initialization process, what happens when you switch from local to remote state, and how you handle the chicken-and-egg problem of the backend bucket itself (do you Terraform the bucket that holds your Terraform state?)</summary>

**GCS backend configuration:**

```hcl
# backend.tf
terraform {
  backend "gcs" {
    bucket = "my-org-terraform-state"
    prefix = "prod/networking"  # isolates this state within the bucket
  }
}
```

GCS backends have state locking built in — no additional configuration needed.

**S3 backend configuration (AWS):**

```hcl
terraform {
  backend "s3" {
    bucket         = "my-org-terraform-state"
    key            = "prod/networking/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-locks"  # required for locking on S3
    encrypt        = true
  }
}
```

S3 locking has two options (as covered in Q4): the legacy approach uses a separate DynamoDB table, while Terraform v1.10+ supports native S3 locking via `use_lockfile = true`. GCS handles locking natively with no extra configuration.

**Initialization process:**

```bash
terraform init
```

On first run with a remote backend, Terraform:
1. Validates the backend configuration
2. Connects to the bucket and checks for existing state
3. If no state exists, creates an empty state file at the prefix/key
4. Downloads provider plugins and modules

**Migrating from local to remote:**

If you had local state and add a backend block, running `terraform init` detects the change:

```
Initializing the backend...
Do you want to copy existing state to the new backend?
  Enter a value: yes
```

Terraform copies `terraform.tfstate` from your local directory to the remote bucket. After confirming it works, delete the local state file. This is safe — Terraform handles the migration automatically.

**The chicken-and-egg problem:**

The bucket that stores Terraform state can't be managed by the Terraform config that uses it as a backend — the bucket must exist before `terraform init` runs.

**Common solutions:**

1. **Bootstrap script** (simplest): Create the state bucket with `gcloud` / `aws cli` outside of Terraform. This is a one-time operation:

```bash
# bootstrap.sh — run once, manually
gcloud storage buckets create gs://my-org-terraform-state \
  --project=my-project \
  --location=US \
  --uniform-bucket-level-access

gcloud storage buckets update gs://my-org-terraform-state \
  --versioning  # enables state file versioning for recovery
```

2. **Separate bootstrap Terraform** with local state: A tiny Terraform config that creates the state bucket and uses local state (committed to git or stored safely). This config is rarely changed and managed separately.

```hcl
# bootstrap/main.tf — uses LOCAL state
terraform {
  # Intentionally local — this IS the bootstrap
}

resource "google_storage_bucket" "terraform_state" {
  name     = "my-org-terraform-state"
  location = "US"
  project  = "my-project"

  versioning { enabled = true }
  uniform_bucket_level_access = true
}
```

3. **Self-referencing with import** (advanced): Create the bucket manually, then write Terraform that manages it using itself as a backend, and import the existing bucket. Works but feels circular.

Most teams use option 1 or 2. The pragmatic answer: creating one bucket manually is fine. Don't over-engineer the bootstrap.

</details>

## Practical — Debugging & Troubleshooting

<details>
<summary>17. Someone modified infrastructure directly in the cloud console (ClickOps) and now Terraform state is out of sync — walk through how you detect this drift with `terraform plan`, how you decide whether to update the HCL to match reality or revert the manual change by applying the original config, and what patterns prevent this from happening repeatedly in a team</summary>

**Step 1 — Detect the drift:**

```bash
terraform plan
```

Terraform refreshes state against reality and shows the diff. For example, if someone manually changed a firewall rule:

```
# google_compute_firewall.allow_https will be updated in-place
~ resource "google_compute_firewall" "allow_https" {
    ~ source_ranges = [
        - "10.0.0.0/8",    # was added manually in console
          "0.0.0.0/0",
      ]
  }
```

The `~` means update, `-` means Terraform wants to remove the manually added value. Terraform is showing: "reality doesn't match your config, here's what I'd do to fix it."

**Step 2 — Decide: adopt or revert:**

| Scenario | Action |
|---|---|
| Manual change was a hotfix for an outage | Update the HCL to match reality, then apply (no-op). The fix is now codified. |
| Manual change was unauthorized/accidental | Run `terraform apply` to revert reality back to the config. The manual change is undone. |
| Manual change reveals a missing feature in the config | Update HCL to include the new capability properly, then apply. |

**Key question**: "Is the manual change correct and intentional?" If yes, update the code. If no, let Terraform revert it.

**Step 3 — For resources created manually (not in state at all):**

If someone created a new resource in the console, Terraform doesn't know about it — it won't show in `plan`. You need to either:
- `terraform import google_compute_firewall.new_rule projects/my-proj/global/firewalls/new-rule` to bring it under management
- Or delete it manually if it shouldn't exist

**Patterns to prevent recurring drift:**

1. **Remove console write access**: Give engineers read-only access to the cloud console. All changes go through Terraform PRs. This is the most effective prevention.

2. **Scheduled drift detection**: Run `terraform plan` on a cron (nightly or hourly) and alert when drift is detected (non-empty plan):

```yaml
# Drift detection cron job
- name: Detect Drift
  run: |
    terraform plan -detailed-exitcode
    # Exit code 2 = changes detected (drift)
  # Alert on exit code 2
```

3. **Organizational policy constraints**: In GCP, use Organization Policies to restrict what can be done outside Terraform. In AWS, use SCPs (Service Control Policies).

4. **Team culture**: Make "all changes through code" a non-negotiable team norm. If someone made a console change during an incident, the follow-up action item is always "codify the change in Terraform."

5. **Tagging/labeling**: All Terraform-managed resources get a `managed_by = "terraform"` label. Alerts fire when untagged resources appear or tagged resources are modified outside of CI.

</details>

<details>
<summary>18. A `terraform plan` is failing with confusing dependency or cycle errors — explain how Terraform builds its dependency graph, the difference between implicit dependencies (resource references) and explicit `depends_on`, when `depends_on` is necessary vs when it hides a design problem, how you debug circular dependency errors by inspecting the graph with `terraform graph`, and when engineers use terraform apply -target to apply a subset of resources, what risks does this introduce (state drift, skipped dependencies) and why is it considered a last resort?</summary>

**How Terraform builds the dependency graph:**

Terraform parses all `.tf` files and builds a directed acyclic graph (DAG) by analyzing resource references. If resource B references an attribute of resource A (`google_compute_instance.app.network_interface[0].network_ip`), Terraform adds an edge: A must be created before B, and B must be destroyed before A.

This graph determines:
- Creation order (topological sort of dependencies)
- Parallelism (independent resources with no edges between them run concurrently)
- Destruction order (reverse of creation order)

**Implicit vs explicit dependencies:**

```hcl
# Implicit — Terraform infers from the reference
resource "google_compute_network" "vpc" {
  name = "my-vpc"
}

resource "google_compute_subnetwork" "subnet" {
  network = google_compute_network.vpc.id  # implicit dependency
}
```

```hcl
# Explicit — depends_on for hidden dependencies
resource "google_project_service" "compute_api" {
  service = "compute.googleapis.com"
}

resource "google_compute_instance" "app" {
  name         = "app"
  machine_type = "e2-small"

  # No direct reference to the API resource, but the instance
  # can't be created until the Compute API is enabled
  depends_on = [google_project_service.compute_api]
}
```

**When `depends_on` is necessary:**
- API enablement must happen before resources that use that API
- IAM bindings must propagate before a service account can access a resource (GCP IAM propagation delay)
- A resource depends on a side effect (e.g., a null_resource provisioner that runs a script)

**When `depends_on` hides a design problem:**
- If you need `depends_on` between two resources that should reference each other — you're probably missing a reference. Use the resource's output attribute instead.
- If you have a chain of `depends_on` across many resources — your module is doing too much and should be split.

**Debugging circular dependency errors:**

```bash
terraform graph | dot -Tsvg > graph.svg
```

This outputs the dependency graph in DOT format. Open the SVG to visually trace cycles. Common causes:
- Resource A references resource B, and B references A (redesign one to not need the other)
- A module outputs feed back into its own inputs (module design problem)
- Security groups referencing each other (use `aws_security_group_rule` as separate resources instead of inline rules)

**`terraform apply -target` — surgical apply and its risks:**

```bash
terraform apply -target=google_compute_instance.app
```

This applies only the targeted resource and its dependencies, ignoring everything else.

**Risks:**

- **State drift**: Other resources in state don't get refreshed. State can diverge from reality for non-targeted resources.
- **Skipped dependencies**: If the target depends on something that needs updating, Terraform may not catch it — leading to partial states.
- **Snowflake state**: After multiple targeted applies, state becomes inconsistent. A full `plan` later may show unexpected changes.
- **False confidence**: The targeted apply succeeded, but a full apply might fail due to conflicts with the skipped resources.

**When it's acceptable (last resort only):**

- Breaking a circular dependency during initial bootstrapping (e.g., create the IAM binding first, then the resource that needs it)
- Recovering from a partially failed apply where one resource is blocking everything
- Emergency production fix where a full apply would touch too many resources

**Always follow a targeted apply with a full `terraform plan`** to verify state consistency. If the full plan shows no changes, you're safe.

</details>

---

## Experience-Based Questions

These questions test real-world experience. Prepare by mapping them to your own projects and situations.

<details>
<summary>19. Tell me about a time you designed the Terraform repository structure, module strategy, or CI/CD workflow for a team — what decisions did you make about state splitting, environment management, and code reuse, and what tradeoffs did you encounter as the team and infrastructure grew?</summary>

**What the interviewer is looking for:**

- Evidence you've made real infrastructure architecture decisions, not just written resources
- Understanding of tradeoffs (not just "we did X" but "we chose X over Y because...")
- Awareness of how IaC decisions compound as teams and infrastructure grow
- Ability to reflect on what worked and what you'd change

**Suggested structure (STAR-like):**

1. **Context**: What was the team/project? How big was the infrastructure? What existed before?
2. **Decisions**: Walk through 2-3 key decisions you made and why
3. **Tradeoffs**: What did you give up? What problems did the decisions create later?
4. **Evolution**: How did the approach hold up as things grew? What would you do differently?

**Key points to hit:**

- **State splitting rationale**: "We split by service and environment because [blast radius / lock contention / team ownership]. We started monolithic and hit [specific pain point] that forced the split."
- **Environment management**: "We used directory-based structure because [workspaces had X problem]. Each environment has its own tfvars and backend config."
- **Module strategy**: "We created modules for patterns used by 3+ services. We deliberately avoided modularizing [X] because it was still changing. We versioned with git tags so teams could upgrade independently."
- **CI/CD**: "Plan on PR, apply on merge. We added [specific guardrail] after [incident/close call]. Workload identity federation replaced static keys because [security reason]."

**Example outline to personalize:**

"At [company], we managed infrastructure for a microservices platform across 3 environments. Initially everything was in one state file — plan took 4 minutes and we had constant lock contention with 5 engineers. I led the restructuring to split state by service layer (networking, databases, compute per service). The main tradeoff was needing `terraform_remote_state` data sources to share VPC IDs and database endpoints between states, which added coupling between states. For environment management, we used a directory structure with shared modules. The module that caused the most friction was our 'service' module — it tried to do too much (compute + IAM + monitoring) and had 25 variables. I eventually split it into smaller, composable modules. Looking back, I'd split state earlier and keep modules smaller from the start."

</details>

<details>
<summary>20. Describe a time you debugged a tricky Terraform issue — unexpected resource destruction, plan/apply mismatch, provider bugs, or mysterious diffs that wouldn't go away. How did you systematically diagnose the root cause and what was the fix?</summary>

**What the interviewer is looking for:**

- Systematic debugging approach (not "I randomly tried things until it worked")
- Understanding of Terraform internals (state, graph, provider behavior)
- Ability to distinguish between user error, provider bugs, and design problems
- Composure under pressure (Terraform issues often involve production infrastructure)

**Suggested structure:**

1. **The symptom**: What did you see? (Unexpected destroy in plan, perpetual diff, apply error)
2. **Diagnosis steps**: How did you narrow it down? What tools/commands did you use?
3. **Root cause**: What was actually wrong?
4. **Fix**: What did you do? How did you verify it?
5. **Prevention**: What did you put in place to prevent recurrence?

**Key debugging techniques to reference:**

- `terraform plan` with `-detailed-exitcode` to detect drift programmatically
- `terraform state show <resource>` to inspect what state thinks vs reality
- `terraform state list` to find orphaned or misnamed resources
- `TF_LOG=DEBUG terraform plan` for provider-level debugging
- `terraform graph | dot -Tsvg` to visualize dependency issues
- `terraform state mv` to fix resource address changes without destroy/recreate
- Reading the provider source code on GitHub when provider behavior is unclear

**Example outlines to personalize:**

*Unexpected destruction*: "During a routine PR, the plan showed 12 resources being destroyed and recreated. The root cause was switching from `count` to `for_each` on a set of GCS buckets — the state addresses changed from `[0]`, `[1]` to `["name1"]`, `["name2"]`. I used `terraform state mv` to rename each resource in state to match the new addresses, then the plan showed no changes. We added a team rule: any `count` to `for_each` migration requires a state migration plan."

*Perpetual diff*: "Every plan showed a diff on a GKE node pool even after applying. The provider was normalizing a field differently than how we specified it (OAuth scopes ordering). Fix was to sort our input list to match the provider's normalization. I found this by running `terraform state show` and comparing the stored value to our config."

*Plan/apply mismatch*: "Plan showed no changes, but apply failed with a 409 conflict. The resource had been modified between plan and apply (no saved plan file). Fix was to always use `-out=plan.tfplan` in CI and apply from the saved plan. Also added a `concurrency` group to prevent parallel applies."

</details>
