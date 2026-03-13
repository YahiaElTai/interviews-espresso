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
<summary>8. What are the real supply chain risks in CI/CD pipelines — how can a compromised third-party orb, GitHub Action, or base image inject malicious code into your production artifacts, what does pinning versions and verifying checksums protect against, and how should you scope CI credentials to enforce least privilege (e.g., deploy-only tokens, namespace-scoped K8s service accounts)?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>9. Compare the tradeoffs of rolling back code-only vs rolling back code plus database migrations — when is fix-forward (shipping a new fix instead of reverting) the better choice, how do you identify that a deploy is the cause of an incident vs an unrelated issue, and what health verification steps should run after every rollback to confirm the system is actually recovered?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>10. Why do teams decouple deployment from release using feature flags -- how does a flag-driven rollout differ from a canary deployment, what is flag cleanup discipline and why does skipping it create long-term technical debt, and when should you use a managed service (LaunchDarkly, Unleash) vs a homegrown feature flag system?</summary>

<!-- Answer will be added later -->

</details>

## Practical — Build, Test & Deploy

<details>
<summary>11. How do secrets leak in CI/CD pipelines and how do you prevent it -- show concrete examples of how environment variables end up in logs (debug mode, error stack traces, docker build args), demonstrate how secret masking works in CircleCI and GitHub Actions, and what are the limitations of masking that still leave you vulnerable?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>12. Configure a CircleCI pipeline for a Node.js monorepo — show how orbs simplify reusable config, how to structure workflows with parallel jobs, implement test splitting across multiple executors to reduce wall-clock time, configure dependency caching (save_cache/restore_cache vs Docker Layer Caching), and add an approval job that gates production deployment behind a manual sign-off.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>13. Configure a GitHub Actions workflow for the same Node.js application — show a reusable workflow (workflow_call) that can be invoked from multiple repos, a matrix build that tests across Node versions, and how composite actions reduce duplication. Compare the GitHub Actions model (event-driven, marketplace actions) against CircleCI (orbs, resource classes, test splitting) — where does each platform have a genuine advantage?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>14. Configure a CI pipeline quality gate that runs linting, type checking, SAST scanning, and dependency vulnerability checking (using tools like Snyk or Dependabot) -- show the pipeline YAML, explain why these steps should be ordered to fail fast (cheapest/fastest checks first), and what happens when teams skip SAST or dependency scanning and only discover vulnerabilities in production?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>15. Configure a CI pipeline that runs integration tests against real dependencies — show the config for spinning up service containers (PostgreSQL, Redis) alongside your test runner in both CircleCI and GitHub Actions, explain how testcontainers offers an alternative approach, and how do you handle test database state (migrations, seeding, cleanup between tests) so tests are reliable and isolated?</summary>

<!-- Answer will be added later -->

</details>

## Practical — Debugging & Troubleshooting

<details>
<summary>16. You discover that a third-party GitHub Action your pipeline depends on was compromised in a recent version update — walk through the immediate response (how you detect this, what you check for in your recent builds), the remediation steps (pinning to a known-good SHA, auditing your artifacts and deployed services), and the long-term prevention strategy (fork-and-mirror, action allowlists, Dependabot for action versions)?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>17. A production deploy goes out at 3pm, and by 3:15pm monitoring shows a spike in 500 errors — walk through the decision process: how do you confirm the deploy caused it vs a coincidental issue, what signals tell you to roll back immediately vs investigate first, show the exact commands to roll back (kubectl rollout undo, reverting the Git commit for GitOps, or re-deploying the previous image tag), and what health checks must pass before you declare the rollback successful?</summary>

<!-- Answer will be added later -->

</details>

---

## Experience-Based Questions

These questions test real-world experience. Prepare by mapping them to your own projects and situations.

<details>
<summary>18. Tell me about a time you designed or significantly improved a CI/CD pipeline — what was the pipeline's state before, what changes did you make, what tradeoffs did you navigate (speed vs safety, complexity vs maintainability), and what was the measurable impact on deployment frequency or developer experience?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>19. Describe a time a deployment went wrong in production and you had to decide between rolling back and fixing forward — what were the symptoms, how did you diagnose the root cause, what decision did you make and why, and what process changes did you implement afterward to prevent a recurrence?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>20. Tell me about a time you dealt with flaky tests or unreliable pipelines that were slowing down the team — how did you identify the root causes, what was your strategy for fixing them (quarantine, rewrite, infrastructure changes), and how did you balance fixing flakiness against shipping features?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>21. Describe a time you had to coordinate a risky database migration alongside a code deployment — what made the migration risky, how did you plan the rollout (backward compatibility, rollback plan, testing strategy), and what would you do differently if you had to do it again?</summary>

<!-- Answer framework will be added later -->

</details>
