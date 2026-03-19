# ArgoCD & GitOps — A Complete Reference

> Written 2026-03-17 after first exposure to ArgoCD at commercetools.
> Covers general concepts first, then maps them to exactly how the UAA team uses them.

---

## Table of Contents

1. [What is GitOps?](#1-what-is-gitops)
2. [What is ArgoCD?](#2-what-is-argocd)
3. [Core ArgoCD Concepts](#3-core-argocd-concepts)
4. [How Deployments Work — The Image Update Problem](#4-how-deployments-work--the-image-update-problem)
5. [Argo Rollouts vs Kubernetes Deployments](#5-argo-rollouts-vs-kubernetes-deployments)
6. [Why a Separate GitOps Repo?](#6-why-a-separate-gitops-repo)
7. [How We Use ArgoCD at commercetools (UAA)](#7-how-we-use-argocd-at-commercetools-uaa)
8. [The CircleCI Pipeline — How Builds and Deploys Work](#8-the-circleci-pipeline--how-builds-and-deploys-work)
9. [The Full Deployment Flow End-to-End](#9-the-full-deployment-flow-end-to-end)
10. [Reading the ArgoCD UI](#10-reading-the-argocd-ui)
11. [Quick Reference — As a Developer on This Team](#11-quick-reference--as-a-developer-on-this-team)

---

## 1. What is GitOps?

GitOps is an operational model where **Git is the single source of truth for what should be running in your infrastructure**.

Instead of a developer or CI system running `kubectl apply` or clicking "deploy" in some UI, the workflow is:

1. You push a commit to a Git repository
2. An automated tool detects the difference between what Git says should exist and what actually exists in the cluster
3. The tool reconciles — it applies changes to make the cluster match Git

The key insight: **the cluster always tries to look like the Git repo**. If someone manually changes something directly in the cluster, the GitOps tool will revert it. Git wins. Always.

### Why GitOps matters

| Problem                                            | GitOps solution                                              |
| -------------------------------------------------- | ------------------------------------------------------------ |
| "Who deployed this and when?"                      | Every change is a Git commit — `git log` is your audit trail |
| "How do I roll back?"                              | `git revert` the commit, the tool syncs the cluster back     |
| "What's actually running in prod?"                 | Read the Git repo — it's the truth                           |
| "Someone manually changed a config in the cluster" | GitOps tool detects drift and reverts it                     |
| "I need prod access to deploy"                     | No — you push a PR, the tool deploys                         |

### GitOps vs traditional CI/CD

```
Traditional:
Code PR → CI builds → CI runs "kubectl apply" → prod updated

GitOps:
Code PR → CI builds image → CI commits image ref to a config repo
                                        ↓
                              GitOps tool detects change
                                        ↓
                              GitOps tool applies to cluster
```

The difference: in GitOps, **CI never touches the cluster**. It only updates Git. The GitOps tool owns the cluster.

---

## 2. What is ArgoCD?

ArgoCD is the most widely used GitOps tool for Kubernetes. It runs inside your cluster and continuously watches a Git repository. When it detects that the Git repo diverges from what's in the cluster, it syncs them.

ArgoCD is developed by the Argo Project (part of the CNCF ecosystem), the same project that makes Argo Workflows and Argo Rollouts.

### What ArgoCD does

- Watches a Git repo (or specific branch/path within it)
- Compares the desired state (Git) to the actual state (cluster)
- Reports health and sync status in a UI
- Syncs automatically or on demand
- Stores rollback history — you can roll back to any previous sync

### What ArgoCD does NOT do

- It does not build Docker images
- It does not run tests
- It does not update image tags by itself (something else must do that)
- It does not manage Git branches or PRs

ArgoCD is purely a **sync engine** — Git → cluster.

---

## 3. Core ArgoCD Concepts

### Application

The central unit in ArgoCD. An **Application** is a configuration object that tells ArgoCD:

- Which Git repo to watch
- Which branch/path within that repo
- Which Kubernetes cluster to deploy to
- Which namespace to deploy into

```yaml
# What an ArgoCD Application looks like conceptually
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: agent-gateway-prd-aws-eu-central1-v1
spec:
  source:
    repoURL: https://github.com/commercetools/uaa-gitops
    targetRevision: production/live # branch to watch
    path: prd-aws-eu-central1-v1/agent-gateway/agent-gateway
  destination:
    server: https://prd-aws-eu-central1-v1 # target cluster
    namespace: agent-gateway
  syncPolicy:
    automated: {} # auto-sync when Git changes
```

One Application = one deployment target. If you deploy to 4 clusters, you have 4 Applications.

### Sync Status

Whether the cluster matches Git:

| Status        | Meaning                                        |
| ------------- | ---------------------------------------------- |
| **Synced**    | Cluster matches Git exactly                    |
| **OutOfSync** | Git has changes not yet applied to the cluster |
| **Unknown**   | ArgoCD can't determine the state               |

### Health Status

Whether the running resources are actually healthy:

| Status          | Meaning                                           |
| --------------- | ------------------------------------------------- |
| **Healthy**     | All resources running and ready                   |
| **Progressing** | Deployment in progress (rolling update happening) |
| **Degraded**    | Something is broken (crash loops, failed pods)    |
| **Suspended**   | Intentionally paused                              |

### Project

ArgoCD Projects group Applications and define RBAC — which teams can access which apps, which clusters, which repos. At commercetools, the UAA team's apps all belong to the `uaa` project.

### Sync Policy — Manual vs Automated

- **Manual sync**: ArgoCD detects drift but waits for a human to click "Sync"
- **Automated sync**: ArgoCD syncs immediately when it detects a Git change

At commercetools, auto-sync is enabled (`Auto sync is enabled` visible in the UI).

---

## 4. How Deployments Work — The Image Update Problem

This is the most important thing to understand when you first encounter GitOps.

### The problem

ArgoCD watches Git manifests. Your Kubernetes manifests contain an image reference like:

```yaml
image: gcr.io/commercetools-platform/agent-gateway/agent-gateway-api@sha256:abc123...
```

When you push new application code:

1. CI builds a new Docker image with a new SHA
2. The new image is pushed to GCR (Google Container Registry)
3. **ArgoCD has no idea** — the Git manifest still references the old SHA
4. Nothing gets deployed

### The solution: CI must write back to Git

For GitOps to work, something needs to update the image reference in Git after a new image is built. There are several approaches:

**Option A — CI commits back (used by commercetools via Serenity)**

```
Code merged → CI builds new image → bot opens PR on gitops repo
                                       updating image SHA in manifest
                                    → PR merges → ArgoCD syncs
```

**Option B — ArgoCD Image Updater**
A sidecar tool watches your container registry for new tags and automatically commits updates to your gitops repo. Removes the CI step but adds another moving part.

**Option C — Mutable tags (anti-pattern, avoid)**
Always push to `latest`. ArgoCD doesn't detect a change (manifest didn't change), but `imagePullPolicy: Always` + a forced rollout restarts pods with the new image. No audit trail — you can't tell from Git what's actually running.

### Why SHA digest instead of a tag?

In the commercetools setup you'll see images referenced by SHA digest:

```
gcr.io/commercetools-platform/agent-gateway/agent-gateway-api@sha256:3c2f0c67...
```

Not by tag like `v1.2.3` or `main`. Tags are mutable — someone can push a different image to the same tag. **SHA digests are immutable** — a SHA permanently identifies a specific image, byte for byte. If you look at the gitops repo, you always know exactly what is running.

---

## 5. Argo Rollouts vs Kubernetes Deployments

These are two different ways to manage how pods are updated when you deploy a new image.

### Standard Kubernetes Deployment

The built-in way. When you update the image:

- Kubernetes performs a **rolling update** — gradually replaces old pods with new ones
- Default: 25% of pods updated at a time
- No traffic control — all traffic routes to both old and new pods during the transition
- No built-in analysis — it doesn't automatically check if the new version is healthy before proceeding

```yaml
apiVersion: apps/v1
kind: Deployment
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 0
```

### Argo Rollouts

A Kubernetes controller that replaces `kind: Deployment` with `kind: Rollout`. Same core concept, but with advanced strategies:

**Canary deployment** — gradually shift traffic to the new version:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
spec:
  strategy:
    canary:
      steps:
        - setWeight: 10 # send 10% of traffic to new version
        - pause:
            duration: 1h # wait 1 hour
        - setWeight: 50 # send 50% to new version
        - pause: {} # wait for manual approval
        # then 100% if promoted
```

**Blue-Green deployment** — run two full environments simultaneously:

```yaml
spec:
  strategy:
    blueGreen:
      activeService: agent-gateway-active # all traffic
      previewService: agent-gateway-preview # new version, no traffic yet
      # promote manually or after analysis passes
```

### Migration: Deployment → Rollout

The change is mostly mechanical. You swap two fields and add a strategy block:

```diff
- apiVersion: apps/v1
- kind: Deployment
+ apiVersion: argoproj.io/v1alpha1
+ kind: Rollout
  metadata:
    name: agent-gateway-api
  spec:
+   strategy:
+     canary:
+       steps:
+         - setWeight: 20
+         - pause: {}
    # everything else stays the same
    selector: ...
    template: ...
```

> The UAA team has a ticket to do this migration. Currently agent-gateway uses standard Kubernetes Deployments. Once migrated, the same Serenity-driven deployment flow applies — only the rollout _behaviour_ inside the cluster changes.

### The `rollout/restartedAt` annotation

Both standard Deployments and Argo Rollouts respect this annotation as a trigger:

```yaml
metadata:
  annotations:
    rollout/restartedAt: "2026-03-17T09:00:35Z"
```

Kubernetes is declarative — if the manifest hasn't changed, applying it again does nothing. This annotation is the trick to force a restart: bumping the timestamp makes the manifest different, which gives ArgoCD something to sync and Kubernetes something to act on. It's equivalent to running `kubectl rollout restart deployment/agent-gateway-api` but tracked in Git.

---

## 6. Why a Separate GitOps Repo?

When you first encounter the setup, this is the thing that trips people up: ArgoCD watches `uaa-gitops`, a completely separate repository from `agent-gateway-services` where the actual application code lives. This is a deliberate GitOps design called the **app repo / config repo split** — the approach recommended by the ArgoCD project and widely adopted.

The two repos have completely different characters:

| Aspect           | App repo (`agent-gateway-services`) | Config repo (`uaa-gitops`)                               |
| ---------------- | ----------------------------------- | -------------------------------------------------------- |
| What it contains | Source code, application logic      | Rendered Kubernetes manifests                            |
| Who touches it   | Developers                          | Serenity bot + developers (for chart changes)            |
| Branch model     | Feature branches, PRs               | Environment branches (`staging/live`, `production/live`) |
| Review model     | Human review required               | Bot auto-merges                                          |
| Commit history   | Code changes, refactors, tests      | Pure deployment log                                      |

### Why not put manifests in the same repo?

**1. Different audit trails for different concerns**

The commit history of `uaa-gitops` is a clean, pure deployment log:

```
22cf15a  ct-serenity-prod[bot]  deploy agent-gateway sha256:474f... to staging
891a3f2  ct-serenity-prod[bot]  deploy mcp-config-service sha256:91bc... to prod
7d4019c  Yahia Eltai            increase HPA maxReplicas from 10 to 100
```

If manifests lived inside `agent-gateway-services`, every feature PR, refactor, test fix, and docs change would pollute this history. When you're debugging "what changed between these two production deploys?", you want a clean signal. Mixing application churn into that history destroys it.

**2. The bot should not have write access to your application code**

Serenity auto-merges PRs without human review. That's safe for a dedicated gitops repo where all it ever does is update rendered manifests produced by a controlled CI pipeline. But if manifests lived in `agent-gateway-services`, Serenity would need merge permissions on your application source repo — a real security risk. An automated bot with write access to your application repo could, if misconfigured, overwrite application code.

**3. Different review requirements for different PR types**

Application code changes need developer review, CI checks, passing tests. Deployment changes (image SHA bump) are purely mechanical and deterministic — nothing to review. These two review models are fundamentally incompatible in a single repo. Separate repos make the rule trivial: all Serenity PRs to `uaa-gitops` auto-merge, all developer PRs to `agent-gateway-services` require review.

**4. Multiple teams, shared clusters**

SRE manages ArgoCD and the cluster infrastructure. The UAA team owns `uaa-gitops`; DEVX likely has their own gitops repo. Each team fully controls their repo without needing access to other teams' application code. SRE configures ArgoCD Applications pointing at each team's gitops repo without needing to read application source. If manifests lived inside application repos, SRE would need read access to every application repo just to configure ArgoCD.

**5. Independent rollback**

You can roll back a deployment (revert a commit in `uaa-gitops`) without touching application code, re-deploy the same image after rolling back without creating a new code commit, and keep production at a specific image while staging advances independently. If manifests and code lived together, a deployment rollback would entangle with the code commit history.

---

## 7. How We Use ArgoCD at commercetools (UAA)

### Serenity — the bridge bot

Serenity (`ct-serenity-prod[bot]`) is an internal commercetools tool that automates image promotion to `uaa-gitops`. It is **not** an autonomous watcher — it is invoked directly by CircleCI as two separate commands at different points in the pipeline.

**`serenity sync`** — creates the PRs in `uaa-gitops`:

1. Reads `serenity.yaml` to know which clusters and values files to target
2. Takes the Helm chart from `agent-gateway-services/k8s/agent-gateway/`, overrides `dockerImageDigest` with the new SHA, and runs `helm template` to produce rendered YAML. The values files (e.g. `k8s/agent-gateway/values/ctp_staging_aws_eu-central-1_v1.yaml`) stay in `agent-gateway-services` — they are render inputs, not committed to `uaa-gitops`
3. Commits the rendered `manifest.yaml` to `uaa-gitops` for each configured cluster across **both staging and production**, also bumping `rollout/restartedAt`
4. Opens PRs into `staging/live` and `production/live` — but does **not** merge them

**`serenity approve --env=<env>`** — merges the open PR for that specific environment. Called separately for staging and production, each gated behind its own CircleCI approval step.

Each service has a `serenity.yaml` in `k8s/<service>/serenity.yaml` inside `agent-gateway-services`. This defines which clusters to target and which values files to use for each environment:

**`k8s/agent-gateway/serenity.yaml`:**

```yaml
chart: .
name: agent-gateway
staging:
  clusters:
    - name: stg-aws-eu-central1-v1
      valuesFile: ./values/ctp_staging_aws_eu-central-1_v1.yaml
    - name: stg-gcp-eu-west1-v1
      valuesFile: ./values/ctp_staging_gcp_europe-west1_v1.yaml
production:
  clusters:
    - name: prd-aws-eu-central1-v1
      valuesFile: ./values/ctp_production_aws_eu-central-1_v1.yaml
    - name: prd-aws-us-east2-v1
      valuesFile: ./values/ctp_production_aws_us-east-2_v1.yaml
    - name: prd-gcp-us-central1-v1
      valuesFile: ./values/ctp_production_gcp_us-central1_v1.yaml
    - name: prd-gcp-eu-west1-v1
      valuesFile: ./values/ctp_production_gcp_europe-west1_v1.yaml
    - name: prd-gcp-au-southeast1-v1
      valuesFile: ./values/ctp_production_gcp_australia-southeast1_v1.yaml
```

Both staging and production are defined here. `serenity sync` creates PRs for all environments in one shot; `serenity approve` merges them selectively by env.

### What `uaa-gitops` actually contains

`uaa-gitops` does **not** contain Helm charts or values files. It contains the **fully pre-rendered output of `helm template`** — plain, flat Kubernetes YAML with all values already substituted in. ArgoCD applies the plain YAML directly — no Helm processing happens inside ArgoCD at all.

Each `manifest.yaml` is a single file containing all Kubernetes resources for that service concatenated with `---` separators. For agent-gateway that includes:

| Resource                   | Purpose                                                        |
| -------------------------- | -------------------------------------------------------------- |
| `ServiceAccount`           | Identity for the pods                                          |
| `ConfigMap` (vault-agent)  | Vault agent config with cluster-specific secret paths baked in |
| `Service`                  | Internal load balancer (ClusterIP)                             |
| `Deployment`               | The app with image SHA hardcoded inline                        |
| `HorizontalPodAutoscaler`  | Auto-scaling (min 3, max 100 replicas at 70% CPU)              |
| `PrometheusRule`           | Alerting rules (fires if >75% replicas unavailable for 5m)     |
| `ServiceMonitor`           | Tells Prometheus how to scrape `/metrics`                      |
| `TargetGroupConfiguration` | AWS load balancer routing config                               |

mcp-config-service has some differences: it uses an Istio `VirtualService` instead of `TargetGroupConfiguration` (it routes through the commercetools API gateway), and it has a database migration `Job` with an ArgoCD PreSync hook (see section 10).

When Serenity "updates the image", it re-renders the Helm chart and replaces the entire manifest file. The image SHA is a literal string in the Deployment resource:

```yaml
image: gcr.io/commercetools-platform/agent-gateway/agent-gateway-api@sha256:3c2f0c67593967...
```

That text change is the Git commit ArgoCD detects.

### The gitops repo structure

ArgoCD is configured with different paths per cluster. Each cluster gets its own folder:

```
uaa-gitops/
├── prd-aws-eu-central1-v1/
│   ├── agent-gateway/agent-gateway/manifest.yaml      ← all agent-gateway k8s resources, rendered
│   └── mcp-config-service/mcp-config-service/manifest.yaml
├── prd-aws-us-east2-v1/
│   ├── agent-gateway/agent-gateway/manifest.yaml
│   └── mcp-config-service/mcp-config-service/manifest.yaml
├── stg-aws-eu-central1-v1/
│   └── ...
└── stg-gcp-eu-west1-v1/
    └── ...
```

### Environments and clusters

For agent-gateway there are 7 ArgoCD Applications (one per cluster):

| ArgoCD App name                          | Environment | Cloud/Region           |
| ---------------------------------------- | ----------- | ---------------------- |
| `agent-gateway-prd-aws-eu-central1-v1`   | Production  | AWS Europe (Frankfurt) |
| `agent-gateway-prd-aws-us-east2-v1`      | Production  | AWS US (Virginia)      |
| `agent-gateway-prd-gcp-us-central1-v1`   | Production  | GCP US (Iowa)          |
| `agent-gateway-prd-gcp-eu-west1-v1`      | Production  | GCP Europe (Belgium)   |
| `agent-gateway-prd-gcp-au-southeast1-v1` | Production  | GCP Australia (Sydney) |
| `agent-gateway-stg-aws-eu-central1-v1`   | Staging     | AWS Europe (Frankfurt) |
| `agent-gateway-stg-gcp-eu-west1-v1`      | Staging     | GCP Europe (Belgium)   |

### Branches in uaa-gitops

| Branch                          | Used for                                                                    |
| ------------------------------- | --------------------------------------------------------------------------- |
| `production/live`               | What's deployed to all production clusters — ArgoCD watches this            |
| `staging/live`                  | What's deployed to all staging clusters — ArgoCD watches this               |
| `development/live`              | Dev environment (likely used for early testing)                             |
| `production/agent-gateway`      | Serenity's working branch for agent-gateway → merges into `production/live` |
| `production/mcp-config-service` | Same pattern for mcp-config-service                                         |
| `staging/agent-gateway`         | Serenity's working branch for agent-gateway staging                         |
| `staging/mcp-config-service`    | Same for mcp-config-service staging                                         |

`production/live` is branch-protected — direct pushes are blocked, even for bots. Serenity creates a dedicated branch per service per environment, commits the re-rendered manifest there, opens a PR into `{env}/live`, and auto-merges it. Per-service branches avoid race conditions when both services deploy simultaneously. Even with auto-merge, a human can still close/block the PR in the short window before merge.

### Who owns what

| Thing                                       | Owner                   |
| ------------------------------------------- | ----------------------- |
| ArgoCD installation in clusters             | SRE team                |
| ArgoCD project config, RBAC                 | SRE team                |
| `uaa-gitops` repo content                   | UAA team + Serenity bot |
| Serenity config (`serenity.yaml`)           | UAA team                |
| Application code (`agent-gateway-services`) | UAA team                |

As a UAA developer, you interact with this infrastructure by merging PRs to `agent-gateway-services` (triggers Serenity → deployment), making direct changes to `uaa-gitops` for chart changes (HPA, config, new resources), and reading the ArgoCD UI to check deployment health.

---

## 8. The CircleCI Pipeline — How Builds and Deploys Work

The pipeline lives in `.circleci/config/workflows/test-build-and-deploy.yml` and `.circleci/config/jobs/`. It has **three distinct phases** separated by manual approval gates: build, sync (create PRs), and deploy (merge PRs).

### The full pipeline shape

```
install_and_cache + vault_get_ci_secrets
       ↓
[parallel] unit tests · typecheck · lint · k8s lint · build_application x3 (no push)
       ↓
▶ can_d  ← MANUAL APPROVAL (gates image push)
       ↓
build_application x3 (push_image_to_repo: true)  ← images pushed to GCR for all 3 services
       ↓
sync_application x3 (serenity sync)  ← creates PRs in uaa-gitops for BOTH staging + production
       ↓
▶ can_d_staging              ▶ can_d_prod  ← separate MANUAL APPROVALs
       ↓                            ↓
deploy_application staging x3   deploy_application prod x3  ← serenity approve: merges the PRs
(mcp-config → multitenant →     (mcp-config → multitenant →
 agent-gateway, in order)        agent-gateway, in order)
```

You cannot deploy without clicking approve at least twice. The build-only step confirms the Docker build works before you commit to pushing anything.

### Job 1 — `build_application`: build (and optionally push) the image

This job runs `pnpm --filter "@commercetools-local/<service>*" run package` for the service.

**When `push_image_to_repo: false`** (first run, before `can_d` approval):

- Builds the Docker image locally to verify it compiles
- Does **not** push to GCR
- Exits early

**When `push_image_to_repo: true`** (after `can_d` approval):

- Builds the image again (using Docker layer cache from the first run)
- Pushes two tags to GCR:
  ```
  gcr.io/commercetools-platform/agent-gateway/agent-gateway-api:latest
  gcr.io/commercetools-platform/agent-gateway/agent-gateway-api:<git-commit-sha>
  ```
- Fetches the immutable **SHA digest** from the registry after push
- Writes that digest to `.circleci/_docker_image_digests/agent-gateway-api`
- **Persists that file to the CircleCI workspace** so the sync and deploy jobs can read it later

The image is tagged with both `:latest` and `:<git-commit-sha>` at push time. This is the link between a Git commit SHA and the running image — if you look up a Docker SHA in GCR, you'll find it also has the git commit SHA as a tag, letting you trace exactly which code version produced that image.

### Traceability — connecting a deployment back to application code

This is a genuine gap in the current setup. ArgoCD only shows the Docker SHA in `uaa-gitops` manifests. The Serenity PR title also only contains the Docker SHA, not the git commit SHA. There is no direct link from the ArgoCD UI back to `agent-gateway-services`.

The full trace path today is manual:

```
ArgoCD UI → Docker SHA (e.g. sha256:3c2f0c67...)
GCR → look up that Docker SHA → also tagged :<git-commit-sha> (e.g. :474f6035)
agent-gateway-services @ 474f6035 → exact code that was deployed
```

A better pattern exists and is already used in other repos at commercetools (e.g. audit-log): embed the git SHA directly as an annotation or label in the manifest, so it's visible in the ArgoCD UI and in `uaa-gitops` commit history without needing to go through GCR.

For now, if you need to find out when a specific commit was deployed, the reliable path is: find the CircleCI pipeline run for that commit on main, look at the `sync_application` job, and match the Docker SHA it passed to Serenity against the ArgoCD sync history.

### Job 2 — `sync_application`: create the gitops PRs with `serenity sync`

This job runs immediately after `build_application` pushes the images — no approval gate between them.

1. **Attaches the workspace** to get the digest file written by `build_application`
2. **Reads the digest**:
   ```bash
   DIGEST=$(cat .circleci/_docker_image_digests/agent-gateway-api)
   echo "export IMAGE_DIGEST=${DIGEST}" >> $BASH_ENV
   ```
3. **Installs Serenity CLI** (`serenity/install`)
4. **Calls `serenity sync`**:
   ```bash
   serenity sync \
     --config-file="k8s/agent-gateway/serenity.yaml" \
     --image-tag="${IMAGE_DIGEST}" \
     --image-tag-value-path="agentGatewayApi.dockerImageDigest" \
     --wait=true
   ```

`serenity sync` reads `serenity.yaml`, renders the Helm chart for **every cluster across both staging and production**, commits the manifest files to `uaa-gitops`, and opens PRs into `staging/live` and `production/live`. It does **not** merge them.

The `--image-tag-value-path` tells Serenity which Helm value to override during rendering — the dot-notation path into the values file that becomes the literal image reference in the rendered YAML.

### Job 3 — `deploy_application`: merge the gitops PR with `serenity approve`

After the human approves `can_d_staging` or `can_d_prod`, this job runs:

```bash
serenity approve \
  --config-file="k8s/agent-gateway/serenity.yaml" \
  --env="staging"  # or "production"
  --wait=true
```

`serenity approve` finds the open PR that `serenity sync` created for that environment and merges it into `{env}/live`. ArgoCD then detects the new commit and syncs the cluster.

**Deployment order constraint**: For both staging and production, agent-gateway is deployed last — it requires `d_mcp-config-service` and `d_multitenant-mcp-service` to complete first. This ensures mcp-config-service (which agent-gateway depends on to fetch configs) is running the new version before agent-gateway starts forwarding traffic.

### The CircleCI workspace — how data passes between jobs

CircleCI jobs are isolated — they don't share a filesystem by default. The **workspace** is the mechanism for passing files between jobs in a pipeline:

```
build_application (push)            sync_application / deploy_application
    ↓                                          ↑
writes digest to                     reads digest from
.circleci/_docker_image_digests/     .circleci/_docker_image_digests/
agent-gateway-api                    agent-gateway-api
    ↓                                          ↑
persist_to_workspace ────────── attach_build_workspace
```

### Why three separate approval gates?

- `can_d` — gates the image push. You don't want to push an image you haven't decided to deploy yet.
- `can_d_staging` / `can_d_prod` — gates the PR merge. `serenity sync` has already created the PRs in uaa-gitops at this point; these approvals control when they get merged (and therefore when ArgoCD syncs).
- `can_d_prod` is also filtered to the default branch (`main`) only — you can't accidentally deploy a feature branch to production.

---

## 9. The Full Deployment Flow End-to-End

### Case 1: Code change (new feature, bug fix)

```
1.  You push a PR to agent-gateway-services
2.  PR review + merge to main

--- CircleCI pipeline starts automatically ---

3.  Tests, typecheck, lint run in parallel
4.  build_application x3 (build only) — confirms Docker builds work, no push yet
5.  ▶ MANUAL APPROVAL: can_d
6.  build_application x3 (push) — pushes all 3 service images to GCR
    (:latest and :<commit-sha> tags), fetches SHA digests, persists to workspace
7.  sync_application x3 — calls serenity sync for each service
    Serenity re-renders the Helm chart, commits manifest.yaml for ALL clusters
    (both staging and production) to uaa-gitops, opens PRs:
      staging/agent-gateway → staging/live
      production/agent-gateway → production/live
    PRs are open but NOT yet merged

--- PRs exist in uaa-gitops, waiting for approval ---

8.  ▶ MANUAL APPROVAL: can_d_staging
9.  deploy_application staging x3 (in order: mcp-config, multitenant, agent-gateway)
    Each calls serenity approve --env=staging, which merges the staging PR
10. ArgoCD detects commits on staging/live, syncs both staging clusters
11. Kubernetes rolling restart with new image

--- Staging verified ---

12. ▶ MANUAL APPROVAL: can_d_prod (main branch only)
13. deploy_application production x3 (same order: mcp-config, multitenant, agent-gateway)
    Each calls serenity approve --env=production, which merges the production PR
14. ArgoCD detects commits on production/live, syncs all 5 production clusters
15. New pods come up with new image, old pods terminated
16. Slack notification: ":tada: Agent Gateway deployed to production"
```

### Case 2: Chart change (adding HPA, changing resource limits, new ConfigMap)

```
1. Edit the manifest directly in uaa-gitops
2. Open a PR, get it reviewed, merge to production/live (or staging/live)
3. ArgoCD detects the commit, syncs — applies the change to the cluster
```

No Serenity involved. This is a direct manifest change.

### What a Serenity PR looks like

PR title format:

```
agent-gateway ➮ staging ⦗📦 e120a7af24ac64093939a7c2cbd99f1cb5b2a82868256990da2efb3c08c09801⦘
```

The SHA in the title is the Docker image digest — not the git commit SHA. This is the only thing that identifies the deployment in the PR title.

The diff inside the PR:

```diff
# Image reference updated across all staging clusters
- image: gcr.io/.../agent-gateway-api@sha256:36242989...
+ image: gcr.io/.../agent-gateway-api@sha256:474f6035...

# Restart annotation bumped
- rollout/restartedAt: "2026-03-16T15:22:25Z"
+ rollout/restartedAt: "2026-03-17T09:00:35Z"
```

Note: `serenity sync` creates both the staging PR and the production PR at the same time (from a single `sync_application` CI job). Both PRs show up in `uaa-gitops` as open simultaneously. `serenity approve` merges them independently — staging first, production later.

---

## 10. Reading the ArgoCD UI

ArgoCD lives at: `https://argocd-infra.sso.sre.europe-west1.gcp.commercetools.com`

### Applications list

Search for `agent-gateway` to see all 4 Applications. Key fields:

- **Status**: `Healthy + Synced` = everything is fine. `OutOfSync` = pending change not yet applied.
- **Target Revision**: which branch ArgoCD is watching (`production/live`, `staging/live`)
- **Last Sync**: when it last applied a change, and which commit it synced to
- **Author**: if it says `ct-serenity-prod[bot]`, Serenity triggered this deploy

### Application detail view

The tree view shows all Kubernetes resources managed by this Application:

```
ConfigMap (vault-agent config)
Service (load balancer)
ServiceAccount
  └── Endpoint
      └── EndpointSlice
Deployment
  └── ReplicaSet (current)     ← "an hour ago, rev:5"
      └── Pod (running 1/1)
  └── ReplicaSet (previous)    ← "19 hours ago, rev:4" — kept for rollback
  └── ReplicaSet (older)       ← "21 hours ago, rev:3"
```

Multiple ReplicaSets visible = ArgoCD keeps previous versions for instant rollback.

### Sync Status panel

```
Synced to production/live (22cf15a)
Auto sync is enabled.
Author: ct-serenity-prod[bot]
Comment: Merge pull request #161 from commercetools/produ...
```

This tells you exactly which Git commit is deployed, who triggered it, and the PR that caused it.

### Pod detail

Click any pod in the tree to see:

- **Images**: exact SHA digest of what's running
- **State**: Running / Pending / Terminated
- **Container State**: per-container (init containers like vault-agent-init show as Completed)
- **Logs tab**: tail logs directly from the ArgoCD UI

### Buttons

| Button                   | What it does                                                                                                                                                                             |
| ------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Sync**                 | Manually trigger a sync (force ArgoCD to apply what's in Git now)                                                                                                                        |
| **Refresh**              | Re-check Git for changes (ArgoCD polls every few minutes; this forces immediate check)                                                                                                   |
| **History and Rollback** | See previous syncs and what commit each deployed. With auto-sync enabled, any rollback via the UI gets immediately overridden — real rollback means reverting the commit in `uaa-gitops` |
| **Diff**                 | See exactly what changed between current cluster state and Git                                                                                                                           |

### ArgoCD PreSync hooks — mcp-config-service DB migrations

The `mcp-config-service` manifest includes an ArgoCD **PreSync hook** — a Kubernetes `Job` that ArgoCD runs automatically before applying any other resources on each sync:

```yaml
kind: Job
annotations:
  argocd.argoproj.io/hook: PreSync
  argocd.argoproj.io/hook-delete-policy: BeforeHookCreation
spec:
  backoffLimit: 3 # retry up to 3 times
  activeDeadlineSeconds: 300 # abort if not done in 5 minutes
```

**What this means in practice:**

1. ArgoCD detects a change in `staging/live` or `production/live`
2. Before touching the Deployment, it creates this migration Job
3. The Job runs `./migrate.sh` inside the same Docker image being deployed (so the migration code always matches the app code)
4. If the Job succeeds → ArgoCD proceeds to sync the Deployment
5. If the Job fails (after 3 retries or 5 min timeout) → ArgoCD aborts the sync, the Deployment is **never updated**, the old version keeps running

`BeforeHookCreation` means ArgoCD deletes the previous Job before creating a new one, so you can re-run migrations on re-deploy without `AlreadyExists` errors. The previous run's logs stay visible in ArgoCD until the next sync kicks off.

This is a safe-deploy pattern: the database schema migration is always guaranteed to succeed before the new app version starts serving traffic.

---

## 11. Quick Reference — As a Developer on This Team

### You need to deploy a code change

→ Merge your PR to `agent-gateway-services`. Serenity handles the rest. Check ArgoCD UI after ~5 minutes.

### You need to add/change a Kubernetes resource (HPA, ConfigMap, resource limits)

→ Edit the manifest directly in `uaa-gitops`, open a PR, merge it. ArgoCD syncs automatically.

### How to check if your deploy went through

1. Go to the ArgoCD UI
2. Search for `agent-gateway`
3. Check `Last Sync` — does it show your Serenity PR commit? Is status `Healthy + Synced`?
4. Click into a pod → check the image SHA matches what was built

### Something looks wrong after a deploy — how to roll back

With auto-sync enabled, clicking "Rollback" in the ArgoCD UI does nothing lasting — ArgoCD immediately re-syncs to the latest Git commit. The only real rollback is reverting the Serenity merge commit in `uaa-gitops`:

```bash
# in uaa-gitops
git revert <serenity-merge-commit-sha>
git push  # ArgoCD auto-syncs to the reverted state
```

If you need to cancel a deployment that's been synced but not yet approved for production: the production PR exists in `uaa-gitops` as an open PR (created by `serenity sync`). You can close it before running `serenity approve`, and production will never be updated. The staging deployment has already happened at this point.

The **History and Rollback** button is still useful for reading what was deployed at each sync point.

### Why is one cluster OutOfSync?

→ A Serenity PR may have only targeted some environments, or auto-sync is temporarily paused. Click **Sync** to manually apply the current Git state.

### Vault agent sidecar

You'll see `vault-agent-init` as an init container in every pod (terminated, exit code 0 = success). This is normal — it runs at startup to fetch secrets from Vault and inject them as environment variables, then exits. The main `agent-gateway-api` container is the one that matters.

### The Argo Rollouts migration (upcoming ticket)

Currently agent-gateway uses `kind: Deployment`. The migration to `kind: Rollout` means:

- Change the manifest kind and add a `strategy` block in `uaa-gitops`
- Serenity's flow doesn't change
- The `rollout/restartedAt` annotation trick continues to work
- The difference is what happens _inside the cluster_ during a deploy — you get canary/blue-green instead of a plain rolling update
