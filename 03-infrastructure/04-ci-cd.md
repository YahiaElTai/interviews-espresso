# CI/CD

> **38 questions** — 11 theory, 27 practical

- CI vs CD (delivery vs deployment): why they're separate concerns
- Pipeline design end to end: git push to production traffic — stage ordering and rationale
- Static analysis and quality gates: linting, type checking, SAST, dependency vulnerability scanning (Snyk, Dependabot), fail-fast ordering in pipelines
- Build-release-run separation (12-factor app)
- Branching strategies: trunk-based development vs long-lived feature branches — CI tradeoffs
- Deployment strategies: rolling, blue-green, canary — infrastructure requirements, rollback speed, blast radius
- Feature flags in deployment: decoupling deploy from release, flag-driven canary rollouts, cleanup discipline, managed services (LaunchDarkly, Unleash) vs homegrown
- Database migrations in pipelines: expand-and-contract, backward compatibility during rollouts
- Environment promotion (dev → staging → production): what should differ vs stay identical
- CircleCI in depth: orbs, workflows, fan-out/fan-in, test splitting, parallelism, caching (save/restore + DLC), contexts, approval jobs, branch filters
- GitHub Actions: workflows, reusable workflows, composite actions, matrix builds, runner model, comparison with CircleCI
- GitOps with ArgoCD and Flux: pull-based vs push-based CD, sync policies, drift detection, when GitOps adds unnecessary complexity
- Secrets in CI/CD: risks of env vars in logs/config, masking, OIDC-based authentication to cloud providers
- Pipeline security: supply chain risks (third-party orbs/actions), pinning versions, least-privilege CI credentials
- Docker image builds in pipelines: tagging strategy, registry push, K8s deployment triggers
- Integration testing in CI: service containers, testcontainers, test database state
- K8s deployments from CI: triggering rollouts, rollout status checks, automated rollback on failure
- Pipeline optimization: caching, parallelism, test splitting, selective builds, monorepo strategies
- Rollback strategies: code-only vs code + migration rollbacks, fix-forward decisions, incident identification, health verification
- Debugging flaky pipelines: timing issues, resource limits, test ordering, local vs CI environment differences, SSH rerun and debug logging

---

## Foundational

<details>
<summary>1. Why are Continuous Integration, Continuous Delivery, and Continuous Deployment treated as separate concerns — what specific problem does each one solve, what does a team lose when they blur the boundaries (e.g., deploying straight from CI without a delivery gate), and how do they compose into a pipeline that balances speed with safety?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>2. Walk through the stages of a CI/CD pipeline from git push to production traffic — what happens at each stage (build, test, static analysis, artifact creation, staging deploy, acceptance tests, production deploy, smoke tests), why are the stages ordered this way, and what are the consequences of skipping or reordering stages (e.g., deploying before integration tests, or running linting after a 20-minute test suite)?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>3. What is the build-release-run separation from the 12-factor app methodology, why does it insist that builds produce immutable artifacts that are strictly separated from configuration, and what real-world problems occur when teams violate this (e.g., building different artifacts per environment, or baking secrets into images)?</summary>

<!-- Answer will be added later -->

</details>

## Conceptual Depth

<details>
<summary>4. Why does trunk-based development enable CI better than long-lived feature branches — what happens to integration frequency, merge conflict severity, and feedback loop speed as branch lifetime increases, and when might short-lived feature branches (1-2 days) be a reasonable middle ground vs committing directly to trunk?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>5. Compare rolling, blue-green, and canary deployment strategies — what infrastructure does each one require, how fast can each roll back, what is the blast radius of a bad deploy for each, and how do you decide which strategy fits a given service (considering factors like database coupling, session state, and traffic volume)?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>6. Why are database migrations one of the hardest problems in CI/CD — what breaks when you deploy new code that expects a new schema before the migration runs (or vice versa), how does the expand-and-contract pattern solve backward compatibility during rolling deployments, and when is a migration too dangerous to run automatically in a pipeline?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>7. When promoting artifacts through environments (dev → staging → production), what should stay identical across environments vs what should differ — why must the artifact (Docker image, compiled binary) be the same, what configuration should change (database URLs, feature flags, resource limits), and what common mistakes cause "works in staging, breaks in production" situations?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>8. What problem does GitOps solve that traditional push-based CD doesn't — how does the pull-based model (ArgoCD, Flux watching a Git repo) differ from push-based (CI pipeline runs kubectl apply), what does drift detection give you, and when does adopting GitOps add unnecessary complexity for a team that would be better served by a simpler push-based pipeline?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>9. What are the real supply chain risks in CI/CD pipelines — how can a compromised third-party orb, GitHub Action, or base image inject malicious code into your production artifacts, what does pinning versions and verifying checksums protect against, and how should you scope CI credentials to enforce least privilege (e.g., deploy-only tokens, namespace-scoped K8s service accounts)?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>10. Compare the tradeoffs of rolling back code-only vs rolling back code plus database migrations — when is fix-forward (shipping a new fix instead of reverting) the better choice, how do you identify that a deploy is the cause of an incident vs an unrelated issue, and what health verification steps should run after every rollback to confirm the system is actually recovered?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>11. Why do teams decouple deployment from release using feature flags -- how does a flag-driven rollout differ from a canary deployment, what is flag cleanup discipline and why does skipping it create long-term technical debt, and when should you use a managed service (LaunchDarkly, Unleash) vs a homegrown feature flag system?</summary>

