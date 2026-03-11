# Frontend Architecture

> **38 questions** — 19 theory, 19 practical

- Frontend architecture tradeoffs: when to centralize vs federate, team autonomy vs consistency, build-time coupling vs runtime composition
- Monorepo strategies: Nx vs Turborepo — task orchestration, caching, dependency graphs
- Micro-frontends: composition strategies (Module Federation, iframe, server-side), tradeoffs and controversies
- Cross-app state and data sharing: shared state boundaries in monorepos, server state vs client state at the architecture level, API layer patterns (shared SDK, GraphQL federation)
- Design systems: tokens, primitives, composed components, versioning, governance, token pipelines (Style Dictionary, CSS custom properties, TypeScript theme objects)
- Testing at scale: Vitest, Testing Library, Playwright — test trophy balance, flaky E2E diagnosis and stabilization, test isolation patterns
- Accessibility (a11y): WCAG compliance, design system enforcement, CI-level a11y checks
- Internationalization (i18n): react-intl, next-intl, compile-time vs runtime loading, bundle impact
- Build tooling: Vite vs webpack — dev server, production builds, migration tradeoffs
- Module boundaries: ESLint import restrictions, Nx module boundary rules, TypeScript path constraints, workspace constraints — preventing cross-app imports
- CI pipelines for monorepos: affected-only testing, remote caching, parallelized E2E
- Frontend deployment: static hosting vs SSR, CDN caching and invalidation, preview environments per PR, environment variable injection at build vs runtime
- Debugging monorepo build failures: workspace hoisting, build order, path resolution
- Bundle size regression analysis and size budgets
- Frontend observability: error tracking (Sentry), performance monitoring (Core Web Vitals), feature flags, source maps in production

---

## Foundational

<details>
<summary>1. What are the key dimensions of frontend architecture (code organization, build pipeline, shared UI, testing, deployment) and why do they need to be designed as separate but interconnected concerns — what goes wrong when teams treat frontend architecture as just "pick a framework and start coding"?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>2. Why have monorepos become the dominant approach for large frontend codebases — what specific problems do they solve compared to polyrepos (dependency management, code sharing, atomic changes), what new problems do they introduce (build times, CI complexity, ownership boundaries), and when is a polyrepo still the better choice?</summary>

<!-- Answer will be added later -->

</details>

## Conceptual Depth

