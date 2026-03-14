# CI/CD

> **21 questions** — 10 theory, 7 practical, 4 experience

- CI vs CD (delivery vs deployment): why they're separate concerns
- Pipeline design end to end: git push to production traffic — stage ordering and rationale
- Static analysis and quality gates: linting, type checking, SAST, dependency vulnerability scanning (Snyk, Dependabot), fail-fast ordering in pipelines
- Build-release-run separation (12-factor app)
- Branching strategies: trunk-based development vs long-lived feature branches — CI tradeoffs
- Deployment strategies: rolling, blue-green, canary — infrastructure requirements, rollback speed, blast radius
- Feature flags in deployment: decoupling deploy from release, flag-driven canary rollouts, cleanup discipline, managed services (LaunchDarkly, Unleash) vs homegrown
- Database migrations in pipelines: expand-and-contract, backward compatibility during rollouts
- Environment promotion (dev → staging → production): what should differ vs stay identical
- Secrets in CI/CD: risks of env vars in logs/config, masking, OIDC-based authentication to cloud providers
- Pipeline security: supply chain risks (third-party orbs/actions), pinning versions, least-privilege CI credentials
- CircleCI: orbs, workflows, test splitting, parallelism, caching (save/restore + DLC), approval jobs
- GitHub Actions: workflows, reusable workflows, matrix build, comparison with CircleCI
- Integration testing in CI: service containers, testcontainers, test database state
- Rollback strategies: code-only vs code + migration rollbacks, fix-forward decisions, incident identification, health verification

---

## Foundational

<details>
<summary>1. Why are Continuous Integration, Continuous Delivery, and Continuous Deployment treated as separate concerns — what specific problem does each one solve, what does a team lose when they blur the boundaries (e.g., deploying straight from CI without a delivery gate), and how do they compose into a pipeline that balances speed with safety?</summary>

Each solves a distinct problem in the path from code to production:

**Continuous Integration (CI)** solves the "integration hell" problem. Developers merge to a shared branch frequently (at least daily), and every merge triggers an automated build + test run. The problem it prevents: developers working in isolation for days or weeks, then discovering their changes conflict or break each other. CI catches integration bugs within minutes, not days.

**Continuous Delivery (CD)** solves the "it works on my machine" and "release day panic" problems. Every commit that passes CI produces a deployable artifact that has been validated through staging environments, acceptance tests, and quality gates. The artifact is *always ready* to deploy to production, but a human makes the final decision. The problem it prevents: manual, error-prone release processes where teams spend days preparing a release.

**Continuous Deployment** solves the "deployment bottleneck" problem. Every commit that passes all automated gates deploys to production automatically — no human approval step. The problem it prevents: deployable artifacts sitting in a queue waiting for someone to press a button.

**What goes wrong when you blur boundaries:**

- **Deploying straight from CI** (skipping delivery gates): you push code that compiles and passes unit tests directly to production. No staging validation, no integration tests against real dependencies, no canary phase. A bad database query or misconfigured environment variable goes live immediately.
- **Treating CD as just "CI with a deploy step"**: you lose the concept of a *validated, promotable artifact*. Teams end up rebuilding for each environment instead of promoting the same artifact, which means what you tested isn't what you deployed.
- **Jumping to Continuous Deployment without Continuous Delivery maturity**: without comprehensive automated tests, monitoring, and rollback capability, every commit becomes a potential outage.

**How they compose:** CI produces a tested artifact, CD promotes it through environments with increasing confidence, and Continuous Deployment makes the final gate automated rather than manual.

</details>

<details>
<summary>2. Walk through the stages of a CI/CD pipeline from git push to production traffic — what happens at each stage (build, test, static analysis, artifact creation, staging deploy, acceptance tests, production deploy, smoke tests), why are the stages ordered this way, and what are the consequences of skipping or reordering stages (e.g., deploying before integration tests, or running linting after a 20-minute test suite)?</summary>

**Stage-by-stage walkthrough:**

**1. Static analysis (seconds)** — Linting (ESLint), type checking (`tsc --noEmit`), formatting checks. These are the cheapest checks — they catch syntax errors, type mismatches, and style violations in seconds with zero infrastructure.

**2. Build (seconds to minutes)** — Compile TypeScript, bundle the application, resolve dependencies. Validates that the code compiles and all imports resolve. Produces the compiled output needed for subsequent stages.

**3. Unit tests (seconds to minutes)** — Fast, isolated tests with no external dependencies. Catch logic bugs, edge cases, and regressions. Run in parallel for speed.

**4. Integration tests (minutes)** — Tests against real dependencies (databases, message queues) using service containers or testcontainers. Catch issues that mocks hide: query syntax errors, serialization bugs, connection handling.

**5. Security scanning (minutes)** — SAST (static application security testing), dependency vulnerability scanning (Snyk, npm audit). Can run in parallel with tests since they're independent.

**6. Artifact creation** — Build the Docker image (or bundle), tag it with the Git SHA, push to a registry. From here forward, the *same artifact* is promoted — never rebuilt.

**7. Staging deploy + acceptance tests** — Deploy the artifact to a staging environment that mirrors production. Run end-to-end tests, smoke tests, or manual QA. Catches environment-specific issues (config, networking, resource constraints).

**8. Production deploy** — Canary or rolling deployment of the validated artifact. May include an approval gate (manual or automated).

**9. Smoke tests + monitoring** — Verify critical paths work (health endpoints, key user flows). Monitor error rates, latency, and business metrics for the first 10-15 minutes.

**Why this order — the fail-fast principle:** Order stages by cost (time + resources), cheapest first. If linting catches a typo in 5 seconds, there's no reason to wait 20 minutes for the test suite to tell you the same thing. Each stage acts as a gate — if it fails, nothing downstream runs.

**Consequences of reordering or skipping:**

- **Linting after tests**: Developers wait 10+ minutes to learn about a missing semicolon. Multiply by every push across a team, and you waste hours of developer time weekly.
- **Deploying before integration tests**: A query that works against mocked data fails against real Postgres. You discover this in staging (best case) or production (worst case).
- **Skipping artifact creation and rebuilding per environment**: The binary you tested isn't the binary you deployed. Subtle differences in build environments, dependency resolution, or timestamps can cause production-only bugs.
- **Skipping smoke tests after deploy**: You have no automated signal that the deploy actually works. You rely on users to report errors.

</details>

<details>
<summary>3. What is the build-release-run separation from the 12-factor app methodology, why does it insist that builds produce immutable artifacts that are strictly separated from configuration, and what real-world problems occur when teams violate this (e.g., building different artifacts per environment, or baking secrets into images)?</summary>

The 12-factor app defines three strictly separate stages:

**Build** — Convert source code into an executable artifact (compile TypeScript, install dependencies, create a Docker image). The output is an immutable, versioned artifact. A build is triggered by a code change.

**Release** — Combine the build artifact with environment-specific configuration (database URLs, API keys, feature flags). A release is a unique combination of build + config, and it's also immutable — you don't patch a release, you create a new one.

**Run** — Execute the release in the target environment. Start the process, manage it, restart on failure. The run stage should be as simple as possible — no building, no config resolution.

**Why immutable artifacts matter:**

The core insight is *what you tested is what you deploy*. If you rebuild for each environment, you can't guarantee the staging build and production build are identical. Different dependency resolution, different build tools versions, different build-time variables — any of these can introduce subtle differences.

**Real-world violations and their consequences:**

- **Building different artifacts per environment**: A team uses `npm install` during deployment to each environment. Staging gets `lodash@4.17.20`, production gets `4.17.21` because a patch was published between deploys. A behavioral change in the patch causes a production bug that never appeared in staging.

- **Baking secrets into images**: `docker build --build-arg DB_PASSWORD=secret .` embeds the secret in a Docker layer. Anyone with access to the image registry can extract it. The image can't be promoted to another environment without rebuilding (with different secrets), which breaks immutability.

- **Config in the build**: A team hardcodes `API_URL=https://staging.api.com` at build time. To deploy to production, they must rebuild with `API_URL=https://api.com`. Now the production artifact was never tested in staging.

**The correct pattern:**

```dockerfile
# Build stage — no config, no secrets
FROM node:20-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Run stage — config comes from environment
FROM node:20-alpine
WORKDIR /app
COPY --from=build /app/dist ./dist
COPY --from=build /app/node_modules ./node_modules
# No ENV for secrets — those come from K8s secrets, Vault, etc. at runtime
CMD ["node", "dist/main.js"]
```

The image is built once, pushed to a registry, and promoted through environments. Each environment injects its own configuration via environment variables, config maps, or secret management systems.

</details>

## Conceptual Depth