<!-- Answer will be added later -->

</details>

## CircleCI — Practical

<details>
<summary>12. Show a CircleCI workflow configuration that uses fan-out/fan-in to run linting, unit tests, and integration tests in parallel, then fans in to a build step, followed by an approval job gating production deployment — include branch filters so only the main branch reaches the approval step. Explain what happens if one of the parallel jobs fails and why the fan-in pattern speeds up pipelines without sacrificing safety.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>13. What are CircleCI orbs and contexts, what problems do they solve, and what are the risks of using third-party orbs — show a config.yml that uses an orb (e.g., aws-cli or docker) and a context for environment-scoped secrets, and explain how to audit orbs for security and when to inline the commands instead of trusting an orb?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>14. Configure CircleCI test splitting with parallelism to distribute a slow test suite across multiple containers -- show the YAML with the parallelism key and circleci tests split command, explain the splitting strategies (by timing data vs by file count), how timing data is collected and stored, and what problems arise when timing data is stale or missing.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>15. Set up dependency caching and Docker Layer Caching in CircleCI -- show the YAML for save_cache/restore_cache with a well-designed cache key, explain what makes a cache key design good vs bad (staleness, cache misses, over-broad keys), when to use Docker Layer Caching, and what are the cost and correctness tradeoffs of DLC?</summary>

<!-- Answer will be added later -->

</details>

## GitHub Actions — Practical

<details>
<summary>16. Show a GitHub Actions workflow that uses reusable workflows and composite actions — explain the difference between the two, when you'd choose one over the other, demonstrate how a reusable workflow is called from a parent workflow with inputs and secrets, and explain how this compares structurally to CircleCI's orbs for sharing CI logic across repos?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>17. Configure a GitHub Actions matrix build that tests across multiple Node.js versions and operating systems — show the YAML with the matrix strategy, explain how fail-fast works, how to exclude specific combinations, and compare the GitHub Actions runner model (hosted vs self-hosted) with CircleCI's resource classes and machine executors in terms of cost, performance, and security?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>18. You're migrating a project from CircleCI to GitHub Actions (or vice versa) -- for the three most important CI features (workflows/parallelism, caching, and approval gates), show the equivalent configuration in both platforms side by side, explain what features exist in one but not the other, and what practical tradeoffs would influence which platform you choose for a new project?</summary>

<!-- Answer will be added later -->

</details>

## Practical — Build, Test & Deploy