<details>
<summary>3. How do Nx and Turborepo differ in their approach to monorepo management — compare their task orchestration models, caching strategies (local and remote), dependency graph computation, and plugin ecosystems, and what factors should drive the choice between them for a team starting a new monorepo?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>4. What are the main micro-frontend composition strategies (Module Federation, iframes, server-side composition) and what are the real tradeoffs of each — why is micro-frontend architecture controversial, when does the complexity pay off, and when is it architectural astronautics that a team will regret?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>5. How should a design system be architecturally layered — what is the role of design tokens, primitive components, and composed components, how do these layers relate to each other, and what are the tradeoffs of a thin design system (tokens + primitives only) vs a thick one (fully composed, opinionated components)?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>6. How do you handle design system versioning and governance at scale — what versioning strategy prevents consuming teams from breaking on updates, what governance model balances consistency with team autonomy, and what are the signs that a design system has become a bottleneck vs drifted into inconsistency?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>7. Why do design systems need a token pipeline (Style Dictionary, CSS custom properties, TypeScript theme objects) instead of just hardcoding values — how does a token pipeline work end to end from design tool to multi-platform output, and what are the tradeoffs between CSS custom properties and TypeScript theme objects as the consumption format?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>8. What does WCAG compliance actually require at levels A and AA — which level should most teams target and why, what are the most commonly violated criteria, and why is AAA compliance impractical for most applications?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>9. How can a design system enforce accessibility by default — what patterns (accessible primitives, required aria props, color contrast tokens) shift the burden from individual developers to the system, and why are CI-level a11y checks (axe-core linting, Playwright audits) necessary even when developers believe they are building accessible UIs?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>10. How do react-intl and next-intl differ in their approach to internationalization — what are the tradeoffs between compile-time message extraction and runtime loading, how does each strategy impact bundle size, and what architectural decisions around i18n are hardest to change later?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>11. How do Vite and webpack differ architecturally in their dev server approach and production build pipeline — why is Vite's native ESM dev server faster, what does webpack still do better for certain use cases, and what are the real migration tradeoffs when moving an existing webpack project to Vite?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>12. Why do large frontend codebases need explicit module boundaries and dependency rules — what happens without them, what enforcement strategies exist (ESLint rules, Nx module boundaries, TypeScript path restrictions, workspace constraints), and how do you design boundaries that match team ownership without creating excessive coupling or duplication?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>13. How should you balance unit tests, integration tests, and E2E tests in a large frontend codebase — why has the "testing trophy" replaced the "testing pyramid" for frontend, what does each layer of the trophy actually test, and what testing strategy gives the best confidence-to-maintenance-cost ratio at scale?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>14. How do Vitest, Testing Library, and Playwright each fit into the frontend testing stack — what layer does each tool own, where do their responsibilities overlap, and what are the tradeoffs of standardizing on this stack vs alternatives (Jest, Cypress, WebdriverIO)?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>15. Why do teams need bundle size budgets and how should regression analysis work — what causes bundle size to silently grow over time, what metrics matter beyond raw bundle size (per-route sizes, shared chunk sizes, tree-shaking effectiveness), and how do you set budgets that are strict enough to catch regressions but not so strict that developers fight them?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>16. Why are E2E tests disproportionately flaky compared to unit and integration tests — what are the main categories of flakiness (timing/race conditions, test isolation, environment differences, third-party dependencies), what stabilization patterns actually work vs just masking the problem, and when should you delete a flaky test instead of fixing it?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>17. How should you handle cross-app state and data sharing in a frontend monorepo — where should the boundaries be between shared state and app-local state, how does the distinction between server state (React Query, SWR) and client state (Zustand, Redux) affect architecture decisions, and what are the tradeoffs of API layer patterns like a shared SDK vs GraphQL federation for data access across apps?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>18. What are the key deployment architecture decisions for a frontend application — when should you choose static hosting vs SSR, how do CDN caching and cache invalidation strategies differ between the two, and why is the choice between injecting environment variables at build time vs runtime an architectural decision that is hard to change later?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>19. Why does a production frontend need dedicated observability beyond backend monitoring — what do error tracking (Sentry), performance monitoring (Core Web Vitals), and feature flags each solve, how do they interact (e.g., feature flag rollouts triggering error spikes), and what are the tradeoffs of uploading source maps to production error tracking services?</summary>

<!-- Answer will be added later -->

</details>

## Practical — Configuration & Implementation

<details>
<summary>20. Set up an Nx or Turborepo monorepo with multiple frontend apps and shared libraries — show the workspace configuration, demonstrate affected-only test execution and remote caching setup, and explain what configuration mistakes lead to cache misses that silently defeat the performance benefits.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>21. Configure Module Federation for a micro-frontend setup where a host app loads remote modules at runtime — show the webpack or Vite plugin configuration for both host and remote, explain how shared dependencies (React, shared state) are handled to avoid duplication, and what breaks when version mismatches occur between host and remote.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>22. Build a design token pipeline using Style Dictionary — show the token definition format, the Style Dictionary configuration that outputs CSS custom properties and TypeScript theme objects, and explain how this pipeline integrates into the build process so that token changes propagate to all consuming applications automatically.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>23. Set up automated accessibility testing at multiple levels — show how to integrate axe-core with Testing Library for component-level a11y checks, configure Playwright for page-level a11y audits, and add CI gates that block merges when a11y violations are introduced. Explain which violations should be errors vs warnings and why.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>24. Configure internationalization in a React application using react-intl or next-intl — show the setup for compile-time message extraction, lazy-loading locale data per route, and the build configuration that ensures unused locales don't bloat the bundle. Explain what breaks if you skip the extraction step and inline messages directly.</summary>