<details>
<summary>4. Why does trunk-based development enable CI better than long-lived feature branches — what happens to integration frequency, merge conflict severity, and feedback loop speed as branch lifetime increases, and when might short-lived feature branches (1-2 days) be a reasonable middle ground vs committing directly to trunk?</summary>

**The core problem with long-lived branches:**

CI literally means "continuous integration" — integrating code changes continuously with the shared codebase. A feature branch that lives for two weeks isn't doing CI at all; it's doing "delayed integration." The longer a branch lives, the worse three things get:

**Integration frequency drops.** If 5 developers each work on week-long branches, their code isn't integrating with each other during that week. They're all building on a snapshot of trunk that gets increasingly stale. When they finally merge, they discover their changes interact in unexpected ways.

**Merge conflict severity grows non-linearly.** A 1-day branch might conflict with 2-3 lines. A 2-week branch can conflict with entire files — not just textual conflicts but *semantic* conflicts where both changes compile individually but break together. Example: developer A renames a function, developer B adds new callers of the old function name. No textual merge conflict, but the code is broken.

**Feedback loops slow down.** On trunk-based development, you push a small change and get CI feedback in minutes. On a long-lived branch, you get CI feedback against your branch (which diverges from trunk), then discover entirely new failures when you finally merge to trunk. The bugs found at merge time are the hardest to debug because the diff is huge.

**Trunk-based development solves this** by making every commit integrate with trunk immediately. Small, frequent commits mean small diffs, easy reviews, fast feedback, and trivial merge conflicts.

**Short-lived feature branches (1-2 days) as a middle ground:**

This is what most teams actually do, and it works well. The key constraints:

- Branch lives no more than 1-2 days
- CI runs on the branch AND requires the branch to be up-to-date with trunk before merging
- Pull requests are small enough to review in one sitting