<details>
<summary>19. How do secrets leak in CI/CD pipelines and how do you prevent it -- show concrete examples of how environment variables end up in logs (debug mode, error stack traces, docker build args), demonstrate how secret masking works in CircleCI and GitHub Actions, and what are the limitations of masking that still leave you vulnerable?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>20. Configure OIDC-based authentication from a CI pipeline to a cloud provider (GCP or AWS) so your pipeline never stores long-lived credentials -- show the workflow YAML for either CircleCI or GitHub Actions, explain how the OIDC token exchange works, and why is this approach better than static service account keys for security and credential rotation?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>21. Configure a CI pipeline quality gate that runs linting, type checking, SAST scanning, and dependency vulnerability checking (using tools like Snyk or Dependabot) -- show the pipeline YAML, explain why these steps should be ordered to fail fast (cheapest/fastest checks first), and what happens when teams skip SAST or dependency scanning and only discover vulnerabilities in production?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>22. Configure a CI pipeline that runs integration tests against real dependencies — show the config for spinning up service containers (PostgreSQL, Redis) alongside your test runner in both CircleCI and GitHub Actions, explain how testcontainers offers an alternative approach, and how do you handle test database state (migrations, seeding, cleanup between tests) so tests are reliable and isolated?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>23. Show the CI pipeline steps for deploying to Kubernetes — include updating the image tag in the deployment manifest, running kubectl rollout status to wait for completion, and configuring automated rollback when the rollout fails (e.g., readiness probe failures). What exit codes and signals tell you a rollout failed, and how do you avoid a situation where CI reports success but the deployment is actually broken?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>24. Configure a canary deployment pipeline that deploys new code to a small percentage of traffic first, runs automated health checks (error rate, latency percentiles), and either promotes to full rollout or automatically rolls back — show the pipeline YAML and deployment configuration (using K8s native, Argo Rollouts, or Flagger), and explain what metrics you'd check and what thresholds trigger a rollback?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>25. Show how to integrate a database migration step into a deployment pipeline using the expand-and-contract pattern — write the pipeline stages that run the "expand" migration before deploying new code, verify both old and new code work with the expanded schema, then schedule the "contract" cleanup migration for a later pipeline. What safeguards prevent a failed migration from taking down the service?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>26. Configure a pipeline that promotes the exact same Docker image through dev → staging → production environments, changing only the configuration (environment variables, resource limits, replica counts) at each stage — show the pipeline YAML and explain how you ensure artifact identity across environments, how approval gates work between stages, and what verification steps run after each promotion?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>27. Set up ArgoCD to watch a Git repository and automatically sync Kubernetes manifests to a cluster — show the Application CRD YAML with sync policies (auto-sync, self-heal, prune), explain how drift detection works (what happens when someone runs kubectl edit directly), and configure a sync window that prevents deployments during off-hours. When would you choose Flux over ArgoCD?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>28. Your pipeline takes 25 minutes and the team is complaining -- walk through the highest-impact optimization strategies: dependency caching (show cache key design for both CircleCI and GitHub Actions) and parallelizing test suites with test splitting by timing data. Show config snippets for each and explain which optimizations typically yield the biggest time savings.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>29. Configure selective/conditional builds in a CI pipeline so that only relevant tests run when specific parts of the codebase change -- show how to implement path-based filtering in both CircleCI and GitHub Actions, explain how monorepo strategies (affected-project detection, Nx/Turborepo integration) differ from simple path filters, and what are the risks of being too aggressive with filtering (missed regressions)?</summary>

<!-- Answer will be added later -->

</details>

## Practical — Debugging & Troubleshooting

<details>
<summary>30. A pipeline that was green for weeks is now failing intermittently — the same commit passes on retry but fails 30% of the time. Walk through the systematic debugging process: checking for timing-dependent tests, resource limits (OOM, disk space), test ordering dependencies, external service flakiness, and race conditions. How do you use SSH rerun (CircleCI) or debug logging (GitHub Actions) to inspect the environment during a failure, and what's your strategy for permanently fixing flaky tests vs quarantining them?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>31. A test suite passes locally on every developer's machine but consistently fails in CI — walk through the differences between local and CI environments that commonly cause this (Docker version, OS differences, filesystem case sensitivity, available memory, network access, time zones, missing system dependencies), how you diagnose which difference is the culprit, and what you change to make local and CI environments as close as possible?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>32. You discover that a third-party GitHub Action your pipeline depends on was compromised in a recent version update — walk through the immediate response (how you detect this, what you check for in your recent builds), the remediation steps (pinning to a known-good SHA, auditing your artifacts and deployed services), and the long-term prevention strategy (fork-and-mirror, action allowlists, Dependabot for action versions)?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>33. A production deploy goes out at 3pm, and by 3:15pm monitoring shows a spike in 500 errors — walk through the decision process: how do you confirm the deploy caused it vs a coincidental issue, what signals tell you to roll back immediately vs investigate first, show the exact commands to roll back (kubectl rollout undo, reverting the Git commit for GitOps, or re-deploying the previous image tag), and what health checks must pass before you declare the rollback successful?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>34. A deployment pipeline runs a database migration that adds a NOT NULL column without a default value, and the migration succeeds — but then the rolling deployment starts failing because old pods (still running) can't write to the table. Walk through why this happened, how to recover without data loss, and what pipeline safeguards would have caught this before it reached production?</summary>

<!-- Answer will be added later -->

</details>

---

## Experience-Based Questions

These questions test real-world experience. Prepare by mapping them to your own projects and situations.

<details>
<summary>35. Tell me about a time you designed or significantly improved a CI/CD pipeline — what was the pipeline's state before, what changes did you make, what tradeoffs did you navigate (speed vs safety, complexity vs maintainability), and what was the measurable impact on deployment frequency or developer experience?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>36. Describe a time a deployment went wrong in production and you had to decide between rolling back and fixing forward — what were the symptoms, how did you diagnose the root cause, what decision did you make and why, and what process changes did you implement afterward to prevent a recurrence?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>37. Tell me about a time you dealt with flaky tests or unreliable pipelines that were slowing down the team — how did you identify the root causes, what was your strategy for fixing them (quarantine, rewrite, infrastructure changes), and how did you balance fixing flakiness against shipping features?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>38. Describe a time you had to coordinate a risky database migration alongside a code deployment — what made the migration risky, how did you plan the rollout (backward compatibility, rollback plan, testing strategy), and what would you do differently if you had to do it again?</summary>

<!-- Answer framework will be added later -->

</details>