<!-- Answer will be added later -->

</details>

## Practical — CI & Production Workflows

<details>
<summary>25. Design a CI pipeline for a frontend monorepo that runs affected-only testing, uses remote caching, and parallelizes E2E tests across multiple workers — show the pipeline configuration (CircleCI or GitHub Actions), explain how the "affected" calculation works when a shared library changes, and what safeguards prevent a cache corruption from silently passing broken builds.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>26. Set up bundle size budgets in a CI pipeline that blocks PRs when a size threshold is exceeded — show the tooling configuration (bundlesize, size-limit, or webpack-bundle-analyzer), explain how to set per-route budgets vs overall budgets, and demonstrate how the CI output helps the developer identify exactly what caused the regression.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>27. Configure Vite for a production React application — show the vite.config.ts with chunking strategy, dependency pre-bundling, environment variables, and proxy setup, then show the equivalent webpack configuration for the same setup and explain which parts of the migration are straightforward vs which require workarounds.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>28. Enforce module boundaries and dependency rules in a monorepo — show how to configure Nx module boundary rules (or ESLint import restrictions for Turborepo), set up workspace-level constraints that prevent apps from importing from other apps, and demonstrate what happens when a developer violates these rules in their IDE and in CI.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>29. Set up a frontend deployment pipeline that deploys to a CDN with proper cache invalidation, creates preview environments for each PR, and handles environment variable injection at runtime instead of build time — show the configuration and explain what breaks when teams hardcode environment variables at build time and need to promote the same artifact across environments.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>30. Set up frontend observability for a production React application — show how to integrate Sentry for error tracking with source map uploads in CI, configure Core Web Vitals reporting, and implement a feature flag check that gates a new feature. Explain what you lose if source maps are not uploaded and how to prevent them from being publicly accessible.</summary>

<!-- Answer will be added later -->

</details>

## Practical — Debugging & Troubleshooting

<details>
<summary>31. A monorepo build is failing with cryptic errors after adding a new shared package — walk through the debugging process for the three most common causes: workspace hoisting conflicts (multiple versions of a dependency), incorrect build order (a package referencing another package's unbundled source), and path resolution errors (TypeScript paths vs bundler aliases). Show the exact commands to diagnose each.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>32. An E2E test suite is failing intermittently — about 10% of runs fail with different tests each time. Walk through the systematic diagnosis: identifying timing-dependent assertions, checking for test isolation failures (shared state, database/API leakage between tests), handling network request flakiness, and deciding between retry strategies vs proper fixes. Show Playwright-specific debugging techniques.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>33. A PR introduces a 150KB bundle size regression but the change looks small — walk through how to trace the regression to its root cause using source map analysis and dependency visualization, identify common culprits (barrel file re-exports pulling in entire libraries, CSS-in-JS runtime additions, locale data bundling, missing tree-shaking), and show the fix for each.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>34. A design system component update (new major version) is breaking multiple consuming applications with different errors — walk through how to diagnose whether the breaks are from changed props, removed tokens, CSS specificity changes, or peer dependency mismatches, and explain the release process that should have prevented this (changesets, visual regression testing, migration codemods).</summary>

<!-- Answer will be added later -->

</details>

---

## Experience-Based Questions

These questions test real-world experience. Prepare by mapping them to your own projects and situations.

<details>
<summary>35. Tell me about a time you designed or significantly refactored a frontend monorepo structure — what drove the decision, how did you handle the migration, and what would you do differently knowing what you know now?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>36. Describe a time you built or evolved a design system that multiple teams consumed — how did you handle governance, versioning, and adoption, and what was the hardest part about scaling it across teams?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>37. Tell me about a time you diagnosed and fixed a persistent CI or build performance problem in a frontend codebase — what were the symptoms, what was the root cause, and what measurable improvement did you achieve?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>38. Describe a time you had to make a build-vs-buy decision for frontend infrastructure tooling (monorepo tool, design system, testing framework, CI setup) — what factors drove your decision, and how did it play out?</summary>

<!-- Answer framework will be added later -->

</details>