This gives you the safety of code review via PRs (which committing directly to trunk doesn't) while keeping integration frequency high enough that conflicts stay trivial.

**When to commit directly to trunk:**

- Very senior teams with strong pair programming culture (the pair review replaces the PR)
- When feature flags are mature enough that incomplete features can be merged behind a flag without risk
- When the team has comprehensive automated tests that catch regressions without human review

**When long-lived branches are tempting but still wrong:**

Large features should be broken into small, independently deployable increments behind feature flags — not developed on a branch for weeks. If a feature "can't be broken down," that's usually a sign of insufficient design work, not a valid reason for a long-lived branch.

</details>

<details>
<summary>5. Compare rolling, blue-green, and canary deployment strategies — what infrastructure does each one require, how fast can each roll back, what is the blast radius of a bad deploy for each, and how do you decide which strategy fits a given service (considering factors like database coupling, session state, and traffic volume)?</summary>

| Aspect | Rolling | Blue-Green | Canary |
|---|---|---|---|
| **How it works** | Replace instances incrementally — take one old pod down, bring one new pod up, repeat | Run two identical environments; route all traffic from old (blue) to new (green) at once | Route a small % of traffic to the new version, gradually increase if healthy |
| **Infrastructure required** | Minimal — just enough capacity for N-1 instances during rollout | 2x capacity — full duplicate environment running simultaneously | Same as rolling + traffic splitting (service mesh, ingress controller, or weighted load balancer) |
| **Rollback speed** | Slow-ish — must roll forward with old version or undo incrementally | Instant — switch the router back to blue | Fast — route 100% traffic back to old version |
| **Blast radius** | Progressive — during rollout, a fraction of traffic hits new instances (1/N, then 2/N, etc.) | All-or-nothing — when you switch, 100% of traffic hits the new version at once | Controlled — you choose the % (start at 1-5%, only increase when metrics are healthy) |
| **Mixed versions during deploy** | Yes — old and new instances serve traffic simultaneously | No — clean cutover (unless database schema changed) | Yes — old and new serve traffic simultaneously |

**Decision factors:**

**Database coupling** is the critical factor. Both rolling and canary deployments run old and new code simultaneously, meaning the database schema must be compatible with both versions. Blue-green avoids this during the switchover itself, but you still need backward-compatible migrations if you want the ability to switch back to blue.

**Session state**: If sessions are pinned to specific instances (sticky sessions), rolling deployments can terminate active sessions as instances are replaced. Blue-green has the same problem at cutover. The real fix is externalizing session state (Redis, database), which makes all three strategies work cleanly.

**Traffic volume and risk tolerance**: High-traffic, revenue-critical services benefit most from canary — you validate with real traffic at minimal risk. Lower-traffic internal services might not generate enough canary traffic for statistically meaningful signals, making blue-green or rolling simpler and sufficient.

**Practical recommendations:**

- **Default choice for Kubernetes workloads**: Rolling deployment — it's built into `kubectl` with `maxUnavailable` and `maxSurge`, requires no extra infrastructure, and handles most cases well.
- **High-risk deploys or critical services**: Canary with automated metric analysis (Argo Rollouts, Flagger). Catches performance regressions and error rate spikes before they hit all users.
- **Simple services with fast rollback needs**: Blue-green when you can afford 2x capacity and want instant, clean rollbacks. Common for serverless or PaaS deployments where the platform handles the duplicate environment.

</details>

<details>
<summary>6. Why are database migrations one of the hardest problems in CI/CD — what breaks when you deploy new code that expects a new schema before the migration runs (or vice versa), how does the expand-and-contract pattern solve backward compatibility during rolling deployments, and when is a migration too dangerous to run automatically in a pipeline?</summary>

**Why it's hard:**

Database schema changes are the one part of a deployment that isn't easily reversible. You can roll back code in seconds, but you can't un-drop a column or un-delete data. And during any deployment strategy that runs old and new code simultaneously (rolling, canary), both versions must work against the *same* database.

**The timing problem:**

- **Migration runs before new code deploys**: Old code is still running and queries a column that was just renamed or dropped. Old code crashes. Example: migration renames `user_name` to `username`, but old pods still query `SELECT user_name FROM users`.
- **New code deploys before migration runs**: New code queries a column that doesn't exist yet. New code crashes. Example: new code queries `SELECT username FROM users`, but the migration hasn't run to rename the column.

Either way, there's a window where the running code and the schema are incompatible.

**Expand-and-contract (aka parallel change) solves this:**

The pattern splits a breaking schema change into multiple safe steps deployed separately:

**Phase 1 — Expand**: Add the new structure alongside the old. Both old and new code work.
```sql
-- Migration 1: Add new column (non-breaking)
ALTER TABLE users ADD COLUMN username VARCHAR(255);

-- Backfill existing data
UPDATE users SET username = user_name WHERE username IS NULL;
```
Deploy code that writes to *both* columns but reads from `user_name`. Old code still works fine.

**Phase 2 — Migrate**: Deploy code that reads from `username` and writes to both. Old and new code coexist safely because both columns stay in sync.

**Phase 3 — Contract**: Once all instances run the new code, deploy a migration that drops the old column.
```sql
-- Migration 3: Remove old column (only after all code uses new column)
ALTER TABLE users DROP COLUMN user_name;
```

This is three deployments for what seems like a simple rename. That's why database migrations are hard — the safe path is tedious.

**When NOT to automate migrations in a pipeline:**

- **Large table operations**: `ALTER TABLE` on a table with hundreds of millions of rows can lock the table for minutes or hours. Use online schema migration tools (gh-ost, pt-online-schema-change) instead.
- **Data migrations that touch every row**: Backfilling a column on a 500M-row table saturates I/O and impacts production traffic. Run these as background jobs with rate limiting, not as a pipeline step.
- **Destructive operations**: Dropping tables/columns should require human verification that the expand-and-contract cycle is complete.
- **Migrations with ambiguous rollback**: If the migration can't be reversed (data transformation that loses information), it needs manual review and a specific rollback plan before running.

</details>

<details>
<summary>7. When promoting artifacts through environments (dev → staging → production), what should stay identical across environments vs what should differ — why must the artifact (Docker image, compiled binary) be the same, what configuration should change (database URLs, feature flags, resource limits), and what common mistakes cause "works in staging, breaks in production" situations?</summary>

**Must stay identical (the artifact):**

- Docker image / compiled binary — the exact same image SHA promoted through every environment
- Application code, dependencies, runtime version — all baked into the artifact at build time
- Startup scripts, health check endpoints, graceful shutdown logic

The reason is simple: if you rebuild for production, you're deploying something you never tested. Even identical source code can produce different artifacts due to non-deterministic dependency resolution, different build tool versions, or transient network issues during `npm install`.

**Should differ (environment-specific configuration):**

- Connection strings (database URLs, Redis hosts, message broker endpoints)
- Secrets and API keys (injected via K8s Secrets, Vault, or cloud secret managers)
- Feature flag values (flags enabled in staging for testing, disabled in production until ready)
- Resource limits (CPU/memory allocations — staging often runs with less capacity)
- Replica counts and scaling policies
- Log levels (debug in dev, info/warn in production)
- Domain names, TLS certificates, ingress rules

These come from environment variables, ConfigMaps, or external configuration services — never baked into the image (as covered in question 3).

**Common "works in staging, breaks in production" mistakes:**

1. **Staging doesn't mirror production infrastructure**: Staging uses a single-node database, production uses a cluster with read replicas. Queries that work on a single node fail on replicas due to replication lag.

2. **Traffic volume differences**: Code that works at staging's 10 req/s falls apart at production's 10,000 req/s. Connection pools exhaust, timeouts cascade, memory leaks that take days to manifest at low traffic appear in hours.

3. **Missing environment variables**: Staging has a config value hardcoded from an earlier debug session. Production expects it from an env var that was never added to the deployment manifest.

4. **Data shape differences**: Staging has clean seed data. Production has 5 years of organic data with nulls, unicode edge cases, and records that violate constraints added after they were created.

**Prevention**: Infrastructure-as-code for environment parity, automated config validation that checks required env vars before startup, and load testing in staging with production-like traffic patterns.

</details>

<details>
<summary>8. What are the real supply chain risks in CI/CD pipelines — how can a compromised third-party orb, GitHub Action, or base image inject malicious code into your production artifacts, what does pinning versions and verifying checksums protect against, and how should you scope CI credentials to enforce least privilege (e.g., deploy-only tokens, namespace-scoped K8s service accounts)?</summary>

**How compromised dependencies inject malicious code:**

**Third-party GitHub Actions / CircleCI Orbs**: Actions run with the same permissions as your workflow. A compromised action can:
- Read all secrets available to the workflow (`${{ secrets.DEPLOY_TOKEN }}`)
- Modify source code or build output before the artifact is created
- Exfiltrate secrets to an external server
- Inject a backdoor into your Docker image during the build step

Real example: the `codecov/codecov-action` compromise in 2021 — the action was modified to extract CI environment variables (including secrets) and send them to an attacker-controlled server.

**Base images**: If your Dockerfile starts with `FROM node:20`, you trust that the `node` image on Docker Hub hasn't been tampered with. A compromised base image could contain a modified Node.js binary that intercepts network calls or a cron job that exfiltrates data.

**npm packages**: A malicious postinstall script in a transitive dependency runs during `npm install` in your CI pipeline with full access to the build environment.

**What pinning and checksums protect against:**

```yaml
# Bad — mutable tag, can be overwritten by attacker
- uses: actions/checkout@v4

# Good — pinned to immutable commit SHA
- uses: actions/checkout@b4ffde65f46336ab88eb53be808477a3936bae11 # v4.1.1
```

Pinning to a SHA means that even if the action's `v4` tag is moved to point to a compromised version, your pipeline still uses the known-good commit. Same principle for Docker images — pin to a digest (`node@sha256:abc123...`) rather than a mutable tag.

Checksums (like `npm ci` with `package-lock.json`) ensure that the exact bytes you audited are the exact bytes you install. Without a lockfile, `npm install` might resolve to a newer, compromised version of a transitive dependency.

**Least-privilege CI credentials:**

The principle: CI credentials should have the minimum permissions needed for each specific job, not a god-token shared across the pipeline.

- **Image registry**: CI gets a push-only token scoped to a single repository. It can't pull other teams' images or delete tags.
- **Kubernetes deployment**: The deploy job uses a service account scoped to a single namespace with only `deployments/update` permission — not cluster-admin.
- **Cloud provider**: Use OIDC-based authentication (GitHub Actions OIDC → AWS STS) instead of long-lived access keys. The token is short-lived, scoped to a specific role, and tied to a specific repo/branch.

```yaml
# GitHub Actions OIDC to AWS — no stored secrets
permissions:
  id-token: write
  contents: read

steps:
  - uses: aws-actions/configure-aws-credentials@v4  # pin to SHA in production (simplified here)
    with:
      role-to-assume: arn:aws:iam::123456789:role/ci-deploy-only
      aws-region: eu-west-1
```

- **Separate credentials per stage**: The test job doesn't need deploy credentials. The deploy job doesn't need npm publish credentials. Use job-level permissions, not workflow-level.
- **Never use personal tokens**: If someone leaves the team, their token should not break the pipeline. Use machine users or OIDC.

</details>

<details>
<summary>9. Compare the tradeoffs of rolling back code-only vs rolling back code plus database migrations — when is fix-forward (shipping a new fix instead of reverting) the better choice, how do you identify that a deploy is the cause of an incident vs an unrelated issue, and what health verification steps should run after every rollback to confirm the system is actually recovered?</summary>

**Code-only rollback:**

Simple and fast. Redeploy the previous image tag or run `kubectl rollout undo`. Works cleanly when:
- No database schema changes were part of the deploy
- The previous code version is compatible with the current database state
- No irreversible side effects occurred (emails sent, payments processed)

Typically takes seconds to minutes. This is why you want to keep database migrations separate from code deploys whenever possible.

**Code + migration rollback:**

Much harder. If the migration added a column, rolling back the code means the old code ignores the new column (usually fine). But if the migration dropped or renamed a column, the old code will crash because it expects the old schema. You'd need a reverse migration, which means:
- You wrote and tested a reverse migration in advance (many teams don't)
- The reverse migration doesn't lose data (dropping a newly-added column is safe; un-dropping a deleted column is impossible without backups)
- The reverse migration can run while old code is trying to serve traffic

This is exactly why the expand-and-contract pattern (covered in question 6) exists — it ensures that at every point in the deployment process, the database schema is compatible with both old and new code.

**When fix-forward is the better choice:**

- **The migration already ran and can't be cleanly reversed** — e.g., a data transformation that's not losslessly reversible. Rolling back the code would leave you with code that doesn't understand the new data shape.
- **The bug is small and well-understood** — a typo in a config key, an off-by-one error. Shipping a 2-line fix is faster and less risky than rolling back and potentially triggering other issues.
- **Rollback would cause its own outage** — if the old version has a known incompatibility with the current state (other services have already upgraded their API contracts, for example).
- **The blast radius is small** — the bug affects a non-critical feature behind a flag you can disable while you fix forward.

**Identifying deploy vs coincidental issues:**

1. **Temporal correlation**: Did the error rate spike start within minutes of the deploy completing? Check the deploy timestamp against the metric graphs.
2. **Error signature**: Do the new errors reference code paths that changed in this deploy? Stack traces pointing to modified files are strong evidence.
3. **Canary comparison**: If using canary deployment, compare error rates between canary (new) and stable (old) pods. If only canary pods error, it's the deploy.
4. **Rollback test**: If you roll back and the errors stop, the deploy caused it. If errors persist after rollback, it's something else (upstream dependency, data issue, traffic spike).
5. **Diff review**: Look at the actual code diff. Does it touch the subsystem that's erroring?

**Health verification after rollback:**

1. **Readiness/liveness probes passing** — all pods report healthy
2. **Error rate returns to pre-deploy baseline** — not just lower, but back to normal
3. **Latency returns to baseline** — P50, P95, P99 all within normal range
4. **Key business metrics recover** — order completion rate, login success rate, etc.
5. **No cascading failures** — downstream services that were affected also recover
6. **Log review** — no new error patterns appearing, no connection pool exhaustion or memory pressure
7. **Synthetic monitoring / smoke tests pass** — automated checks of critical user flows succeed

</details>

<details>
<summary>10. Why do teams decouple deployment from release using feature flags -- how does a flag-driven rollout differ from a canary deployment, what is flag cleanup discipline and why does skipping it create long-term technical debt, and when should you use a managed service (LaunchDarkly, Unleash) vs a homegrown feature flag system?</summary>

**Why decouple deployment from release:**

Deployment is putting new code on production servers. Release is making a feature available to users. When these are the same event, every deploy is a feature launch — which means deploys carry business risk and require coordination ("don't deploy on Fridays"). Feature flags break this coupling: you deploy code anytime (it's behind a flag, disabled by default), then enable the feature independently when you're ready.

This enables:
- **Deploying continuously** without worrying about incomplete features reaching users
- **Gradual rollouts** — enable for 1% of users, then 10%, then 50%, then 100%
- **Instant rollback of a feature** without rolling back the deployment (just flip the flag off)
- **Testing in production** — enable the flag for internal users or specific accounts before public release

**Flag-driven rollout vs canary deployment:**

A canary deployment operates at the *infrastructure level* — you route X% of traffic to new pods running new code. You can't control *which* users hit the canary, and the canary runs the entire new codebase, not just one feature.

A flag-driven rollout operates at the *application level* — all pods run the same code, but the flag controls which users see the new feature. You can target specific segments (internal users, beta testers, users in a specific region), and you can roll out multiple features independently on different timelines.

They complement each other: use canary deployment to validate that the new code doesn't crash or degrade performance, then use feature flags to gradually release the business feature to users.

**Flag cleanup discipline:**

Every feature flag is a branch in your code — an `if/else` that makes the code harder to understand, test, and maintain. Without cleanup discipline:

- Flags accumulate. Teams end up with hundreds of flags, many of which are permanently "on" or reference features that launched months ago.
- Code paths multiply. Each flag doubles the number of possible states. 10 boolean flags = 1,024 possible code paths, most of which are never tested.
- Dead code hides behind "off" flags. Nobody remembers what the flag controlled, so nobody dares remove it.
- Testing becomes unreliable. Tests pass with the flag on *or* off, but nobody tests the combination of flag A on + flag B off + flag C on.

**Cleanup discipline means:** Every flag has an owner and an expiration date. When a feature is fully rolled out, the flag is removed (not just set to "on permanently") within a sprint. Some teams enforce this with lint rules that fail if a flag older than N days still exists in the codebase.

**Managed service vs homegrown:**

| | Managed (LaunchDarkly, Unleash) | Homegrown |
|---|---|---|
| **Targeting** | Rich rules: user segments, percentages, A/B, geographic | Usually boolean or simple percentage |
| **Audit trail** | Built-in — who changed what flag when | You build it yourself |
| **SDK support** | Server + client SDKs, streaming updates in real-time | Often polling-based with latency |
| **Cost** | $$$, especially at scale (LaunchDarkly charges per seat/flag) | Engineering time to build and maintain |
| **Flag lifecycle** | Built-in stale flag detection, dashboards | Manual tracking |

**Use managed** when: you have more than a handful of flags, multiple services need flag consistency, non-engineers (PMs) need to control rollouts, or you need audit trails for compliance.

**Use homegrown** when: you need a few simple boolean flags, budget is very tight, or you have specific requirements that managed services don't support. A simple implementation backed by a database or config file works for basic cases. Unleash (open source, self-hosted) is a good middle ground — managed-quality features without the SaaS cost.

</details>

## Practical — Build, Test & Deploy

<details>
<summary>11. How do secrets leak in CI/CD pipelines and how do you prevent it -- show concrete examples of how environment variables end up in logs (debug mode, error stack traces, docker build args), demonstrate how secret masking works in CircleCI and GitHub Actions, and what are the limitations of masking that still leave you vulnerable?</summary>

**How secrets leak — concrete examples:**

**1. Debug mode printing all env vars:**
```typescript
// Someone adds this while debugging a CI failure and forgets to remove it
console.log('Environment:', JSON.stringify(process.env, null, 2));
// Output includes: "DATABASE_URL": "postgres://admin:s3cret@prod-db:5432/app"
```

**2. Error stack traces including config:**
```typescript
// Connection error includes the full connection string in the error message
const client = new Client(process.env.DATABASE_URL);
await client.connect();
// Error: connect ECONNREFUSED postgres://admin:s3cret@prod-db:5432/app
// This error gets logged and appears in CI output
```

**3. Docker build args visible in image history:**
```dockerfile
# NEVER do this — build args are stored in image metadata
ARG DB_PASSWORD
ENV DB_PASSWORD=$DB_PASSWORD
```
```bash
# Anyone with image access can run:
docker history myapp:latest
# Shows: ARG DB_PASSWORD=s3cret
```

**4. Shell command echoing:**
```yaml
# CI step with set -x (debug mode) enabled
- run: |
    set -x
    curl -H "Authorization: Bearer $API_TOKEN" https://api.example.com
    # Output: + curl -H 'Authorization: Bearer sk-live-abc123...' https://api.example.com
```

**5. npm/yarn logs:** Package manager verbose logging can include registry auth tokens in HTTP request headers.

**How secret masking works:**

**CircleCI:** Any value stored as a project or context environment variable is automatically masked in job output. If the value `s3cret` appears in a log line, it's replaced with `****`.

**GitHub Actions:** Secrets defined in `Settings → Secrets` are automatically masked. You can also manually register values for masking:

```yaml
steps:
  - run: |
      TOKEN=$(generate-dynamic-token)
      echo "::add-mask::$TOKEN"  # Register for masking
      echo "Token is $TOKEN"     # Output: Token is ***
```

**Limitations of masking that leave you vulnerable:**

1. **Masking is string-matching, not semantic.** If your secret is `abc123` and the log contains `abc123def`, most maskers won't catch it — the surrounding characters change the string. Base64-encoding a secret produces a different string that won't be masked.

2. **Structured output bypasses masking.** If a secret is `password123` and your code logs it as JSON with each character on a new line, or URL-encodes it as `password%31%32%33`, the masker won't recognize it.

3. **Masking only applies to CI output, not artifacts.** If a secret ends up in a build artifact, test report, or coverage file that gets uploaded, masking doesn't help.

4. **Short secrets may not be masked.** Some platforms skip masking for values shorter than 4 characters to avoid false positives — a secret like `key` might appear unmasked.

5. **Secrets can be exfiltrated without appearing in logs.** A compromised action can `curl` your secrets to an external server. Masking the CI log does nothing to prevent this.

**Real prevention:**

- Use OIDC-based auth instead of stored secrets (as covered in question 8) — no static secrets to leak
- Use secret scanning tools (GitGuardian, GitHub secret scanning) that detect secrets in commits
- Run `docker history` and `docker inspect` in CI to verify no secrets are baked into images
- Restrict `set -x` in CI scripts — use `set +x` before lines that handle secrets
- Never pass secrets as command-line arguments (visible in `/proc` on Linux) — use environment variables or files

</details>

<details>
<summary>12. Configure a CircleCI pipeline for a Node.js monorepo — show how orbs simplify reusable config, how to structure workflows with parallel jobs, implement test splitting across multiple executors to reduce wall-clock time, configure dependency caching (save_cache/restore_cache vs Docker Layer Caching), and add an approval job that gates production deployment behind a manual sign-off.</summary>

```yaml
version: 2.1

# Orbs: reusable packages of config (jobs, commands, executors)
# Instead of writing 30 lines of Node setup, use the orb
orbs:
  node: circleci/node@5.2.0
  docker: circleci/docker@2.6.0
  slack: circleci/slack@4.13.3

executors:
  node-executor:
    docker:
      - image: cimg/node:20.11
    resource_class: medium

jobs:
  lint-and-typecheck:
    executor: node-executor
    steps:
      - checkout
      - node/install-packages  # orb command — handles caching automatically
      - run: npm run lint
      - run: npm run typecheck

  unit-test:
    executor: node-executor
    parallelism: 4  # spin up 4 containers
    steps:
      - checkout
      - node/install-packages
      - run:
          name: Run tests with splitting
          command: |
            # circleci tests splits test files across the 4 containers
            # Uses timing data from previous runs to balance load
            TESTFILES=$(circleci tests glob "packages/*/src/**/*.test.ts" \
              | circleci tests split --split-by=timings)
            npx jest $TESTFILES --ci --reporters=default --reporters=jest-junit
      - store_test_results:
          path: ./junit  # feeds timing data back for smarter splitting
      - store_artifacts:
          path: ./coverage

  integration-test:
    docker:
      - image: cimg/node:20.11
      - image: cimg/postgres:15.4  # service container — available at localhost
        environment:
          POSTGRES_DB: test
          POSTGRES_PASSWORD: test
      - image: cimg/redis:7.2
    steps:
      - checkout
      - node/install-packages
      - run:
          name: Wait for dependencies
          command: dockerize -wait tcp://localhost:5432 -wait tcp://localhost:6379 -timeout 30s
      - run: npm run test:integration

  build-and-push:
    executor: node-executor
    steps:
      - checkout
      - setup_remote_docker:
          docker_layer_caching: true  # DLC: caches Docker layers between builds
      - docker/build:
          image: myorg/myapp
          tag: ${CIRCLE_SHA1}  # resolved at runtime by the shell, not by YAML
      - docker/push:
          image: myorg/myapp
          tag: ${CIRCLE_SHA1}

  deploy-staging:
    executor: node-executor
    steps:
      - checkout
      - run:
          name: Deploy to staging
          command: ./scripts/deploy.sh staging ${CIRCLE_SHA1}

  deploy-production:
    executor: node-executor
    steps:
      - checkout
      - run:
          name: Deploy to production
          command: ./scripts/deploy.sh production ${CIRCLE_SHA1}
      - slack/notify:
          event: pass
          template: basic_success_1

workflows:
  build-test-deploy:
    jobs:
      # Lint and tests run in parallel — no dependency between them
      - lint-and-typecheck
      - unit-test
      - integration-test

      # Build only after all checks pass
      - build-and-push:
          requires:
            - lint-and-typecheck
            - unit-test
            - integration-test

      # Auto-deploy to staging
      - deploy-staging:
          requires:
            - build-and-push

      # Manual approval gate before production
      - hold-for-approval:
          type: approval
          requires:
            - deploy-staging

      # Production deploy only after human approves
      - deploy-production:
          requires:
            - hold-for-approval
```

**Key concepts explained:**

**Orbs** package reusable CI logic. `circleci/node` provides `install-packages` (with built-in caching), executor definitions, and common commands. Without orbs, you'd write 15-20 lines of cache save/restore logic yourself. Orbs are versioned and can be pinned.

**Test splitting with `parallelism: 4`**: CircleCI spins up 4 identical containers. `circleci tests split --split-by=timings` divides test files across containers using historical timing data so each container finishes at roughly the same time. A 20-minute test suite drops to ~5 minutes wall-clock time.

**Caching — `save_cache`/`restore_cache` vs Docker Layer Caching (DLC):**

- **`save_cache`/`restore_cache`**: Caches files on disk between builds (typically `node_modules`). The `node/install-packages` orb command handles this automatically with a checksum of `package-lock.json` as the cache key.
- **DLC**: Caches Docker build layers between builds. If your Dockerfile hasn't changed and the base image is the same, `npm ci` doesn't re-run. DLC is a paid feature and only applies to `setup_remote_docker` — it doesn't affect the job's primary container.

**Approval jobs** create a pause point in the workflow. The pipeline stops at `hold-for-approval` until someone clicks "Approve" in the CircleCI UI. This gates production deployment behind a human decision while keeping everything before it fully automated.

</details>

<details>
<summary>13. Configure a GitHub Actions workflow for the same Node.js application — show a reusable workflow (workflow_call) that can be invoked from multiple repos, a matrix build that tests across Node versions, and how composite actions reduce duplication. Compare the GitHub Actions model (event-driven, marketplace actions) against CircleCI (orbs, resource classes, test splitting) — where does each platform have a genuine advantage?</summary>

**Reusable workflow (called from multiple repos):**

```yaml
# .github/workflows/node-ci.yml in a shared repo (e.g., myorg/ci-workflows)
name: Node.js CI

on:
  workflow_call:  # Makes this callable from other repos
    inputs:
      node-version:
        required: false
        type: string
        default: '20'
      working-directory:
        required: false
        type: string
        default: '.'
    secrets:
      NPM_TOKEN:
        required: false

jobs:
  lint-and-test:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: ${{ inputs.working-directory }}
    steps:
      - uses: actions/checkout@b4ffde65f46336ab88eb53be808477a3936bae11 # v4
      - uses: actions/setup-node@60edb5dd545a775178f52524783378180af0d1f8 # v4
        with:
          node-version: ${{ inputs.node-version }}
          cache: 'npm'
          cache-dependency-path: ${{ inputs.working-directory }}/package-lock.json
      - run: npm ci
      - run: npm run lint
      - run: npm run typecheck
      - run: npm test -- --ci
```

**Calling the reusable workflow from another repo:**

```yaml
# .github/workflows/ci.yml in any consuming repo
name: CI
on: [push, pull_request]

jobs:
  ci:
    uses: myorg/ci-workflows/.github/workflows/node-ci.yml@main
    with:
      node-version: '20'
    secrets:
      NPM_TOKEN: ${{ secrets.NPM_TOKEN }}
```

**Matrix build for testing across Node versions:**

```yaml
jobs:
  test-matrix:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node-version: [18, 20, 22]
      fail-fast: false  # Don't cancel other versions if one fails
    steps:
      - uses: actions/checkout@b4ffde65f46336ab88eb53be808477a3936bae11
      - uses: actions/setup-node@60edb5dd545a775178f52524783378180af0d1f8
        with:
          node-version: ${{ matrix.node-version }}
          cache: 'npm'
      - run: npm ci
      - run: npm test -- --ci
```

**Composite action (reduces duplication within a repo):**

```yaml
# .github/actions/setup-node-project/action.yml
name: Setup Node Project
description: Checkout, install Node, install deps
inputs:
  node-version:
    default: '20'

runs:
  using: composite
  steps:
    - uses: actions/checkout@b4ffde65f46336ab88eb53be808477a3936bae11
    - uses: actions/setup-node@60edb5dd545a775178f52524783378180af0d1f8
      with:
        node-version: ${{ inputs.node-version }}
        cache: 'npm'
    - run: npm ci
      shell: bash
```

Used in workflows as:
```yaml
- uses: ./.github/actions/setup-node-project
  with:
    node-version: '20'
```

**GitHub Actions vs CircleCI — honest comparison:**

| Aspect | GitHub Actions advantage | CircleCI advantage |
|---|---|---|
| **Ecosystem** | Marketplace has thousands of actions; deep GitHub integration (PR checks, issue automation, release triggers) | Orbs are more curated and stable; less "action supply chain" risk |
| **Event model** | Rich event triggers — `issue_comment`, `release`, `schedule`, `workflow_dispatch` — natural for GitHub-centric workflows | Simpler model: push/PR/schedule/API. Less flexible but less complex |
| **Test splitting** | No native equivalent — must DIY or use third-party actions | Built-in `circleci tests split --split-by=timings` with automatic timing data collection. Genuine differentiator for large test suites |
| **Resource classes** | Limited to GitHub-hosted runner sizes or self-hosted | Fine-grained resource classes (small/medium/large/xlarge) with predictable performance and pricing |
| **Caching** | `actions/cache` works but is more verbose; limited cache size | Deeply integrated caching + DLC for Docker builds |
| **Parallelism** | Matrix builds run separate jobs — good for multi-version testing, clunky for splitting a single test suite | Native parallelism with test splitting — designed for exactly this use case |
| **Cost for open source** | Free for public repos — massive advantage | Free tier exists but more limited |
| **Self-hosted runners** | Easy to set up, scales with Kubernetes (actions-runner-controller) | Machine runner setup is more involved |

**Bottom line:** GitHub Actions wins on ecosystem integration, event model, and cost for open-source. CircleCI wins on test performance optimization (splitting, resource classes, DLC). For most teams already on GitHub, Actions is the simpler choice. Teams with large test suites and performance-sensitive CI should evaluate CircleCI's test splitting seriously.

</details>

<details>
<summary>14. Configure a CI pipeline quality gate that runs linting, type checking, SAST scanning, and dependency vulnerability checking (using tools like Snyk or Dependabot) -- show the pipeline YAML, explain why these steps should be ordered to fail fast (cheapest/fastest checks first), and what happens when teams skip SAST or dependency scanning and only discover vulnerabilities in production?</summary>

```yaml
# GitHub Actions quality gate pipeline
name: Quality Gate
on: [push, pull_request]

jobs:
  # Fast checks first — fail in seconds, not minutes
  lint-and-types:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@b4ffde65f46336ab88eb53be808477a3936bae11
      - uses: actions/setup-node@60edb5dd545a775178f52524783378180af0d1f8
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - run: npm run lint          # ~5-15 seconds
      - run: npx tsc --noEmit      # ~10-30 seconds

  # Security scans — can run in parallel with tests
  dependency-audit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@b4ffde65f46336ab88eb53be808477a3936bae11
      - uses: actions/setup-node@60edb5dd545a775178f52524783378180af0d1f8
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci

      # npm audit — catches known vulnerabilities in dependencies
      - run: npm audit --audit-level=high  # Fail on high/critical only

      # Snyk — deeper analysis with reachability (are you actually calling the vulnerable code?)
      - uses: snyk/actions/node@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
        with:
          args: --severity-threshold=high

  sast-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@b4ffde65f46336ab88eb53be808477a3936bae11

      # CodeQL — GitHub's SAST tool, free for public repos
      - uses: github/codeql-action/init@v3
        with:
          languages: javascript-typescript
      - uses: github/codeql-action/analyze@v3

  # Tests only run if lint/types pass
  test:
    needs: lint-and-types
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@b4ffde65f46336ab88eb53be808477a3936bae11
      - uses: actions/setup-node@60edb5dd545a775178f52524783378180af0d1f8
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - run: npm test -- --ci
```

**Why this ordering matters (fail-fast economics):**

| Check | Time | What it catches | Cost if delayed |
|---|---|---|---|
| Linting | 5-15s | Style issues, import errors, unused vars | Devs wait for test suite to learn about a missing import |
| Type checking | 10-30s | Type mismatches, missing properties | Runtime crashes that could've been caught statically |
| Dependency audit | 30-60s | Known CVEs in dependencies | Deploying code with a publicly exploitable vulnerability |
| SAST | 2-5min | SQL injection, XSS, path traversal, hardcoded secrets | Security vulnerabilities discovered by attackers instead of your pipeline |
| Tests | 5-20min | Logic bugs, regressions | Self-explanatory |

The lint and type check jobs fail fastest and catch the most common issues (syntax mistakes, type errors). If those fail, there's no reason to wait for the 20-minute test suite. Security scans run in parallel with tests because they're independent — a vulnerability finding doesn't block test execution and vice versa.

**What happens when you skip SAST and dependency scanning:**

**Dependency vulnerabilities:** The `event-stream` incident (2018) injected malicious code into a popular npm package to steal cryptocurrency. Teams without dependency scanning had compromised code in production for weeks. `npm audit` would have flagged it immediately.

**SAST findings:** Without SAST, SQL injection and XSS vulnerabilities only surface during manual code review (if someone catches it) or penetration testing (if you do it). Common real-world examples:
- String concatenation in SQL queries instead of parameterized queries — caught by SAST in seconds, potentially missed in code review
- Unsanitized user input rendered in HTML — caught by SAST rules for XSS patterns

**The compounding risk:** Vulnerabilities found in production require emergency patches, incident response, customer notification (if data was exposed), and post-mortems. The cost is orders of magnitude higher than running a 2-minute scan in CI. Dependency scanning and SAST are the cheapest security investment a team can make.

</details>

<details>
<summary>15. Configure a CI pipeline that runs integration tests against real dependencies — show the config for spinning up service containers (PostgreSQL, Redis) alongside your test runner in both CircleCI and GitHub Actions, explain how testcontainers offers an alternative approach, and how do you handle test database state (migrations, seeding, cleanup between tests) so tests are reliable and isolated?</summary>

**CircleCI — service containers:**

```yaml
jobs:
  integration-test:
    docker:
      # Primary container — your test runner
      - image: cimg/node:20.11
      # Service containers — available at localhost
      - image: cimg/postgres:15.4
        environment:
          POSTGRES_DB: test_db
          POSTGRES_USER: test_user
          POSTGRES_PASSWORD: test_pass
      - image: cimg/redis:7.2
    steps:
      - checkout
      - node/install-packages
      - run:
          name: Wait for services
          command: dockerize -wait tcp://localhost:5432 -wait tcp://localhost:6379 -timeout 30s
      - run:
          name: Run migrations
          command: npm run db:migrate
          environment:
            DATABASE_URL: postgres://test_user:test_pass@localhost:5432/test_db
      - run:
          name: Integration tests
          command: npm run test:integration
          environment:
            DATABASE_URL: postgres://test_user:test_pass@localhost:5432/test_db
            REDIS_URL: redis://localhost:6379
```

**GitHub Actions — service containers:**

```yaml
jobs:
  integration-test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_DB: test_db
          POSTGRES_USER: test_user
          POSTGRES_PASSWORD: test_pass
        ports:
          - 5432:5432
        # Health check ensures Postgres is ready before tests start
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
      redis:
        image: redis:7
        ports:
          - 6379:6379
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - uses: actions/checkout@b4ffde65f46336ab88eb53be808477a3936bae11
      - uses: actions/setup-node@60edb5dd545a775178f52524783378180af0d1f8
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - run: npm run db:migrate
        env:
          DATABASE_URL: postgres://test_user:test_pass@localhost:5432/test_db
      - run: npm run test:integration
        env:
          DATABASE_URL: postgres://test_user:test_pass@localhost:5432/test_db
          REDIS_URL: redis://localhost:6379
```

**Testcontainers — an alternative approach:**

Instead of configuring service containers in CI YAML, testcontainers spins up Docker containers programmatically from within your test code. The test itself defines what it needs:

```typescript
import { PostgreSqlContainer } from '@testcontainers/postgresql';
import { RedisContainer } from '@testcontainers/redis';

describe('Order Service', () => {
  let pgContainer: StartedPostgreSqlContainer;
  let redisContainer: StartedRedisContainer;

  beforeAll(async () => {
    // Containers start from scratch for each test suite
    pgContainer = await new PostgreSqlContainer('postgres:15')
      .withDatabase('test_db')
      .start();

    redisContainer = await new RedisContainer('redis:7').start();

    // Run migrations against the fresh container
    await runMigrations(pgContainer.getConnectionUri());
  }, 60_000); // longer timeout for container startup

  afterAll(async () => {
    await pgContainer.stop();
    await redisContainer.stop();
  });

  it('should create an order', async () => {
    const service = new OrderService({
      databaseUrl: pgContainer.getConnectionUri(),
      redisUrl: redisContainer.getConnectionUrl(),
    });
    // Test against real Postgres and Redis
  });
});
```

**Testcontainers advantages:**
- Tests define their own infrastructure — no CI-specific config needed. Same test runs locally and in CI.
- Each test suite gets a fresh container — no shared state between test files.
- Version pinning happens in test code, not CI config.

**Testcontainers disadvantages:**
- Slower startup — each suite pays the container boot cost (5-15 seconds per container).
- Requires Docker socket access in CI (Docker-in-Docker or mounting the host socket).
- Resource-heavy when many test suites each spin up their own containers.

**Handling test database state:**

**1. Migrations before tests:** Run the full migration suite once before the test run starts. This ensures the schema is current.

**2. Transaction rollback per test (preferred for speed):**
```typescript
beforeEach(async () => {
  await db.query('BEGIN');
});

afterEach(async () => {
  await db.query('ROLLBACK'); // Undo all changes — fast, no cleanup needed
});
```
Each test runs in a transaction that's rolled back afterward. The database returns to its pre-test state instantly. Limitation: doesn't work if the code under test manages its own transactions.

**3. Truncate between tests (when transactions won't work):**
```typescript
afterEach(async () => {
  await db.query('TRUNCATE orders, users, products RESTART IDENTITY CASCADE');
});
```
Slower than rollback but handles cases where tests commit transactions.

**4. Seeding:** Use factory functions that create only the data each test needs, rather than a shared seed file. Shared seed data creates implicit coupling between tests — changing the seed can break unrelated tests.

```typescript
// Factory approach — each test creates exactly what it needs
const user = await createUser({ email: 'test@example.com' });
const order = await createOrder({ userId: user.id, total: 100 });
```

</details>

## Practical — Debugging & Troubleshooting

<details>
<summary>16. You discover that a third-party GitHub Action your pipeline depends on was compromised in a recent version update — walk through the immediate response (how you detect this, what you check for in your recent builds), the remediation steps (pinning to a known-good SHA, auditing your artifacts and deployed services), and the long-term prevention strategy (fork-and-mirror, action allowlists, Dependabot for action versions)?</summary>

**Immediate response (first 30 minutes):**

**How you detect it:** Security advisories (GitHub Security Advisories, Twitter/social media reports, CVE databases), unusual CI behavior (builds suddenly slower, unexpected network calls in logs), or an alert from a supply chain security tool (StepSecurity, Socket.dev).

**What to check immediately:**

1. **Identify the blast radius** — which repos use this action and which versions?
```bash
# Search across all org repos for the compromised action
gh search code "uses: compromised-org/action" --owner=myorg
```

2. **Check recent build logs** — look for the compromised version in workflow runs. Did any builds use the malicious version?

3. **Check what secrets were exposed** — look at the workflow files that use this action. What secrets are available in those jobs? Those secrets are now compromised.

4. **Disable the action immediately** — push a commit to every affected repo that either removes the action or pins to a known-good SHA.

**Remediation steps:**

1. **Rotate all exposed secrets** — every secret that was available to workflows using the compromised action. This includes:
   - Deploy tokens and API keys
   - NPM tokens
   - Cloud provider credentials (or verify OIDC tokens are short-lived enough)
   - Docker registry credentials

2. **Pin to a known-good SHA:**
```yaml
# Replace the tag reference with a verified commit SHA
# Before (vulnerable):
- uses: compromised-org/action@v2

# After (pinned to last known-good commit):
- uses: compromised-org/action@a1b2c3d4e5f6  # v2.0.3 — verified safe
```

3. **Audit deployed artifacts** — check if any builds during the compromised window produced artifacts that were deployed. If so:
   - Rebuild those artifacts from source without the compromised action
   - Redeploy the clean artifacts
   - Check production logs for unusual outbound network calls

4. **Check for persistence** — did the compromised action modify source code in any commits? Check for unexpected changes in PRs that were merged during the window.

**Long-term prevention strategy:**

**1. Fork and mirror critical actions:**
```yaml
# Instead of using the upstream action directly:
- uses: myorg/fork-of-checkout@pinned-sha
```
Fork the action into your org, audit the code, and only update when you've reviewed the diff. This insulates you from upstream compromises entirely.

**2. Action allowlists (GitHub organization settings):**
Configure the org to only allow actions from verified creators, specific repos, or your own org:
- Settings → Actions → General → "Allow select actions and reusable workflows"
- Whitelist only `actions/*`, `github/*`, and your org's own actions

**3. Dependabot for action versions:**
```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: github-actions
    directory: /
    schedule:
      interval: weekly
    # Dependabot will PR action version bumps for review
```
This creates PRs when actions have updates, giving you a review step before adopting new versions.

**4. SHA pinning as policy:** Require all actions to be pinned to SHAs, not tags. Enforce with a linter like `actionlint` or `pin-github-action`:
```bash
# Automated SHA pinning
npx pin-github-action .github/workflows/*.yml
```

**5. Minimal permissions on workflows:**
```yaml
# Restrict the GITHUB_TOKEN to read-only by default
permissions:
  contents: read
# Only grant write permissions to specific jobs that need them
```
Even if an action is compromised, it can only do what the token allows.

</details>

<details>
<summary>17. A production deploy goes out at 3pm, and by 3:15pm monitoring shows a spike in 500 errors — walk through the decision process: how do you confirm the deploy caused it vs a coincidental issue, what signals tell you to roll back immediately vs investigate first, show the exact commands to roll back (kubectl rollout undo, reverting the Git commit for GitOps, or re-deploying the previous image tag), and what health checks must pass before you declare the rollback successful?</summary>

**Step 1: Confirm the deploy caused it (2-3 minutes)**

Don't assume — correlate. Check these signals:

1. **Timing**: Open your monitoring dashboard (Grafana, Datadog). Does the error spike start at exactly the deploy completion time or during the rollout? If errors started 5 minutes *before* the deploy, it's probably not the deploy.

2. **Error source**: Filter errors by pod/instance. If only newly-deployed pods (the ones running the new image) are producing 500s while old pods are healthy, it's the deploy. In a rolling deployment, this is visible before the rollout completes.

3. **Error content**: Check the error logs. Do stack traces reference code paths that changed in this deploy? A `TypeError: Cannot read property 'id' of undefined` in a file you just modified is strong evidence.

4. **Scope**: Is the error rate proportional to the rollout progress? If 30% of pods are updated and error rate increased by ~30%, that's the deploy.

**Step 2: Roll back or investigate? (30-second decision)**

**Roll back immediately if:**
- Error rate is high (>1% of requests failing) and climbing
- The errors affect a critical path (checkout, authentication, payments)
- Users are impacted right now — latency is spiked, pages are broken
- You don't understand the root cause within 2 minutes

**Investigate first if:**
- Error rate is low and stable (a handful of errors, not growing)
- Errors are isolated to a non-critical endpoint
- You can see the root cause in logs and a fix is trivial (config typo, missing env var)
- A feature flag can disable the broken functionality without rolling back

**The default should be: roll back first, investigate later.** The cost of a 5-minute outage while you investigate usually exceeds the cost of rolling back and re-deploying after you've fixed the issue.

**Step 3: Execute the rollback**

**Kubernetes — direct rollback:**
```bash
# Undo the last deployment rollout
kubectl rollout undo deployment/myapp -n production

# Or roll back to a specific revision
kubectl rollout history deployment/myapp -n production
kubectl rollout undo deployment/myapp -n production --to-revision=42

# Watch the rollback progress
kubectl rollout status deployment/myapp -n production
```

**GitOps (ArgoCD/Flux) — revert the Git commit:**
```bash
# Find the deploy commit
git log --oneline -5

# Revert it (creates a new commit, preserves history)
git revert abc1234 --no-edit
git push origin main

# ArgoCD will detect the change and sync automatically
# Or force sync immediately:
argocd app sync myapp
```

**Re-deploy the previous image tag (CI-driven):**
```bash
# If your deploy is triggered by CI, re-run the previous successful deploy
# Or update the deployment manifest directly:
kubectl set image deployment/myapp \
  myapp=myorg/myapp:previous-sha \
  -n production
```

**Step 4: Verify the rollback succeeded**

Run through these checks systematically — don't declare recovery until all pass:

1. **All pods healthy:**
```bash
kubectl get pods -n production -l app=myapp
# All pods should be Running with the old image tag
kubectl describe deployment myapp -n production | grep Image
# Confirm the image SHA matches the previous version
```

2. **Error rate returned to baseline:** Check your dashboard — 500 error rate should drop to pre-deploy levels within minutes of the rollback completing.

3. **Latency returned to baseline:** P50, P95, P99 should all be back to normal. An elevated P99 after rollback might indicate connection pool exhaustion or cache invalidation still unwinding.

4. **Smoke tests pass:** Run automated health checks against production:
```bash
curl -f https://api.myapp.com/health
# Run a synthetic transaction if you have one
npm run smoke-test:production
```

5. **Downstream services healthy:** If your service is a dependency for others, check that their error rates also returned to normal.

6. **No lingering effects:** Check for connection pool exhaustion, queue backlogs, or cache corruption that might persist after rollback. These can cause a "slow recovery" where the rollback succeeds but performance doesn't fully return for 10-15 minutes.

**Post-rollback:** Write a brief incident timeline, create a ticket for root cause investigation, and fix the issue on a branch where you can test it properly before the next deploy attempt.

</details>

---

## Experience-Based Questions

These questions test real-world experience. Prepare by mapping them to your own projects and situations.

<details>
<summary>18. Tell me about a time you designed or significantly improved a CI/CD pipeline — what was the pipeline's state before, what changes did you make, what tradeoffs did you navigate (speed vs safety, complexity vs maintainability), and what was the measurable impact on deployment frequency or developer experience?</summary>

**What the interviewer is looking for:**
- Can you identify pipeline bottlenecks and prioritize improvements?
- Do you think about developer experience, not just technical correctness?
- Can you navigate tradeoffs (speed vs safety, complexity vs maintenance)?
- Do you measure the impact of your changes, not just "it felt better"?

**Suggested structure (STAR-ish, adapted for technical depth):**

1. **Context** (30 seconds): What was the pipeline? What team/product? Why did it need improvement?
2. **Problem** (30 seconds): What was the pain? Be specific — "builds took 25 minutes" or "we deployed once a week because the pipeline was fragile."
3. **Changes you made** (2 minutes): Walk through the specific improvements. Show technical depth: caching strategies, parallelization, test splitting, security additions. Explain *why* you chose each change over alternatives.
4. **Tradeoffs** (1 minute): What did you consider but reject? What was the cost of your approach?
5. **Impact** (30 seconds): Measurable results — build time reduction, deployment frequency increase, developer satisfaction.

**Example outline to personalize:**

- **Before**: Monorepo with a single serial CI pipeline. Lint → build → all tests → Docker build → deploy. Total: 22 minutes. No caching. Tests ran on every push even if only docs changed.
- **Changes**: (1) Added path filtering so docs-only PRs skip tests. (2) Parallelized lint, type-check, and unit tests. (3) Added dependency caching (cut `npm ci` from 90s to 8s). (4) Implemented test splitting across 4 containers. (5) Added Docker layer caching for the image build step.
- **Tradeoff navigated**: Considered splitting into per-package pipelines for the monorepo but chose a single pipeline with path filters. Per-package would be faster but harder to maintain — every new package needs CI config. Path filters were simpler and covered 80% of the benefit.
- **Impact**: Pipeline dropped from 22 minutes to 7 minutes. Team went from deploying 2x/week to daily. Fewer "I'm waiting for CI" Slack messages.

**Key points to hit:**
- Show you understand the *cost* of slow CI (developer idle time, batched deploys, less frequent feedback)
- Demonstrate you measured before and after, not just "it felt faster"
- If you added safety gates (security scanning, approval steps), explain why the added time was worth it
- Mention if you documented the pipeline or made it maintainable for the team

</details>

<details>
<summary>19. Describe a time a deployment went wrong in production and you had to decide between rolling back and fixing forward — what were the symptoms, how did you diagnose the root cause, what decision did you make and why, and what process changes did you implement afterward to prevent a recurrence?</summary>

**What the interviewer is looking for:**
- Incident response instincts — do you prioritize mitigation over root-cause analysis during an active outage?
- Decision-making under pressure — can you explain *why* you chose rollback vs fix-forward?
- Post-incident learning — did you improve the system, not just fix the bug?
- Ownership and communication — did you keep stakeholders informed?

**Suggested structure:**

1. **The deploy and the symptoms** (30 seconds): What went out? What did monitoring show? How fast did you notice?
2. **Diagnosis** (1 minute): How did you confirm the deploy caused it? What tools did you use (logs, metrics, traces)? Walk through the debugging process.
3. **The decision** (1 minute): Rollback or fix-forward? Why? What factors influenced the choice (blast radius, migration state, time to fix)?
4. **Execution** (30 seconds): How did you execute the rollback/fix? How long until recovery?
5. **Aftermath** (1 minute): What process changes did you make? What did you add to prevent recurrence?

**Example outline to personalize:**

- **Symptoms**: After a deploy, error rate jumped from 0.1% to 5%. Alerts fired within 3 minutes. Errors were `500 Internal Server Error` on the order creation endpoint.
- **Diagnosis**: Filtered logs by the new pods — saw `column "discount_type" does not exist`. A database migration was supposed to run before the deploy, but the migration job had silently failed due to a connection timeout.
- **Decision — fix-forward**: Rolled back code to stop the bleeding (2 minutes). Then investigated: the migration was safe (adding a column, not destructive). Manually ran the migration, verified the column existed, then re-deployed. Total downtime: ~8 minutes.
- **Why not just rollback?**: Rollback alone would have stopped the errors, but the feature was needed for a business deadline. The migration was non-destructive and the fix was clear. If the migration had been destructive or the root cause unclear, pure rollback would have been the right call.
- **Aftermath**: (1) Added a pre-deploy step that verifies migration state matches what the code expects. (2) Made migration job failures block the deploy pipeline instead of failing silently. (3) Added a runbook for "migration didn't run" scenarios.

**Key points to hit:**
- Demonstrate that your first priority was stopping user impact, not understanding the bug
- Show the factors that influenced your rollback vs fix-forward decision (as covered in question 9)
- Emphasize systemic improvements, not just "I fixed the bug" — the interviewer wants to see you make the system more resilient
- If you communicated with stakeholders during the incident, mention it — this signals leadership maturity

</details>

<details>
<summary>20. Tell me about a time you dealt with flaky tests or unreliable pipelines that were slowing down the team — how did you identify the root causes, what was your strategy for fixing them (quarantine, rewrite, infrastructure changes), and how did you balance fixing flakiness against shipping features?</summary>

**What the interviewer is looking for:**
- Can you systematically identify root causes of flakiness (not just retry and hope)?
- Do you understand the cost of flaky tests on team velocity and trust in CI?
- Can you prioritize — fix the highest-impact flakes first, not boil the ocean?
- How do you make the case for investing in reliability when there's feature pressure?

**Suggested structure:**

1. **The problem and its impact** (30 seconds): How bad was the flakiness? Quantify it if possible (% of builds that failed spuriously, hours of developer time wasted retrying).
2. **Root cause identification** (1 minute): How did you find the causes? What categories of flakiness did you find?
3. **Strategy and execution** (1.5 minutes): What did you do about it? Walk through the specific fixes.
4. **Balancing reliability vs features** (30 seconds): How did you justify the investment?
5. **Results** (30 seconds): What improved? How did you measure it?

**Example outline to personalize:**

- **Problem**: ~20% of CI runs on `main` failed on retry. Developers were re-running pipelines 2-3 times per PR. Team started ignoring CI failures ("it's probably flaky"), which meant real bugs also got merged.
- **Root cause identification**: Pulled 30 days of CI run data. Categorized failures:
  - 40% — timing-dependent integration tests (race conditions with database containers not fully ready)
  - 30% — tests that depended on execution order (shared mutable state between test files)
  - 20% — resource exhaustion (tests OOM on CI's smaller containers)
  - 10% — actual flaky external service calls in tests (no mocking)
- **Strategy**:
  - **Immediate**: Quarantined the 5 worst offenders — moved them to a separate "flaky" test suite that ran but didn't block merges. This restored trust in CI immediately.
  - **Infrastructure**: Added proper health check waits for service containers (instead of `sleep 5`). Increased CI container memory.
  - **Test isolation**: Added `afterEach` cleanup, ensured each test file creates its own data instead of depending on shared seeds. Ran tests with `--randomize` to catch ordering dependencies.
  - **Rewrites**: Tests that called real external APIs were rewritten to use recorded responses (nock/msw).
- **Balancing**: Framed it as a productivity issue, not a testing issue. "20% of CI runs waste ~15 minutes each. Across 50 PRs/week, that's 150 minutes of developer idle time per week, plus the cognitive cost of re-running and re-checking." This made the case without competing with feature work — it was treated as developer productivity infrastructure.
- **Results**: Flaky failure rate dropped from 20% to <2% over 3 sprints. Developers stopped reflexively retrying and started trusting CI results again.

**Key points to hit:**
- Show data-driven identification, not just "I noticed some tests were flaky"
- Distinguish between categories of flakiness — each has a different fix
- Quarantine as an immediate tactic (restore trust now) vs rewrites as a long-term strategy
- The biggest insight: flaky tests erode trust in CI, which means real bugs slip through. The cost isn't just "wasted time" — it's reduced quality.

</details>

<details>
<summary>21. Describe a time you had to coordinate a risky database migration alongside a code deployment — what made the migration risky, how did you plan the rollout (backward compatibility, rollback plan, testing strategy), and what would you do differently if you had to do it again?</summary>

**What the interviewer is looking for:**
- Do you understand why database migrations are uniquely dangerous in CI/CD (as covered in question 6)?
- Can you plan a multi-step rollout with backward compatibility at every stage?
- Do you think about rollback *before* you deploy, not after something breaks?
- Can you reflect honestly on what you'd improve — showing growth mindset, not just war stories?

**Suggested structure:**

1. **Context and risk** (30 seconds): What was the migration? What made it risky? (Large table, schema change on a hot path, data transformation, tight coupling between migration and code change.)
2. **Planning** (1.5 minutes): How did you approach backward compatibility? What was the rollback plan? How did you test it? Walk through the expand-and-contract phases if applicable.
3. **Execution** (30 seconds): How did the actual rollout go? Any surprises?
4. **Retrospective** (30 seconds): What would you do differently?

**Example outline to personalize:**

- **Context**: Needed to split a `users` table — extracting `billing_address`, `shipping_address`, and `payment_method` columns into a separate `user_profiles` table. The `users` table had 15M rows, was read on every authenticated request, and multiple services queried it directly.
- **What made it risky**: (1) Large table — `ALTER TABLE` could lock it. (2) Multiple services needed to change simultaneously. (3) Data needed to be backfilled into the new table. (4) Rollback wasn't trivial — if services started writing to the new table, rolling back meant data loss.
- **Planning**:
  - **Phase 1 (expand)**: Created the `user_profiles` table. Deployed code that writes to *both* tables but reads from `users` only. No service needed to change behavior yet. Ran a background job to backfill existing data (batched, rate-limited to avoid I/O pressure).
  - **Phase 2 (migrate reads)**: Deployed code that reads from `user_profiles` for address/payment data but falls back to `users` if the profile doesn't exist (belt-and-suspenders for any backfill gaps). Monitored for a week to confirm reads were working correctly.
  - **Phase 3 (contract)**: Stopped writing address/payment data to `users`. Removed the old columns after confirming no service was reading them (checked query logs).
  - **Rollback plan**: At each phase, the previous code version could still work because both tables existed with consistent data. Phase 3 (dropping columns) was the point of no return — we delayed it by two weeks and required manual approval.
  - **Testing**: Ran the full migration against a production-size snapshot in staging. Measured lock duration, backfill time, and query performance. Ran load tests against the dual-write code to ensure it didn't significantly increase latency.
- **Execution**: Phase 1 and 2 went smoothly. During backfill, we discovered ~200 rows with malformed address data that caused the backfill job to fail. Had to add error handling and a dead-letter queue for bad rows.
- **What I'd do differently**: (1) Profile the source data before starting — we would have caught the malformed rows earlier. (2) Set up a dashboard tracking migration progress (% of reads from new table, backfill completion) from day one instead of building it reactively. (3) Use a feature flag for the read cutover instead of a code deploy — would have allowed instant rollback of the read path.

**Key points to hit:**
- Show that you broke the migration into safe, independently deployable phases — not a big-bang "migrate everything at once"
- Demonstrate that at every phase, both old and new code could coexist (backward compatibility)
- Mention how you tested at production scale, not just "it worked in dev"
- The "what I'd do differently" should be genuine and specific — interviewers are testing self-awareness, not perfection
- If you coordinated across teams (multiple services needed changes), highlight the communication and sequencing

</details>
