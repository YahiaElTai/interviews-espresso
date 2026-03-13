# Frontend Architecture

> **25 questions** — 13 theory, 8 practical, 4 experience

- Frontend architecture tradeoffs: when to centralize vs federate, team autonomy vs consistency, build-time coupling vs runtime composition
- Monorepo strategies: Nx vs Turborepo — task orchestration, caching, dependency graphs
- Micro-frontends: composition strategies (Module Federation, iframe, server-side), tradeoffs and controversies
- Cross-app state and data sharing: shared state boundaries in monorepos, server state vs client state at the architecture level, API layer patterns (shared SDK, GraphQL federation)
- Design systems: tokens, primitives, composed components, versioning, governance
- Testing at scale: Vitest, Testing Library, Playwright — test trophy balance, flaky E2E diagnosis and stabilization, test isolation patterns
- Build tooling: Vite vs webpack — dev server, production builds, migration tradeoffs
- Module boundaries: ESLint import restrictions, Nx module boundary rules, TypeScript path constraints, workspace constraints — preventing cross-app imports
- CI pipelines for monorepos: affected-only testing, remote caching, parallelized E2E
- Frontend deployment: static hosting vs SSR, CDN caching and invalidation, preview environments per PR, environment variable injection at build vs runtime

---

## Foundational

<details>
<summary>1. What are the key dimensions of frontend architecture (code organization, build pipeline, shared UI, testing, deployment) and why do they need to be designed as separate but interconnected concerns — what goes wrong when teams treat frontend architecture as just "pick a framework and start coding"?</summary>

The key dimensions are:

- **Code organization** — How source code is structured: monorepo vs polyrepo, app boundaries, shared libraries, module boundaries, and ownership mapping.
- **Build pipeline** — How code becomes deployable artifacts: bundler choice, compilation, code splitting, tree-shaking, and build caching.
- **Shared UI** — Design systems and component libraries: tokens, primitives, composed components, and how they're consumed across apps.
- **Testing** — Strategy across layers: unit/integration/E2E balance, tooling choices, CI integration, and test isolation.
- **Deployment** — How artifacts reach users: static hosting vs SSR, CDN caching, preview environments, and environment variable injection.

These are interconnected but distinct. Code organization determines what can be built independently. Build pipeline constraints shape code organization (circular dependency). Shared UI affects both build pipeline (how it's packaged) and testing (how components are tested in isolation vs in consuming apps). Deployment architecture constrains build output format.

**What goes wrong without intentional design:**

- **Accidental coupling** — Without module boundaries, apps quietly import from each other. A change in one app breaks three others, and nobody understands the dependency graph.
- **Build times explode** — Without caching strategy or affected-only builds, CI rebuilds everything on every commit. A 40-minute pipeline kills developer velocity.
- **Inconsistent UI** — Without a design system, teams create slightly different buttons, modals, and form inputs. The product looks fragmented, and fixing a cross-cutting concern (accessibility, dark mode) becomes an N-app problem.
- **Testing gaps** — Without a deliberate strategy, teams either over-test at the wrong layer (brittle E2E tests for logic that should be unit tested) or under-test integration points.
- **Deployment friction** — Hardcoded environment variables at build time, no preview environments, and manual deployment processes mean slow and risky releases.

The common failure mode is that teams make implicit decisions about these dimensions by default (whatever the framework CLI scaffolds), then discover the consequences months later when the codebase is too large to easily restructure.

</details>

<details>
<summary>2. Why have monorepos become the dominant approach for large frontend codebases — what specific problems do they solve compared to polyrepos (dependency management, code sharing, atomic changes), what new problems do they introduce (build times, CI complexity, ownership boundaries), and when is a polyrepo still the better choice?</summary>

**Problems monorepos solve:**

- **Dependency management** — In a polyrepo, shared libraries are published as packages. Updating a shared utility means publishing a new version, then updating every consuming repo. In a monorepo, you change the shared code and all consumers see it immediately — no version drift.
- **Code sharing** — Extracting shared code in polyrepos requires creating a new repo, setting up publishing, and managing semver. In a monorepo, you create a new library directory and import from it. The friction to share code drops dramatically.
- **Atomic changes** — When a refactor touches a shared API and its consumers, a monorepo lets you make the change in a single commit and PR. In polyrepos, you coordinate across repos, often with backwards-compatible intermediate steps.
- **Consistent tooling** — One ESLint config, one TypeScript config, one CI pipeline. No drift between repos on formatting, linting, or build conventions.

**Problems monorepos introduce:**

- **Build times** — Without affected-only builds and caching, CI rebuilds the entire monorepo on every change. A naive setup gets slower with every new app.
- **CI complexity** — You need task orchestration (Nx, Turborepo), remote caching, and affected-only testing. The CI pipeline itself becomes infrastructure that needs maintenance.
- **Ownership boundaries** — When everything is in one repo, it's easy for teams to bypass boundaries — importing from another team's app, making changes to shared code without consulting owners. You need explicit module boundary rules.
- **IDE performance** — Large monorepos strain IDE indexing, type checking, and autocomplete. TypeScript project references and incremental builds help but add configuration complexity.
- **Git performance** — Very large monorepos can slow down git operations. Shallow clones, sparse checkouts, or tools like VFS for Git become necessary at extreme scale.

**When polyrepos are still better:**

- **Truly independent products** with different tech stacks, release cadences, and teams that rarely share code. The coordination cost of a monorepo exceeds the sharing benefit.
- **Open-source or cross-organization projects** where access control boundaries matter (you can't give every contributor access to everything).
- **Small teams with 1-3 apps** where the monorepo tooling overhead isn't justified — a simple polyrepo with a published component library works fine.
- **Regulatory or compliance boundaries** requiring strict code separation between domains.

</details>

## Conceptual Depth

<details>
<summary>3. How do Nx and Turborepo differ in their approach to monorepo management — compare their task orchestration models, caching strategies (local and remote), dependency graph computation, and plugin ecosystems, and what factors should drive the choice between them for a team starting a new monorepo?</summary>

**Task orchestration:**

- **Nx** computes a full project dependency graph (including implicit dependencies from source imports) and uses it to determine build order, parallelization, and affected projects. It has a rich task pipeline configuration with `targetDefaults` and per-project targets.
- **Turborepo** uses `turbo.json` to define a task pipeline with explicit `dependsOn` declarations. It relies on `package.json` workspace dependencies for the graph rather than source-level analysis. Simpler model, but you must explicitly declare task dependencies.

**Caching:**

- Both support local file system caching and remote caching (Nx Cloud for Nx, Vercel Remote Cache for Turborepo). The core concept is the same: hash inputs (source files, dependencies, environment variables) and skip execution if the output is already cached.
- **Nx** has finer-grained cache input configuration and source-level dependency tracking, which means it can more accurately determine what's affected. It also supports distributed task execution (DTE) where CI tasks are spread across multiple agents.
- **Turborepo** uses file globs and environment variable lists to define cache inputs. Simpler to configure but requires more manual specification of what affects the cache key.

**Dependency graph:**

- **Nx** analyzes actual source code imports to build the dependency graph. If app A imports from lib B, Nx knows — even without an explicit `package.json` dependency. This powers accurate affected detection.
- **Turborepo** relies on the `package.json` workspace dependency declarations. If you forget to declare a dependency, Turborepo won't know about it, and affected detection can miss changes.

**Plugin ecosystem:**

- **Nx** has a rich plugin system with first-party plugins for React, Angular, Next.js, Node, and more. Plugins provide generators (scaffolding), executors (build/test commands), and migrations. This is powerful but adds complexity and coupling to the Nx ecosystem.
- **Turborepo** is intentionally minimal — it orchestrates scripts already defined in `package.json`. No code generation, no custom executors. You bring your own tooling. Simpler mental model but more manual setup.

**Decision factors:**

| Factor | Choose Nx | Choose Turborepo |
|--------|-----------|-----------------|
| Team prefers convention + scaffolding | Yes — generators save time | No — too opinionated |
| Existing tooling is custom | No — plugins may conflict | Yes — works with any setup |
| Need accurate affected detection | Yes — source-level analysis | Adequate for most cases |
| Team size is small, wants simplicity | Overkill | Good fit |
| Need distributed CI execution | Yes — Nx Cloud DTE | Not natively supported |
| Angular project | Yes — first-class support | Possible but no special support |

For most teams starting a new React/TypeScript monorepo, Turborepo is the simpler starting point. Choose Nx when you need its graph intelligence, generators, or distributed task execution.

</details>

<details>
<summary>4. What are the main micro-frontend composition strategies (Module Federation, iframes, server-side composition) and what are the real tradeoffs of each — why is micro-frontend architecture controversial, when does the complexity pay off, and when is it architectural astronautics that a team will regret?</summary>

**Composition strategies:**

**Module Federation (webpack/Vite):**
- A host app loads remote bundles at runtime. Remotes expose modules; the host declares what it consumes.
- Shared dependencies (React, etc.) can be configured to load once, avoiding duplication.
- **Pros**: Seamless UX (feels like one app), shared dependencies reduce bundle size, TypeScript type sharing is possible.
- **Tradeoffs**: Tight coupling to bundler configuration, shared dependency version mismatches cause subtle runtime errors, debugging across host/remote boundaries is painful, and deployment order matters (a broken remote can break the host).

**Iframes:**
- Each micro-frontend is a fully independent app loaded in an iframe.
- **Pros**: Complete isolation — different frameworks, different versions, different build tools. One app crashing can't break another. Security boundaries are real.
- **Tradeoffs**: Communication between iframes is limited to `postMessage` (slow, error-prone). No shared styling, so achieving consistent look-and-feel requires duplicating the design system. Navigation, deep linking, and browser history are difficult. Performance overhead from loading separate apps.

**Server-side composition:**
- A server (or edge function) stitches HTML fragments from different micro-frontends before sending to the client. Used by frameworks like Podium or custom setups with ESI (Edge Side Includes).
- **Pros**: Fast initial load (server-rendered HTML), works without JavaScript, good for content-heavy sites.
- **Tradeoffs**: Client-side interactivity requires hydration coordination. More infrastructure complexity (a composition layer that must be highly available). Harder to develop locally — you need the composition server running.

**Why it's controversial:**

Micro-frontends solve an organizational problem (team independence) at the cost of technical complexity. The promise is that teams can deploy independently, use different tech stacks, and move at their own pace. The reality is:

- **Shared dependency management** is still hard. If all micro-frontends use React, you're managing version alignment across teams anyway.
- **UX consistency** requires a shared design system regardless of architecture — micro-frontends don't eliminate that problem.
- **The composition layer** becomes a critical piece of infrastructure that someone must own.
- **Performance** often suffers: duplicate framework code, extra network requests, hydration complexity.

**When the complexity pays off:**

- Very large organizations (50+ frontend developers) where team autonomy genuinely matters more than technical efficiency.
- Gradual migration from a legacy app (e.g., migrating from Angular to React incrementally — the old and new coexist via composition).
- Products that are genuinely composed of independent domains (e.g., an e-commerce site where checkout, catalog, and account management have different release cadences and teams).

**When it's architectural astronautics:**

- Teams under 20 developers who could share a monorepo.
- When the "independence" is theoretical — if teams deploy on the same schedule and share most of their code, micro-frontends add complexity without real benefit.
- When chosen because "Netflix does it" without having Netflix's organizational constraints.

A monorepo with proper module boundaries gives you most of the code isolation benefits without the runtime composition complexity.

</details>

<details>
<summary>5. How should a design system be architecturally layered — what is the role of design tokens, primitive components, and composed components, how do these layers relate to each other, and what are the tradeoffs of a thin design system (tokens + primitives only) vs a thick one (fully composed, opinionated components)?</summary>

**The three layers:**

**Design tokens** are the atomic values: colors, spacing, typography scales, border radii, shadows, breakpoints. They encode design decisions as named constants — `color.primary.500`, `spacing.md`, `font.size.lg`. Tokens are framework-agnostic (usually JSON/CSS custom properties) and are the source of truth that everything else references.

**Primitive components** are the basic building blocks: Button, Input, Select, Modal, Tooltip, Typography. They implement tokens into interactive UI elements with consistent behavior (keyboard navigation, aria attributes, focus management). Primitives are intentionally unopinionated about layout and composition — a Button doesn't know if it's in a form or a toolbar.

**Composed components** are higher-level patterns assembled from primitives: LoginForm, DataTable, SearchBar, NavigationHeader. They encode business-level UI patterns with opinionated layout, validation behavior, and data flow.

**How layers relate:**

```
Tokens → Primitives → Composed Components → Application UI
```

Each layer only depends on the layer below. Tokens have zero dependencies. Primitives consume tokens. Composed components consume primitives and tokens. Application code consumes any layer.

**Thin design system (tokens + primitives only):**

- **Pros**: Maximum flexibility for consuming teams. Each team composes primitives however their domain requires. Fewer breaking changes — primitives have stable, minimal APIs. Easier to maintain with a small design system team.
- **Cons**: Teams duplicate composition patterns. Five teams build five slightly different data tables. Consistency at the pattern level requires discipline, not enforcement.
- **Best for**: Organizations where teams have diverse UI needs and the overhead of standardizing composed patterns is higher than the inconsistency cost.

**Thick design system (fully composed, opinionated components):**

- **Pros**: Strong consistency — a DataTable looks and behaves the same everywhere. Less code for consuming teams to write. Patterns like pagination, sorting, and filtering are solved once.
- **Cons**: Opinionated components don't fit every use case. Teams fight the design system when their needs diverge. More breaking changes because composed components have larger API surfaces. The design system team becomes a bottleneck — every new pattern requires their involvement.
- **Best for**: Organizations building a product family with consistent UX where the cost of divergence is high (e.g., an admin console with 20 similar CRUD views).

**Practical recommendation**: Start thin. Tokens + primitives cover 80% of needs with 20% of the maintenance burden. When you see teams repeatedly building the same composed pattern (DataTable, FormLayout), graduate those specific patterns to the design system. Don't preemptively compose — let real usage drive what gets elevated.

</details>

<details>
<summary>6. How do you handle design system versioning and governance at scale — what versioning strategy prevents consuming teams from breaking on updates, what governance model balances consistency with team autonomy, and what are the signs that a design system has become a bottleneck vs drifted into inconsistency?</summary>

**Versioning strategy:**

Semantic versioning (semver) is the foundation, but the execution details matter:

- **Use changesets** (tools like `@changesets/cli`) so every PR to the design system includes a changeset describing what changed and whether it's a patch, minor, or major. This makes versioning a deliberate decision, not an afterthought.
- **Avoid frequent major versions** — every major version requires migration effort from consuming teams. Design the API to be additive: new props with defaults, new components alongside old ones, deprecation warnings before removal.
- **Provide codemods** for breaking changes. If you rename a prop from `variant="primary"` to `variant="brand"`, ship a codemod that consuming teams can run to auto-migrate. This dramatically reduces the friction of upgrading.
- **Support N-1 versions** — maintain bug fixes for the previous major version for a defined period (e.g., 3-6 months). Teams shouldn't be forced to upgrade immediately.
- **Pin to minor ranges** in consuming apps (`^2.3.0`) so teams get patches and new features automatically but can opt in to majors.

**Governance model:**

- **Inner-source model**: The design system team owns core tokens and primitives, but accepting contributions from product teams for composed components. PRs go through design system team review for API consistency and accessibility compliance.
- **RFC process for new patterns**: Before building a new composed component, teams write a short RFC explaining the use case, proposed API, and which existing teams would consume it. This prevents building patterns nobody uses and ensures input from consumers.
- **Office hours / design system council**: Regular sync (biweekly) where consuming teams raise pain points, request new components, and surface inconsistencies. This prevents the design system from diverging from real needs.
- **Automated enforcement**: Visual regression testing (Chromatic, Percy) catches unintended visual changes. Lint rules can flag deprecated component usage. A dashboard showing adoption metrics (which teams are on which version, which components are most used) keeps governance data-driven.

**Signs it's a bottleneck:**

- Consuming teams are blocked waiting for design system team to review PRs or build new components.
- Teams are wrapping design system components with extensive overrides, essentially building a shadow design system on top.
- Version upgrades are dreaded — they take days, break things, and teams defer them indefinitely.
- The design system backlog grows faster than it shrinks.

**Signs of inconsistency drift:**

- Teams are building their own Button, Modal, or Input instead of using the design system — they've given up on it fitting their needs.
- The same UI pattern looks noticeably different across different parts of the product.
- Token usage is low — teams are hardcoding colors and spacing values instead of referencing tokens.
- No team tracks or enforces design system adoption metrics.

</details>

<details>
<summary>7. How do Vite and webpack differ architecturally in their dev server approach and production build pipeline — why is Vite's native ESM dev server faster, what does webpack still do better for certain use cases, and what are the real migration tradeoffs when moving an existing webpack project to Vite?</summary>

**Dev server architecture:**

**Webpack** bundles the entire application before serving it to the browser, even in development. When you change a file, webpack rebuilds the affected module(s) and sends an HMR update. For large apps, the initial startup can take 30-60 seconds because the entire dependency graph must be compiled.

**Vite** uses the browser's native ES module support. On dev server start, it only transforms the entry point. When the browser requests a module via `import`, Vite transforms that specific file on demand. Dependencies (node_modules) are pre-bundled once with esbuild (which is 10-100x faster than webpack's JavaScript-based bundling). The result: server starts in under a second regardless of app size.

**Why Vite is faster in dev:**
- No upfront bundling — only transforms files the browser actually requests.
- Pre-bundling dependencies with esbuild (Go-based, natively compiled) instead of JavaScript-based bundling.
- HMR only invalidates the changed module and its direct importers, not the full bundle graph.

**Production builds:**

- **Vite** uses Rollup (or the newer Rolldown) for production. Rollup produces highly optimized bundles with excellent tree-shaking.
- **Webpack** uses its own bundler. It has more mature code splitting, especially for complex dynamic import patterns and shared chunks across multiple entry points.

**What webpack still does better:**

- **Module Federation** — webpack invented it, and the integration is mature. Vite has plugins for it, but they're less battle-tested.
- **Complex legacy setups** — webpack's loader system handles edge cases (importing CSS from node_modules, non-standard module formats) that Vite's plugin system may handle differently or not at all.
- **Extremely large apps with complex code splitting** — webpack's `SplitChunksPlugin` has more granular control than Rollup's `manualChunks`.

**Migration tradeoffs:**

| Aspect | Straightforward | Requires workarounds |
|--------|----------------|---------------------|
| Basic React/TypeScript app | Drop-in replacement | — |
| Environment variables | Rename `REACT_APP_*` to `VITE_*`, use `import.meta.env` instead of `process.env` | — |
| CSS Modules | Works out of the box | — |
| Custom webpack loaders | — | Must find Vite/Rollup plugin equivalents or write custom plugins |
| `require()` / CommonJS | — | Vite is ESM-first; CJS requires plugin or refactoring |
| Proxy configuration | Similar `server.proxy` config | — |
| Module Federation | — | Less mature plugin ecosystem |
| Jest config relying on webpack transforms | — | Switch to Vitest (significant but usually beneficial) |

The biggest real-world migration cost isn't the config rewrite — it's finding that 5% of your code relies on webpack-specific behaviors (raw loaders, specific resolve behaviors, or CommonJS patterns) that need individual attention.

</details>

<details>
<summary>8. Why do large frontend codebases need explicit module boundaries and dependency rules — what happens without them, what enforcement strategies exist (ESLint rules, Nx module boundaries, TypeScript path restrictions, workspace constraints), and how do you design boundaries that match team ownership without creating excessive coupling or duplication?</summary>

**What happens without explicit boundaries:**

In a monorepo with 10 apps and 30 shared libraries, developers take the path of least resistance. An app developer needs a utility function that exists in another app — they import it directly. Over months, you end up with a tangled dependency graph where:

- Changing any file potentially affects any other project (no accurate "affected" detection).
- Circular dependencies emerge between apps and libraries.
- A shared library change triggers rebuilds across the entire monorepo.
- Nobody can confidently refactor because the blast radius is unknown.
- Team ownership becomes meaningless — "your" code depends on "their" internals.

**Enforcement strategies:**

**Nx module boundary rules** (`@nx/enforce-module-boundaries`):

Projects are tagged in `project.json`, and ESLint rules enforce which tags can depend on which:

```javascript
// eslint.config.mjs
import nx from '@nx/eslint-plugin';

export default [
  ...nx.configs['flat/base'],
  ...nx.configs['flat/typescript'],
  {
    files: ['**/*.ts', '**/*.tsx'],
    rules: {
      '@nx/enforce-module-boundaries': [
        'error',
        {
          depConstraints: [
            {
              sourceTag: 'type:app',
              onlyDependOnLibsWithTags: ['type:feature', 'type:ui', 'type:util'],
            },
            {
              sourceTag: 'type:feature',
              onlyDependOnLibsWithTags: ['type:ui', 'type:util'],
            },
            {
              sourceTag: 'type:ui',
              onlyDependOnLibsWithTags: ['type:ui', 'type:util'],
            },
            {
              sourceTag: 'type:util',
              onlyDependOnLibsWithTags: ['type:util'],
            },
          ],
        },
      ],
    },
  },
];
```

This prevents apps from importing from other apps, and ensures utility libraries don't depend on feature libraries.

**ESLint import restrictions** (for Turborepo or custom setups):

```javascript
// eslint.config.mjs
{
  rules: {
    'no-restricted-imports': ['error', {
      patterns: [
        { group: ['@myorg/app-*'], message: 'Apps cannot import from other apps.' },
        { group: ['../**/src/*'], message: 'Import from package entry points, not internal files.' },
      ],
    }],
  },
}
```

**TypeScript path restrictions**: Configure `tsconfig.json` paths to only expose public APIs of libraries. Internal files aren't reachable through path mappings.

**Workspace constraints** (pnpm, Yarn): `pnpm` workspace protocol and Yarn constraints can enforce which packages can depend on which at the package manager level.

**Designing boundaries that match ownership:**

- **Vertical slicing by domain**: Each team owns feature libraries for their domain (checkout, catalog, account) plus app shells. Shared utilities and UI components are in cross-team libraries with clear ownership.
- **Layer-based tags**: `type:app` > `type:feature` > `type:ui` > `type:util`. Dependencies only flow downward.
- **Scope-based tags**: `scope:checkout`, `scope:shared`. Combine with type tags for two-dimensional constraints.
- **Avoid premature extraction**: Don't create a shared library until two or more teams genuinely need the same code. Duplication is cheaper than wrong abstraction.
- **Public API enforcement**: Each library exports through an `index.ts` barrel. TypeScript paths point to the barrel, not internal files. This lets teams refactor internals without breaking consumers.

</details>

<details>
<summary>9. How should you balance unit tests, integration tests, and E2E tests in a large frontend codebase — why has the "testing trophy" replaced the "testing pyramid" for frontend, what does each layer of the trophy actually test, and what testing strategy gives the best confidence-to-maintenance-cost ratio at scale?</summary>

**Why the testing trophy replaced the pyramid:**

The traditional testing pyramid (many unit tests, fewer integration tests, few E2E tests) was designed for backend systems where units of business logic are well-isolated functions. In frontend applications, the unit of behavior is often a component interacting with the DOM, handling user events, and coordinating with other components. Testing a React component's internal state management in isolation tells you very little about whether the feature actually works.

The **testing trophy** (coined by Kent C. Dodds) reshapes the distribution:

```
        /  E2E  \        ← few, critical user journeys
       / Integration \    ← bulk of tests (components + interactions)
      /    Unit        \  ← pure logic, utilities, complex algorithms
     /   Static Types   \ ← TypeScript catches a whole class of bugs
```

**What each layer tests:**

- **Static analysis (TypeScript + ESLint)**: Catches type errors, unused variables, incorrect imports. This is the base — it's always running, zero maintenance cost, and catches a surprising number of bugs that would otherwise be unit tests.
- **Unit tests**: Pure functions, utility libraries, complex business logic (calculations, transformers, validators). Things with clear inputs and outputs, no DOM, no side effects.
- **Integration tests**: The main layer. A component rendered with Testing Library, interacting with it like a user would — clicking buttons, filling forms, verifying output. May include mocked API calls. Tests the component's behavior, not its implementation.
- **E2E tests**: Full user journeys through the real application: sign up, add to cart, checkout. Tests the entire stack including API, database, and real browser behavior. Expensive to run and maintain.

**Strategy for best confidence-to-cost ratio:**

- **Lean heavily on integration tests** (60-70% of effort). Test components the way users use them. Use Testing Library's queries (`getByRole`, `getByText`) that mirror accessibility — if your test can't find the button, a screen reader can't either.
- **Unit test complex logic** (15-20%). Parsers, formatters, state machines, calculation engines. If it's a pure function with edge cases, unit test it.
- **E2E for critical paths only** (10-15%). Login, checkout, core workflow. Not every feature — just the ones where a failure means lost revenue or broken access. Keep E2E tests focused on happy paths and the most important error paths.
- **Let TypeScript do the rest**. Strict mode with `noUncheckedIndexedAccess` catches nullable access, exhaustive switch checks catch missing cases. This replaces dozens of unit tests.

**At scale**, the biggest cost isn't writing tests — it's maintaining them. Integration tests are more resilient to refactoring than unit tests (they test behavior, not implementation) and more stable than E2E tests (no network flakiness, no timing issues). This is why the trophy's middle layer should be thickest.

</details>

<details>
<summary>10. How do Vitest, Testing Library, and Playwright each fit into the frontend testing stack — what layer does each tool own, where do their responsibilities overlap, and what are the tradeoffs of standardizing on this stack vs alternatives (Jest, Cypress, WebdriverIO)?</summary>

**What each tool owns:**

- **Vitest** is the test runner and assertion library. It runs tests, manages test lifecycle (setup, teardown, mocking), provides `describe`/`it`/`expect`, and reports results. It handles the unit and integration test layers. Vitest is Vite-native, meaning it uses the same transform pipeline as your dev server — no separate babel/webpack config for tests.

- **Testing Library** (`@testing-library/react`) is the rendering and interaction layer for integration tests. It renders React components into a simulated DOM (jsdom or happy-dom) and provides user-centric queries (`getByRole`, `getByText`, `getByLabelText`) and interaction helpers (`userEvent.click`, `userEvent.type`). It's not a test runner — it needs Vitest (or Jest) to execute.

- **Playwright** is the E2E testing framework. It launches real browsers (Chromium, Firefox, WebKit), navigates to your actual running application, and interacts with it like a real user. It has its own test runner (`@playwright/test`), assertions, and reporting.

**Where they overlap:**

- Vitest + Testing Library vs Playwright: Both can test "does clicking this button show a modal." The distinction is scope — Testing Library tests the component in isolation (fast, no real browser), Playwright tests it in the full application (slow, real browser, real API).
- Playwright has component testing capabilities (`@playwright/experimental-ct-react`) that overlap with Testing Library's domain, but it's less mature.

**Tradeoffs vs alternatives:**

| Stack | Vitest + Testing Library + Playwright | Jest + Testing Library + Cypress |
|-------|--------------------------------------|----------------------------------|
| Test runner speed | Vitest is significantly faster (native ESM, Vite transforms) | Jest is slower, especially with TypeScript (requires ts-jest or babel) |
| Config alignment | Vitest shares Vite config — no duplicate transform setup | Jest needs separate transform config |
| Browser coverage | Playwright tests across Chromium, Firefox, WebKit | Cypress runs in Chromium only (Firefox experimental) |
| E2E DX | Playwright's auto-waiting is robust, trace viewer is excellent | Cypress has better interactive debugging UI, time-travel snapshots |
| Network mocking | Playwright's `route` API intercepts at browser level | Cypress's `cy.intercept` is similar but more intuitive for simple cases |
| Community/ecosystem | Growing rapidly, newer | Larger existing ecosystem, more tutorials/plugins |
| CI performance | Playwright parallelizes natively across workers | Cypress parallelization requires Cypress Cloud (paid) |

**WebdriverIO** is still relevant for cross-browser testing that needs Selenium grid infrastructure or mobile testing (via Appium), but for web-only testing, Playwright has largely superseded it.

**Recommendation**: The Vitest + Testing Library + Playwright stack is the modern default for React/TypeScript projects. The main reason to stay on Jest is if you have a large existing test suite and the migration cost isn't justified. The main reason to choose Cypress over Playwright is if your team highly values the interactive debugging experience and doesn't need WebKit testing.

</details>

<details>
<summary>11. Why are E2E tests disproportionately flaky compared to unit and integration tests — what are the main categories of flakiness (timing/race conditions, test isolation, environment differences, third-party dependencies), what stabilization patterns actually work vs just masking the problem, and when should you delete a flaky test instead of fixing it?</summary>

**Why E2E tests are inherently flakier:**

E2E tests interact with the full stack — browser rendering, JavaScript execution, network requests, API servers, databases. Each layer introduces non-determinism. A unit test has one source of variability (the code). An E2E test has dozens.

**Categories of flakiness:**

**1. Timing / race conditions:**
- Assertions run before the UI has finished updating. The test checks for text that hasn't rendered yet, an animation that hasn't completed, or data that hasn't loaded.
- **Stabilization that works**: Use auto-waiting assertions (Playwright's `expect(locator).toHaveText()` retries automatically). Never use `sleep()` or fixed timeouts. Wait for specific conditions, not arbitrary time.
- **Masking the problem**: Adding `page.waitForTimeout(2000)` — this "fixes" it on your machine but fails on slower CI runners.

**2. Test isolation failures:**
- Tests share state: a previous test leaves data in the database, a cookie persists, or a browser session isn't properly reset. Test order matters — running tests in a different order or in parallel produces different results.
- **Stabilization that works**: Each test sets up its own data (API calls or database seeding), runs in a fresh browser context, and cleans up after itself. Playwright's `test.describe` with `beforeEach` creating fresh contexts handles this well.
- **Masking the problem**: Running tests serially in a fixed order — this "works" until someone adds a new test or CI parallelizes.

**3. Environment differences:**
- The test passes locally but fails in CI because of different screen resolution, font rendering, timezone, locale, or available system resources (slower CPU means more timing issues).
- **Stabilization that works**: Run the same Docker image locally and in CI. Set explicit viewport sizes, timezones, and locales in Playwright config. Use headless mode consistently.

**4. Third-party dependencies:**
- Tests interact with external APIs (payment providers, auth services, analytics) that are slow, rate-limited, or occasionally down.
- **Stabilization that works**: Mock external services at the network level. Playwright's `page.route()` intercepts HTTP requests and returns deterministic responses. Only hit real external services in a separate, less frequent smoke test suite.

**5. Resource contention:**
- CI runners under heavy load, shared test databases, or parallel tests competing for the same resources.
- **Stabilization that works**: Isolate test databases per worker (each worker gets a unique schema/prefix), use Playwright's worker-scoped fixtures for shared resources.

**When to delete instead of fix:**

- The test covers a non-critical path and has failed intermittently for weeks without anyone investigating — the signal-to-noise ratio is negative.
- The test is a duplicate of integration tests that already cover the same logic more reliably.
- The cost of stabilizing it (mocking complex third-party flows, rewriting test data setup) exceeds the confidence it provides.
- The feature it tests is being deprecated or redesigned.

**Rule of thumb**: If a test has been quarantined (marked as skip/flaky) for more than 2 weeks without someone actively fixing it, delete it. A skipped test provides zero value and creates a false sense of coverage.

</details>

<details>
<summary>12. How should you handle cross-app state and data sharing in a frontend monorepo — where should the boundaries be between shared state and app-local state, how does the distinction between server state (React Query, SWR) and client state (Zustand, Redux) affect architecture decisions, and what are the tradeoffs of API layer patterns like a shared SDK vs GraphQL federation for data access across apps?</summary>

**Boundaries between shared and app-local state:**

The default should be **app-local**. Each app owns its own state. Sharing state between apps should be the exception, not the rule, because shared state creates coupling — changing the shape of shared state requires coordinating across teams.

State should be shared only when:
- Multiple apps need to read the same data and it must be consistent (e.g., the current user's profile, feature flags, permissions).
- It represents a cross-cutting concern (theme, locale, authentication status).

Everything else — form state, UI state (sidebar open/closed), page-specific data — stays app-local.

**Server state vs client state at the architecture level:**

**Server state** (React Query, SWR) is data that lives on the server and is cached on the client. It has natural characteristics: it can be stale, it needs background refetching, multiple components might request the same data.

**Client state** (Zustand, Redux, Context) is data that originates and lives on the client: UI state, form drafts, client-side preferences.

Architecturally this distinction matters because:

- **Server state doesn't need to be "shared" explicitly** — if two apps both query `/api/user/me` via React Query with the same query key, they each maintain their own cache. You don't need shared global state; you need a shared data-fetching library.
- **Client state that crosses app boundaries** is a smell. If app A needs to know about app B's UI state, either the boundary is wrong (they should be one app) or you need an event-based communication pattern, not shared state.

**API layer patterns:**

**Shared SDK (typed API client):**
- A shared library (`@myorg/api-client`) wraps your REST/GraphQL APIs with typed functions: `getUser(id)`, `listOrders(filters)`.
- **Pros**: Type safety across all apps, single place to update when APIs change, consistent error handling and auth token injection.
- **Cons**: The SDK becomes a coupling point — every API change requires updating the shared library and potentially all consumers. Risk of the SDK growing into a "god object" that everyone depends on.
- **Best for**: REST APIs consumed by multiple apps in the same monorepo.

**GraphQL federation:**
- Multiple backend services expose their own GraphQL schemas. A gateway federates them into a single unified schema. Frontend apps query one endpoint and get exactly the data they need.
- **Pros**: Apps request exactly what they need (no over-fetching), schema stitching is handled at the gateway, strong typing via codegen.
- **Cons**: Requires GraphQL infrastructure (gateway, schema registry). Query complexity and N+1 problems at the gateway level. Learning curve if the team isn't already using GraphQL.
- **Best for**: Large organizations with multiple backend services and frontend apps that need flexible data composition.

**Practical recommendation**: For most monorepos, a shared typed API client (generated from OpenAPI specs or GraphQL codegen) combined with React Query for caching is the sweet spot. It gives you type safety, caching, and background refetching without the infrastructure overhead of GraphQL federation.

</details>

<details>
<summary>13. What are the key deployment architecture decisions for a frontend application — when should you choose static hosting vs SSR, how do CDN caching and cache invalidation strategies differ between the two, and why is the choice between injecting environment variables at build time vs runtime an architectural decision that is hard to change later?</summary>

**Static hosting vs SSR:**

**Static hosting** (S3 + CloudFront, Vercel static, Netlify): The build produces HTML, JS, and CSS files. A CDN serves them directly. No server-side processing at request time.

- **Choose when**: The app is a SPA, content doesn't need to be personalized on first render, SEO is either not important or handled by pre-rendering. Most admin dashboards, internal tools, and authenticated apps fit here.
- **Advantages**: Simple infrastructure, infinitely scalable (CDN handles traffic), cheap, no server to manage.

**SSR** (Next.js, Remix, custom Node server): The server renders HTML on each request (or for specific routes), then sends it to the client for hydration.

- **Choose when**: SEO matters (marketing sites, e-commerce product pages), first contentful paint must be fast for unauthenticated users, content is dynamic per request (personalization, A/B testing at the edge).
- **Advantages**: Better initial load performance, SEO-friendly, can personalize per request.
- **Cost**: Server infrastructure, scaling concerns, higher complexity, cold start latency for serverless SSR.

**CDN caching differences:**

**Static hosting**: All assets are immutable — they have content hashes in filenames (`main.a3f2b1.js`). Set `Cache-Control: max-age=31536000, immutable` on all hashed assets. The `index.html` is the only mutable file — cache it briefly (`max-age=60`) or not at all, so users always get the latest asset references. Invalidation is rarely needed because new deploys produce new filenames.

**SSR**: The CDN caches rendered HTML pages. Cache keys include the URL path and potentially headers (cookies, accept-language). Invalidation is more complex:
- **Time-based** (`s-maxage=60`): Pages are stale after a set time. Simple but means stale content for up to that duration.
- **Tag-based invalidation**: Tag cached pages by content type (e.g., `product:123`). When product 123 is updated, purge all pages tagged with it. Requires CDN support (Fastly, CloudFront with Lambda@Edge).
- **Stale-while-revalidate**: Serve stale content immediately while fetching fresh content in the background. Best UX tradeoff for most cases.

**Environment variables: build time vs runtime:**

**Build time injection** (`VITE_API_URL` baked into the JS bundle via `import.meta.env`):
- The value is compiled into the JavaScript as a string literal. The bundle for staging and production is different because the environment variable is different.
- **Problem**: You can't promote the same artifact across environments. The staging build is literally different code than the production build. This means you're not testing what you deploy.

**Runtime injection** (fetching config from an endpoint or injecting via `window.__CONFIG__` in the HTML):

```html
<!-- index.html served by a thin server or edge function -->
<script>
  window.__CONFIG__ = {
    API_URL: "%%API_URL%%",  // replaced at serve time
  };
</script>
```

```typescript
// config.ts
export const config = {
  apiUrl: window.__CONFIG__?.API_URL ?? 'http://localhost:3000',
};
```

- The same build artifact works in every environment. Promote from staging to production by changing the config, not rebuilding.
- **Cost**: Slightly more setup (a config endpoint or template replacement at serve time). Every config access goes through the config object instead of `import.meta.env`.

**Why it's hard to change later**: If you start with build-time injection, `import.meta.env.VITE_*` references are scattered across the entire codebase. Migrating to runtime injection means finding and replacing every reference, introducing a config module, and changing your deployment pipeline. The longer you wait, the more references exist. Choose runtime injection from the start if you anticipate multiple environments or artifact promotion.

</details>

## Practical — Configuration & Implementation

<details>
<summary>14. Set up an Nx or Turborepo monorepo with multiple frontend apps and shared libraries — show the workspace configuration, demonstrate affected-only test execution and remote caching setup, and explain what configuration mistakes lead to cache misses that silently defeat the performance benefits.</summary>

Here's a Turborepo setup (simpler starting point) with two apps and shared libraries:

**Directory structure:**
```
my-monorepo/
  apps/
    web/              # Main customer-facing app
      package.json
    admin/            # Internal admin dashboard
      package.json
  packages/
    ui/               # Shared design system components
      package.json
    utils/            # Shared utilities
      package.json
    tsconfig/         # Shared TypeScript configs
      package.json
  turbo.json
  package.json
  pnpm-workspace.yaml
```

**Root `package.json`:**
```json
{
  "name": "my-monorepo",
  "private": true,
  "scripts": {
    "build": "turbo build",
    "test": "turbo test",
    "lint": "turbo lint",
    "dev": "turbo dev"
  },
  "devDependencies": {
    "turbo": "^2.0.0"
  }
}
```

**`pnpm-workspace.yaml`:**
```yaml
packages:
  - "apps/*"
  - "packages/*"
```

**`turbo.json`:**
```json
{
  "$schema": "https://turbo.build/schema.json",
  "globalDependencies": ["**/.env.*local"],
  "globalEnv": ["NODE_ENV", "CI"],
  "tasks": {
    "build": {
      "dependsOn": ["^build"],
      "inputs": ["src/**", "tsconfig.json", "vite.config.ts"],
      "outputs": ["dist/**", ".next/**"],
      "env": ["VITE_API_URL"]
    },
    "test": {
      "dependsOn": ["^build"],
      "inputs": ["src/**", "test/**", "vitest.config.ts"],
      "outputs": ["coverage/**"]
    },
    "lint": {
      "dependsOn": ["^build"],
      "inputs": ["src/**", "eslint.config.mjs"]
    },
    "dev": {
      "cache": false,
      "persistent": true
    }
  }
}
```

**App `package.json` (e.g., `apps/web/package.json`):**
```json
{
  "name": "@myorg/web",
  "private": true,
  "dependencies": {
    "@myorg/ui": "workspace:*",
    "@myorg/utils": "workspace:*",
    "react": "^18.3.0"
  }
}
```

**Affected-only execution:**

Turborepo compares the current state against a base commit to determine which packages changed:

```bash
# Run tests only for packages affected since main
turbo test --filter="...[origin/main]"

# In CI (GitHub Actions):
- name: Test affected
  run: npx turbo test --filter="...[HEAD~1]"
```

**Remote caching setup:**

```bash
# Link to Vercel Remote Cache
npx turbo login
npx turbo link

# Or self-hosted with environment variables
TURBO_TOKEN=your-token
TURBO_TEAM=your-team
TURBO_API=https://your-cache-server.com
```

In CI:
```yaml
# .github/workflows/ci.yml
env:
  TURBO_TOKEN: ${{ secrets.TURBO_TOKEN }}
  TURBO_TEAM: ${{ secrets.TURBO_TEAM }}

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0  # Full history for affected detection
      - uses: pnpm/action-setup@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'pnpm'
      - run: pnpm install --frozen-lockfile
      - run: pnpm turbo build test lint --filter="...[origin/main]"
```

**Configuration mistakes that cause cache misses:**

1. **Missing `inputs` declarations**: Without explicit `inputs`, Turborepo hashes all files in the package. Changing a README invalidates the cache. Always specify which files affect the output.

2. **Unlisted environment variables**: If `VITE_API_URL` affects the build output but isn't listed in `env` or `globalEnv`, the cache key doesn't include it. You get a cache hit that serves the wrong environment's build.

3. **Non-deterministic outputs**: If your build includes timestamps, random IDs, or git SHAs in the output, the cache stores one version but the next build produces different output — the cache hit serves stale content.

4. **Missing `dependsOn: ["^build"]`**: If a task depends on built output from dependencies but doesn't declare `^build`, Turborepo may run it before dependencies are built. The task fails or uses stale output, and caching captures the broken state.

5. **Shallow git clones**: `fetch-depth: 1` in CI means Turborepo can't compare against `origin/main` — everything looks "affected." Use `fetch-depth: 0` or at least enough depth to include the merge base.

</details>

<details>
<summary>15. Configure Module Federation for a micro-frontend setup where a host app loads remote modules at runtime — show the webpack or Vite plugin configuration for both host and remote, explain how shared dependencies (React, shared state) are handled to avoid duplication, and what breaks when version mismatches occur between host and remote.</summary>

**webpack Module Federation setup:**

**Remote app (`remote/webpack.config.js`):**
```javascript
const { ModuleFederationPlugin } = require('webpack').container;

module.exports = {
  output: {
    publicPath: 'http://localhost:3001/', // Where the remote is served
    uniqueName: 'remoteApp',
  },
  plugins: [
    new ModuleFederationPlugin({
      name: 'remoteApp',
      filename: 'remoteEntry.js', // The manifest file the host loads
      exposes: {
        './ProductList': './src/components/ProductList',
        './CartWidget': './src/components/CartWidget',
      },
      shared: {
        react: {
          singleton: true,        // Only one instance of React
          requiredVersion: '^18.3.0',
          eager: false,           // Loaded asynchronously
        },
        'react-dom': {
          singleton: true,
          requiredVersion: '^18.3.0',
        },
        '@myorg/design-system': {
          singleton: true,
          requiredVersion: '^2.0.0',
        },
      },
    }),
  ],
};
```

**Host app (`host/webpack.config.js`):**
```javascript
const { ModuleFederationPlugin } = require('webpack').container;

module.exports = {
  plugins: [
    new ModuleFederationPlugin({
      name: 'hostApp',
      remotes: {
        // Load remote at runtime from this URL
        remoteApp: 'remoteApp@http://localhost:3001/remoteEntry.js',
      },
      shared: {
        react: {
          singleton: true,
          requiredVersion: '^18.3.0',
          eager: true, // Host loads React eagerly since it's the shell
        },
        'react-dom': {
          singleton: true,
          requiredVersion: '^18.3.0',
          eager: true,
        },
        '@myorg/design-system': {
          singleton: true,
          requiredVersion: '^2.0.0',
        },
      },
    }),
  ],
};
```

**Using the remote in the host:**

```typescript
// host/src/bootstrap.tsx (async boundary required)
import React, { Suspense, lazy } from 'react';

// Dynamic import — loaded at runtime from the remote
const ProductList = lazy(() => import('remoteApp/ProductList'));

export function App() {
  return (
    <div>
      <h1>Host Application</h1>
      <Suspense fallback={<div>Loading products...</div>}>
        <ProductList />
      </Suspense>
    </div>
  );
}
```

```typescript
// host/src/index.ts — must be dynamic to allow shared module negotiation
import('./bootstrap');
```

**Type safety for remotes:**

```typescript
// host/src/types/remoteApp.d.ts
declare module 'remoteApp/ProductList' {
  const ProductList: React.ComponentType<{ category?: string }>;
  export default ProductList;
}
```

**How shared dependencies work:**

Module Federation's `shared` config creates a negotiation protocol at runtime:
1. When the host loads, it registers its version of React in a shared scope.
2. When the remote loads, it checks the shared scope. If a compatible version exists (satisfies `requiredVersion`), it uses the host's React instead of bundling its own.
3. `singleton: true` ensures only one instance exists — critical for React (two React instances cause hooks to break).

**What breaks with version mismatches:**

**Scenario 1: Minor version mismatch (React 18.2 vs 18.3, `singleton: true`)**
- With `requiredVersion: '^18.2.0'` on the remote and the host providing 18.3.0, the version satisfies the range. The remote uses the host's React. Usually works fine.

**Scenario 2: Major version mismatch (React 17 vs 18, `singleton: true`)**
- The remote requires React 17, host has React 18. Module Federation logs a warning and forces the singleton (host's version). The remote's code, written for React 17, runs against React 18 APIs. Result: subtle runtime errors — hooks behaving differently, deprecated APIs throwing, or silent rendering bugs.

**Scenario 3: Singleton not set, both bundle React**
- Two React instances load. Components from the remote use remote's React; host uses host's React. When you pass state or context across the boundary, it breaks — `useContext` from one React instance can't read context from another. Hooks throw "Invalid hook call" errors.

**Scenario 4: Shared library with internal state (design system with ThemeProvider)**
- If the design system isn't marked as singleton, the host and remote each get their own ThemeProvider. Components in the remote don't inherit the host's theme. The UI looks broken — mismatched colors, fonts, and spacing.

**Key lesson**: Anything with internal state (React, state management libraries, design system providers) MUST be `singleton: true`. Stateless utilities can safely be duplicated.

</details>

## Practical — CI & Production Workflows

<details>
<summary>16. Design a CI pipeline for a frontend monorepo that runs affected-only testing, uses remote caching, and parallelizes E2E tests across multiple workers — show the pipeline configuration (CircleCI or GitHub Actions), explain how the "affected" calculation works when a shared library changes, and what safeguards prevent a cache corruption from silently passing broken builds.</summary>

**GitHub Actions pipeline:**

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  TURBO_TOKEN: ${{ secrets.TURBO_TOKEN }}
  TURBO_TEAM: ${{ secrets.TURBO_TEAM }}

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0  # Full history for affected calculation

      - uses: pnpm/action-setup@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'pnpm'

      - run: pnpm install --frozen-lockfile

      # Type check, lint, unit/integration tests — affected only
      - name: Build affected
        run: pnpm turbo build --filter="...[origin/main]"

      - name: Lint affected
        run: pnpm turbo lint --filter="...[origin/main]"

      - name: Unit & integration tests
        run: pnpm turbo test --filter="...[origin/main]"

  e2e:
    needs: build-and-test
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        shard: [1, 2, 3, 4]  # 4 parallel workers
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - uses: pnpm/action-setup@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'pnpm'

      - run: pnpm install --frozen-lockfile

      # Install Playwright browsers (cached)
      - name: Cache Playwright browsers
        uses: actions/cache@v4
        with:
          path: ~/.cache/ms-playwright
          key: playwright-${{ hashFiles('pnpm-lock.yaml') }}

      - run: pnpm exec playwright install --with-deps chromium

      # Build only what's needed for E2E
      - name: Build
        run: pnpm turbo build --filter="@myorg/web"
        env:
          TURBO_TOKEN: ${{ secrets.TURBO_TOKEN }}
          TURBO_TEAM: ${{ secrets.TURBO_TEAM }}

      # Run sharded E2E tests
      - name: E2E tests (shard ${{ matrix.shard }}/4)
        run: |
          pnpm exec playwright test \
            --shard=${{ matrix.shard }}/4 \
            --reporter=blob
        working-directory: apps/web

      # Upload shard results for merge
      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: e2e-report-${{ matrix.shard }}
          path: apps/web/blob-report/

  e2e-report:
    if: always()
    needs: e2e
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/download-artifact@v4
        with:
          path: all-blob-reports
          pattern: e2e-report-*
          merge-multiple: true

      - run: pnpm exec playwright merge-reports --reporter=html all-blob-reports

      - uses: actions/upload-artifact@v4
        with:
          name: e2e-html-report
          path: playwright-report/
```

**How affected calculation works when a shared library changes:**

Turborepo's `--filter="...[origin/main]"` syntax means: "find all packages that changed since `origin/main`, AND all packages that depend on them (transitively)."

When `packages/ui` (the shared design system) changes:
1. Turborepo detects file changes in `packages/ui` by comparing against `origin/main`.
2. It walks the dependency graph: `apps/web` depends on `@myorg/ui`, `apps/admin` depends on `@myorg/ui`.
3. All three packages (`ui`, `web`, `admin`) are included in the affected set.
4. Tasks run in dependency order: `ui` builds first, then `web` and `admin` build in parallel.

If only `apps/web` changes, `apps/admin` and `packages/ui` are not affected — their tasks are skipped (or served from cache).

**Safeguards against cache corruption:**

1. **Include all inputs in cache keys**: Every environment variable, config file, and source file that affects output must be listed in `turbo.json` `inputs` and `env`. If you miss one, the cache serves stale output.

```json
{
  "tasks": {
    "build": {
      "inputs": ["src/**", "tsconfig.json", "vite.config.ts", "index.html"],
      "env": ["VITE_API_URL", "NODE_ENV"]
    }
  }
}
```

2. **Run full builds on `main`** (no `--filter`): After a PR merges, run the full pipeline without affected filtering. This catches cases where affected detection missed a dependency. Populate the cache for all packages so subsequent PRs benefit from cache hits.

3. **Lockfile in global dependencies**: Include `pnpm-lock.yaml` in `globalDependencies` so dependency version changes invalidate all caches:

```json
{
  "globalDependencies": ["pnpm-lock.yaml", "**/.env.*local"]
}
```

4. **Periodic cache purge**: Remote cache TTL should be set (e.g., 7 days). Old cache entries can mask issues when the build toolchain changes in ways not captured by input hashing.

5. **CI verification step**: On a schedule (nightly), run a full build with `--force` (no cache) and compare outputs. If a cached build and a fresh build produce different test results, the cache is corrupted.

</details>

<details>
<summary>17. Configure Vite for a production React application — show the vite.config.ts with chunking strategy, dependency pre-bundling, environment variables, and proxy setup, then show the equivalent webpack configuration for the same setup and explain which parts of the migration are straightforward vs which require workarounds.</summary>

**Vite configuration (`vite.config.ts`):**

```typescript
import { defineConfig, loadEnv } from 'vite';
import react from '@vitejs/plugin-react';
import { resolve } from 'path';

export default defineConfig(({ mode }) => {
  const env = loadEnv(mode, process.cwd(), '');

  return {
    plugins: [react()],

    // Path aliases
    resolve: {
      alias: {
        '@': resolve(__dirname, 'src'),
        '@components': resolve(__dirname, 'src/components'),
      },
    },

    // Dev server proxy
    server: {
      port: 3000,
      proxy: {
        '/api': {
          target: env.API_URL || 'http://localhost:8080',
          changeOrigin: true,
          rewrite: (path) => path.replace(/^\/api/, ''),
        },
      },
    },

    // Dependency pre-bundling
    optimizeDeps: {
      include: [
        'react',
        'react-dom',
        'react-router-dom',
        '@tanstack/react-query',
      ],
      // Force re-optimization when these change
      exclude: ['@myorg/ui'], // Linked workspace packages — don't pre-bundle
    },

    // Production build
    build: {
      target: 'es2022',
      sourcemap: true,
      rollupOptions: {
        output: {
          manualChunks: {
            // Vendor chunk — changes rarely, cached long
            vendor: ['react', 'react-dom', 'react-router-dom'],
            // Query layer — changes occasionally
            query: ['@tanstack/react-query', 'axios'],
          },
        },
      },
      // Chunk size warning threshold
      chunkSizeWarningLimit: 500,
    },

    // Environment variables — only VITE_ prefixed are exposed to client
    envPrefix: 'VITE_',
    define: {
      // Expose a non-prefixed variable explicitly
      '__APP_VERSION__': JSON.stringify(process.env.npm_package_version),
    },
  };
});
```

**Equivalent webpack configuration (`webpack.config.js`):**

```javascript
const path = require('path');
const HtmlWebpackPlugin = require('html-webpack-plugin');
const webpack = require('webpack');
const MiniCssExtractPlugin = require('mini-css-extract-plugin');

module.exports = (env, argv) => {
  const isProd = argv.mode === 'production';

  return {
    entry: './src/index.tsx',
    output: {
      path: path.resolve(__dirname, 'dist'),
      filename: isProd ? '[name].[contenthash].js' : '[name].js',
      publicPath: '/',
      clean: true,
    },

    resolve: {
      extensions: ['.ts', '.tsx', '.js', '.jsx'],
      alias: {
        '@': path.resolve(__dirname, 'src'),
        '@components': path.resolve(__dirname, 'src/components'),
      },
    },

    module: {
      rules: [
        {
          test: /\.tsx?$/,
          use: 'ts-loader', // or babel-loader with presets
          exclude: /node_modules/,
        },
        {
          test: /\.css$/,
          use: [
            isProd ? MiniCssExtractPlugin.loader : 'style-loader',
            'css-loader',
          ],
        },
      ],
    },

    plugins: [
      new HtmlWebpackPlugin({ template: './index.html' }),
      new webpack.DefinePlugin({
        'process.env.VITE_API_URL': JSON.stringify(process.env.VITE_API_URL),
        '__APP_VERSION__': JSON.stringify(process.env.npm_package_version),
      }),
      ...(isProd ? [new MiniCssExtractPlugin()] : []),
    ],

    optimization: {
      splitChunks: {
        cacheGroups: {
          vendor: {
            test: /[\\/]node_modules[\\/](react|react-dom|react-router-dom)[\\/]/,
            name: 'vendor',
            chunks: 'all',
          },
          query: {
            test: /[\\/]node_modules[\\/](@tanstack|axios)[\\/]/,
            name: 'query',
            chunks: 'all',
          },
        },
      },
    },

    devtool: isProd ? 'source-map' : 'eval-source-map',

    devServer: {
      port: 3000,
      historyApiFallback: true,
      proxy: [{
        context: ['/api'],
        target: 'http://localhost:8080',
        pathRewrite: { '^/api': '' },
      }],
    },
  };
};
```

**Migration comparison:**

| Aspect | Migration difficulty | Notes |
|--------|---------------------|-------|
| Path aliases | Straightforward | Same concept, slightly different syntax |
| Dev server proxy | Straightforward | Nearly identical API |
| Environment variables | Straightforward but tedious | Rename all `process.env.REACT_APP_*` to `import.meta.env.VITE_*` across entire codebase |
| CSS/CSS Modules | Straightforward | Vite handles both natively — remove css-loader, style-loader |
| TypeScript | Straightforward | Vite uses esbuild for transforms — remove ts-loader entirely |
| Chunking strategy | Straightforward | `manualChunks` vs `splitChunks.cacheGroups` — different API, same concept |
| Custom loaders | Requires workarounds | Must find Rollup/Vite plugin equivalents. SVG-as-component (SVGR), raw file imports, and GraphQL imports may need specific plugins |
| `require()` / CommonJS | Requires workarounds | Vite is ESM-first. `require()` calls need refactoring to `import`. `@vitejs/plugin-commonjs` helps for dependencies but not source code |
| `module.rules` patterns | Requires workarounds | Vite doesn't have a loader chain. Complex loader pipelines (markdown → HTML → React component) need custom Vite plugins |
| Jest test config | Separate migration | Tests using webpack transforms (module name mapping, file mocks) need reconfiguration for Vitest |
| HTML template features | May need plugin | webpack's `HtmlWebpackPlugin` with EJS templates needs equivalent in Vite (`vite-plugin-html`) |

The biggest hidden cost is the **test migration** — if your Jest config depends on webpack module resolution and transforms, switching to Vitest is a significant parallel effort.

</details>

<details>
<summary>18. Enforce module boundaries and dependency rules in a monorepo — show how to configure Nx module boundary rules (or ESLint import restrictions for Turborepo), set up workspace-level constraints that prevent apps from importing from other apps, and demonstrate what happens when a developer violates these rules in their IDE and in CI.</summary>

**Nx module boundary rules:**

First, tag projects in their `project.json`:

```json
// apps/web/project.json
{
  "name": "web",
  "tags": ["type:app", "scope:customer"]
}

// apps/admin/project.json
{
  "name": "admin",
  "tags": ["type:app", "scope:admin"]
}

// packages/ui/project.json
{
  "name": "ui",
  "tags": ["type:ui", "scope:shared"]
}

// packages/utils/project.json
{
  "name": "utils",
  "tags": ["type:util", "scope:shared"]
}

// packages/feature-checkout/project.json
{
  "name": "feature-checkout",
  "tags": ["type:feature", "scope:customer"]
}
```

Then configure the enforcement rules:

```javascript
// eslint.config.mjs
import nx from '@nx/eslint-plugin';

export default [
  ...nx.configs['flat/base'],
  ...nx.configs['flat/typescript'],
  {
    files: ['**/*.ts', '**/*.tsx'],
    rules: {
      '@nx/enforce-module-boundaries': [
        'error',
        {
          allow: [],
          enforceBuildableLibDependency: true,
          depConstraints: [
            // Apps can depend on features, UI, and utils — NOT other apps
            {
              sourceTag: 'type:app',
              onlyDependOnLibsWithTags: ['type:feature', 'type:ui', 'type:util'],
            },
            // Features can depend on UI and utils — NOT apps or other features
            {
              sourceTag: 'type:feature',
              onlyDependOnLibsWithTags: ['type:ui', 'type:util'],
            },
            // UI can depend on other UI and utils only
            {
              sourceTag: 'type:ui',
              onlyDependOnLibsWithTags: ['type:ui', 'type:util'],
            },
            // Utils can only depend on other utils
            {
              sourceTag: 'type:util',
              onlyDependOnLibsWithTags: ['type:util'],
            },
            // Scope constraints: customer scope can't import admin scope
            {
              sourceTag: 'scope:customer',
              onlyDependOnLibsWithTags: ['scope:customer', 'scope:shared'],
            },
            {
              sourceTag: 'scope:admin',
              onlyDependOnLibsWithTags: ['scope:admin', 'scope:shared'],
            },
          ],
        },
      ],
    },
  },
];
```

**For Turborepo (ESLint-only approach):**

Without Nx's tag system, use ESLint's `no-restricted-imports` and `import/no-restricted-paths`:

```javascript
// eslint.config.mjs (for apps/web)
export default [
  {
    files: ['**/*.ts', '**/*.tsx'],
    rules: {
      'no-restricted-imports': ['error', {
        patterns: [
          {
            group: ['@myorg/admin', '@myorg/admin/*'],
            message: 'Apps cannot import from other apps. Extract shared code to a package.',
          },
        ],
      }],
      // Prevent importing internal files from other packages
      'import/no-restricted-paths': ['error', {
        zones: [
          {
            target: './src',
            from: '../admin/src',
            message: 'Cannot import from admin app internals.',
          },
          {
            // Only allow importing from package entry points
            target: './src',
            from: '../../packages/ui/src',
            except: ['../../packages/ui/src/index.ts'],
            message: 'Import from @myorg/ui entry point, not internal files.',
          },
        ],
      }],
    },
  },
];
```

**TypeScript path constraints** (additional layer):

```json
// tsconfig.base.json — only expose public APIs
{
  "compilerOptions": {
    "paths": {
      "@myorg/ui": ["packages/ui/src/index.ts"],
      "@myorg/utils": ["packages/utils/src/index.ts"]
      // NO path mapping for @myorg/admin or @myorg/web
      // Attempting to import them gives a TypeScript resolution error
    }
  }
}
```

**What happens when a developer violates the rules:**

**In the IDE (with ESLint extension):**
The import line gets a red squiggly underline immediately. The error message appears on hover:

```
ESLint: A project tagged with "type:app" can only depend on libs tagged with
"type:feature", "type:ui", "type:util". (@nx/enforce-module-boundaries)
```

The developer sees the violation before they even save the file. This is the most effective enforcement — it's immediate feedback in the developer's workflow.

**In CI:**
```bash
$ pnpm turbo lint

apps/web:lint: error  A project tagged with "type:app" can only depend on libs
tagged with "type:feature", "type:ui", "type:util"
apps/web:lint:   at apps/web/src/pages/Dashboard.tsx:3:1
apps/web:lint:     import { AdminPanel } from '@myorg/admin';
apps/web:lint:                                  ~~~~~~~~~~~~

apps/web:lint: ✖ 1 problem (1 error, 0 warnings)
apps/web:lint: ERROR: command finished with error: process exited with code 1
```

The CI pipeline fails, and the PR cannot be merged. The error message tells the developer exactly what's wrong and which constraint they violated.

**Key point**: IDE enforcement catches 95% of violations before they're even committed. CI enforcement is the safety net for developers who haven't configured their editor correctly or bypass it. Both layers are necessary.

</details>

<details>
<summary>19. Set up a frontend deployment pipeline that deploys to a CDN with proper cache invalidation, creates preview environments for each PR, and handles environment variable injection at runtime instead of build time — show the configuration and explain what breaks when teams hardcode environment variables at build time and need to promote the same artifact across environments.</summary>

**Runtime environment variable injection:**

Instead of `import.meta.env.VITE_*` (build-time), use a config endpoint or HTML template replacement:

```typescript
// src/config.ts — single source of truth for all config
interface AppConfig {
  apiUrl: string;
  authDomain: string;
  analyticsId: string;
  environment: string;
}

// In production, config is injected into window by the serving layer
// In development, fall back to Vite env vars
export const config: AppConfig = {
  apiUrl: window.__APP_CONFIG__?.apiUrl ?? import.meta.env.VITE_API_URL ?? 'http://localhost:8080',
  authDomain: window.__APP_CONFIG__?.authDomain ?? import.meta.env.VITE_AUTH_DOMAIN ?? 'localhost',
  analyticsId: window.__APP_CONFIG__?.analyticsId ?? '',
  environment: window.__APP_CONFIG__?.environment ?? 'development',
};

// Type declaration for the global config
declare global {
  interface Window {
    __APP_CONFIG__?: Partial<AppConfig>;
  }
}
```

```html
<!-- index.html — placeholders replaced at serve time -->
<!DOCTYPE html>
<html>
  <head>
    <script>
      window.__APP_CONFIG__ = {
        apiUrl: "{{API_URL}}",
        authDomain: "{{AUTH_DOMAIN}}",
        analyticsId: "{{ANALYTICS_ID}}",
        environment: "{{ENVIRONMENT}}"
      };
    </script>
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.tsx"></script>
  </body>
</html>
```

**Edge function that replaces placeholders (e.g., CloudFront Function):**

```javascript
// cloudfront-function.js — runs on every request for index.html
function handler(event) {
  var request = event.request;
  var uri = request.uri;

  // Serve index.html for all non-asset routes (SPA routing)
  if (!uri.includes('.')) {
    request.uri = '/index.html';
  }

  return request;
}

// Lambda@Edge for config injection (origin response)
exports.handler = async (event) => {
  const response = event.Records[0].cf.response;

  if (response.headers['content-type']?.[0]?.value?.includes('text/html')) {
    let body = Buffer.from(response.body, 'base64').toString('utf-8');

    // Replace placeholders with environment-specific values
    const envMap = {
      '{{API_URL}}': process.env.API_URL,
      '{{AUTH_DOMAIN}}': process.env.AUTH_DOMAIN,
      '{{ANALYTICS_ID}}': process.env.ANALYTICS_ID,
      '{{ENVIRONMENT}}': process.env.ENVIRONMENT,
    };

    for (const [placeholder, value] of Object.entries(envMap)) {
      body = body.replace(placeholder, value || '');
    }

    response.body = body;
  }

  return response;
};
```

**GitHub Actions deployment with preview environments:**

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main]
  pull_request:

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'pnpm'

      - run: pnpm install --frozen-lockfile

      # Build ONCE — no environment variables baked in
      - run: pnpm build
        env:
          NODE_ENV: production
          # NO VITE_API_URL here — it's injected at runtime

      - uses: actions/upload-artifact@v4
        with:
          name: build-output
          path: dist/

  deploy-preview:
    if: github.event_name == 'pull_request'
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: build-output
          path: dist/

      # Deploy to a unique preview URL per PR
      - name: Deploy to preview
        id: deploy
        run: |
          PREVIEW_URL="preview-pr-${{ github.event.number }}"
          aws s3 sync dist/ "s3://my-app-previews/${PREVIEW_URL}/" \
            --delete \
            --cache-control "max-age=0, no-cache"
          echo "url=https://${PREVIEW_URL}.previews.myapp.com" >> "$GITHUB_OUTPUT"

      - name: Comment PR with preview URL
        uses: actions/github-script@v7
        with:
          script: |
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `Preview deployed: ${{ steps.deploy.outputs.url }}`
            });

  deploy-production:
    if: github.ref == 'refs/heads/main'
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: build-output
          path: dist/

      # Same artifact, different environment — only runtime config changes
      - name: Deploy to S3
        run: |
          # Hashed assets — cache forever
          aws s3 sync dist/assets/ s3://my-app-prod/assets/ \
            --cache-control "max-age=31536000, immutable"

          # index.html — never cache (config placeholders are replaced at edge)
          aws s3 cp dist/index.html s3://my-app-prod/index.html \
            --cache-control "max-age=0, no-cache, no-store"

      - name: Invalidate CloudFront
        run: |
          aws cloudfront create-invalidation \
            --distribution-id ${{ secrets.CF_DISTRIBUTION_ID }} \
            --paths "/index.html"
          # No need to invalidate /assets/* — new hashes = new URLs
```

**What breaks with build-time environment variables:**

1. **No artifact promotion**: The staging build has `VITE_API_URL=https://api.staging.myapp.com` baked into the JS bundle. The production build has `VITE_API_URL=https://api.myapp.com`. These are different binaries. You cannot promote the staging artifact to production — you must rebuild. The code you tested in staging is not the code running in production.

2. **Preview environments require per-PR builds**: Each preview environment needs a different API URL. With build-time injection, you rebuild for every PR with different env vars. With runtime injection, one build artifact works for all previews.

3. **Rollback complexity**: Rolling back means redeploying a previous build artifact. If env vars changed between then and now, the old artifact has the old env vars baked in — potentially pointing at decommissioned services.

4. **Secret leakage risk**: Build-time env vars end up as string literals in the JS bundle. Even non-`VITE_` prefixed vars might leak through `define` or misconfigured plugins. Runtime injection keeps config in the HTML response, which can be served with different caching rules and isn't bundled into cacheable JS files.

</details>

## Practical — Debugging & Troubleshooting

<details>
<summary>20. An E2E test suite is failing intermittently — about 10% of runs fail with different tests each time. Walk through the systematic diagnosis: identifying timing-dependent assertions, checking for test isolation failures (shared state, database/API leakage between tests), handling network request flakiness, and deciding between retry strategies vs proper fixes. Show Playwright-specific debugging techniques.</summary>

**Step 1: Gather data before changing anything.**

Enable Playwright's trace recording on failure to capture what actually happened:

```typescript
// playwright.config.ts
import { defineConfig } from '@playwright/test';

export default defineConfig({
  retries: process.env.CI ? 2 : 0,
  use: {
    trace: 'on-first-retry', // Captures trace on first retry of a failed test
    screenshot: 'only-on-failure',
    video: 'retain-on-failure',
  },
  reporter: [
    ['html'],
    ['json', { outputFile: 'test-results.json' }],
  ],
});
```

Run the suite 10-20 times and collect failure data. Look for patterns: Are the same 5 tests failing in rotation? Are failures correlated (test A and test B always fail together)? Are failures clustered in specific spec files?

```bash
# Run the suite multiple times and collect results
for i in {1..20}; do
  npx playwright test --reporter=json 2>/dev/null | \
    jq '.suites[].suites[].specs[] | select(.ok == false) | .title' \
    >> failures.txt
done

# Count which tests fail most often
sort failures.txt | uniq -c | sort -rn
```

**Step 2: Diagnose timing-dependent assertions.**

Open the trace viewer for failed tests:

```bash
npx playwright show-trace test-results/my-test-retry1/trace.zip
```

The trace shows a timeline of actions and assertions. Look for:
- An assertion that fires before the UI update completes. The trace will show the assertion attempting to match text that hasn't appeared yet.
- A click on an element that's being animated or repositioned (the click lands on the wrong element).

**Common fixes:**

```typescript
// BAD: Race condition — text might not be rendered yet
const text = await page.textContent('.result');
expect(text).toBe('Success');

// GOOD: Auto-retrying assertion — Playwright retries until timeout
await expect(page.locator('.result')).toHaveText('Success');

// BAD: Element might be covered by a loading overlay
await page.click('.submit-button');

// GOOD: Wait for the overlay to disappear first
await expect(page.locator('.loading-overlay')).toBeHidden();
await page.locator('.submit-button').click();

// BAD: Checking immediately after navigation
await page.goto('/dashboard');
const count = await page.locator('.items').count();

// GOOD: Wait for the data to load
await page.goto('/dashboard');
await expect(page.locator('.items')).not.toHaveCount(0);
```

**Step 3: Check for test isolation failures.**

If test B fails only when test A runs before it, you have a state leakage problem. Test in two ways:

```bash
# Run only the failing test — does it pass in isolation?
npx playwright test tests/checkout.spec.ts --grep "should complete purchase"

# Run with --shard to randomize order
npx playwright test --shard=1/1 --workers=1
```

Common isolation failures and fixes:

```typescript
// BAD: Tests share browser state
test('test A creates a user', async ({ page }) => {
  // Creates user, leaves cookies/session
});
test('test B expects clean state', async ({ page }) => {
  // Fails because test A's session data persists
});

// GOOD: Each test gets a fresh context (Playwright default with { page } fixture)
// But if you're using a shared storageState, that's your leak:

// playwright.config.ts — isolate auth state
export default defineConfig({
  projects: [
    {
      name: 'setup',
      testMatch: /.*\.setup\.ts/,
    },
    {
      name: 'tests',
      dependencies: ['setup'],
      use: {
        // Each worker gets its own storage state
        storageState: 'playwright/.auth/user.json',
      },
    },
  ],
});
```

**Step 4: Handle network request flakiness.**

If tests hit real APIs, network latency and server errors cause intermittent failures:

```typescript
// Mock external APIs in E2E tests for determinism
test('shows product list', async ({ page }) => {
  // Intercept API calls and return deterministic data
  await page.route('**/api/products', (route) => {
    route.fulfill({
      status: 200,
      contentType: 'application/json',
      body: JSON.stringify([
        { id: 1, name: 'Widget', price: 9.99 },
        { id: 2, name: 'Gadget', price: 19.99 },
      ]),
    });
  });

  await page.goto('/products');
  await expect(page.locator('.product-card')).toHaveCount(2);
});

// For tests that MUST hit real APIs, add explicit wait conditions
test('loads real data', async ({ page }) => {
  // Wait for the specific API response before asserting
  const responsePromise = page.waitForResponse('**/api/products');
  await page.goto('/products');
  await responsePromise;
  await expect(page.locator('.product-card')).not.toHaveCount(0);
});
```

**Step 5: Retry strategy vs proper fix decision.**

| Situation | Approach |
|-----------|----------|
| Root cause identified and fixable | Fix it. No retries. |
| Timing issue in third-party service you don't control | Retry (2 retries max) + mock the service for most tests |
| Flakiness from browser rendering timing | Use Playwright's auto-waiting assertions — this IS the proper fix |
| Test data shared across parallel workers | Fix isolation — retries just make a bad design pass sometimes |
| Flaky for 2+ weeks, nobody can diagnose | Delete it. Write a simpler test that covers the critical path |

**Retries are a diagnostic tool, not a fix.** Configure `retries: 2` in CI to keep the pipeline green while you investigate, but track retry rate. If retry rate exceeds 5%, you have systemic issues that retries are masking.

```typescript
// playwright.config.ts — retries with tracking
export default defineConfig({
  retries: process.env.CI ? 2 : 0,
  reporter: [
    ['html'],
    // Track flaky tests over time
    ['json', { outputFile: 'test-results.json' }],
  ],
});
```

</details>

<details>
<summary>21. A design system component update (new major version) is breaking multiple consuming applications with different errors — walk through how to diagnose whether the breaks are from changed props, removed tokens, CSS specificity changes, or peer dependency mismatches, and explain the release process that should have prevented this (changesets, visual regression testing, migration codemods).</summary>

**Step 1: Triage the errors by category.**

Collect all the build and runtime errors across consuming apps. Group them:

- **TypeScript compilation errors** (changed/removed props, renamed exports) — these are the easiest to diagnose because the compiler tells you exactly what changed.
- **Runtime errors** (component crashes, hooks failures) — usually peer dependency mismatches or internal behavioral changes.
- **Visual regressions** (layout shifts, wrong colors, spacing changes) — token removals, CSS specificity changes, or default value changes.

```bash
# In each consuming app, try building against the new version
pnpm install @myorg/design-system@3.0.0
pnpm build 2>&1 | tee build-errors.log

# Categorize errors
grep "Property .* does not exist" build-errors.log    # Changed props
grep "Module .* has no exported member" build-errors.log  # Removed exports
grep "has no properties in common" build-errors.log    # Renamed/restructured types
```

**Step 2: Diagnose changed props.**

If the design system renamed `variant="primary"` to `variant="brand"`, or changed `size` from `"sm" | "md" | "lg"` to `"small" | "medium" | "large"`, every consumer using the old values gets a TypeScript error. Check the design system's changelog or diff:

```bash
# Compare the public API between versions
diff <(npm pack @myorg/design-system@2.0.0 --dry-run 2>/dev/null) \
     <(npm pack @myorg/design-system@3.0.0 --dry-run 2>/dev/null)

# Or compare type definitions directly
diff node_modules/@myorg/design-system-v2/dist/index.d.ts \
     node_modules/@myorg/design-system/dist/index.d.ts
```

**Step 3: Diagnose removed tokens.**

If the design system removed or renamed CSS custom properties / design tokens, consuming apps that reference them directly get broken styles with no build error:

```bash
# Search for tokens that existed in v2 but not v3
diff <(grep -r "var(--ds-" node_modules/@myorg/design-system-v2/dist/) \
     <(grep -r "var(--ds-" node_modules/@myorg/design-system/dist/)

# Search consuming apps for references to removed tokens
grep -rn "var(--ds-color-primary)" apps/
```

Removed tokens produce silent failures — the CSS custom property resolves to nothing, and elements render with default/inherited values. This is the hardest category to catch without visual regression testing.

**Step 4: Diagnose CSS specificity changes.**

If the design system changed from CSS Modules to CSS-in-JS (or vice versa), or restructured component internals, specificity changes can break consumer overrides:

```css
/* Consumer's override — worked in v2 because the DS used .button class */
.my-form .button { padding: 12px; }

/* In v3, DS changed to .ds-button-root — consumer's override no longer matches */
```

Check by inspecting the rendered DOM in the browser — if class names changed, consumer CSS selectors targeting internal classes break. This is why design systems should document which class names are part of the public API and which are internal.

**Step 5: Diagnose peer dependency mismatches.**

If the design system upgraded its `react` peer dependency from `^17` to `^18`, or changed a shared dependency version, consumers on the old version get runtime errors:

```bash
# Check peer dependency changes
diff <(npm info @myorg/design-system@2.0.0 peerDependencies --json) \
     <(npm info @myorg/design-system@3.0.0 peerDependencies --json)

# Check for duplicate React instances
npm ls react  # Should show one version, not two
```

Symptoms: "Invalid hook call" errors, context providers not connecting to consumers, `instanceof` checks failing across package boundaries.

**The release process that should have prevented this:**

1. **Changesets for intentional versioning**: Every PR to the design system includes a changeset (`npx changeset`) describing what changed and whether it's a patch, minor, or major bump. The release pipeline aggregates changesets into a changelog and publishes the correct version.

2. **Visual regression testing (Chromatic/Percy)**: On every PR, a tool renders every component variant and diffs screenshots against the baseline. Token removals, specificity changes, and default value changes show up as visual diffs that must be explicitly approved. This catches the silent failures that TypeScript can't.

3. **Integration test suite against consuming apps**: Before publishing a major version, run the build and test suites of the top 3-5 consuming apps against the release candidate. If they break, the release is blocked until either the DS is fixed or codemods are ready.

4. **Migration codemods**: For every breaking change, ship an automated codemod:

```bash
# Consumer runs the codemod to auto-migrate
npx @myorg/design-system-codemods v2-to-v3 --path=src/

# The codemod handles:
# - Renaming variant="primary" → variant="brand"
# - Updating removed token references
# - Replacing deprecated component imports
```

5. **Pre-release versions**: Publish `3.0.0-rc.1` and let consuming teams test before the final release. Use npm dist-tags so `npm install @myorg/design-system` still gets v2 until v3 is stable.

6. **Deprecation warnings in v2.x**: Before removing anything in v3, add runtime deprecation warnings in a v2.x minor release. Consumers see console warnings for 2-4 weeks before the breaking change lands.

</details>

---

## Experience-Based Questions

These questions test real-world experience. Prepare by mapping them to your own projects and situations.

<details>
<summary>22. Tell me about a time you designed or significantly refactored a frontend monorepo structure — what drove the decision, how did you handle the migration, and what would you do differently knowing what you know now?</summary>

**What the interviewer is looking for:**

- Ability to identify structural problems in a codebase and make the case for change
- Understanding of monorepo tooling tradeoffs (not just "we picked Nx")
- Migration strategy — how you moved incrementally without blocking the team
- Honest reflection on what didn't go well

**Suggested structure (STAR+):**

1. **Situation**: Describe the codebase state — how many apps, what was the pain (slow builds, code duplication, deployment coupling, unclear ownership)?
2. **Task**: What was the goal? Faster CI? Better code sharing? Team autonomy?
3. **Action**: Walk through the key decisions:
   - Why monorepo (or why restructure within existing monorepo)?
   - Which tool (Nx, Turborepo, custom) and why?
   - How did you handle the migration? Big bang or incremental? How did you keep the team productive during the transition?
   - What module boundary rules did you set up?
4. **Result**: Concrete improvements — CI time reduction, fewer cross-team merge conflicts, measurable code sharing improvement.
5. **Reflection**: What would you change? Common answers: "Should have set up module boundaries from day one," "Underestimated the Jest-to-Vitest migration cost," "Should have migrated one app first as a proof of concept."

**Example outline to personalize:**

- "We had 4 React apps in separate repos sharing a component library published to npm. Every shared component change required publish → update → test across 4 repos — a change that should take an hour took a day."
- "I proposed consolidating into a Turborepo monorepo. Key decisions: Turborepo over Nx (simpler, team already had custom tooling), pnpm workspaces, shared tsconfig and ESLint."
- "Migration was incremental — moved one app per sprint, kept the old npm package as a compatibility layer during transition."
- "Result: shared component changes went from 1 day to 15 minutes. CI with remote caching dropped from 20 minutes to 6 minutes for unaffected apps."
- "What I'd change: I'd set up `no-restricted-imports` rules immediately. We spent 3 weeks untangling cross-app imports that crept in during the first month."

</details>

<details>
<summary>23. Describe a time you built or evolved a design system that multiple teams consumed — how did you handle governance, versioning, and adoption, and what was the hardest part about scaling it across teams?</summary>

**What the interviewer is looking for:**

- Understanding of design systems as organizational infrastructure, not just a component library
- Experience with the human side — governance, adoption resistance, balancing team needs
- Versioning and release strategy that works at scale
- Honest assessment of what's hard about shared UI

**Suggested structure:**

1. **Context**: How many teams? How many apps? What existed before (nothing, a loose collection of shared components, a full design system that wasn't being adopted)?
2. **Architecture decisions**: Token structure, component layering (primitives vs composed), packaging strategy (one package vs scoped packages), framework coupling.
3. **Governance model**: Who owns it? How do teams contribute? How are breaking changes managed? Was there an RFC process?
4. **Adoption strategy**: How did you get teams to actually use it? Mandated vs voluntary? Migration tooling?
5. **The hard part**: Be specific. Common honest answers:
   - "Teams kept wrapping our components with overrides instead of contributing back"
   - "Versioning was a constant source of friction — we couldn't release fast enough for some teams and too fast for others"
   - "Getting design and engineering aligned on what's 'standard' vs what's 'app-specific'"
6. **What you learned**: What would you do differently?

**Example outline to personalize:**

- "We had 3 frontend teams building similar-looking admin dashboards with completely different button, modal, and form implementations. Users noticed the inconsistency."
- "I led the effort to extract a shared design system. Key decisions: token-first approach (colors, spacing, typography as CSS custom properties), primitives only (no composed components initially), published as a single npm package with semver."
- "Governance: inner-source model — my team owned the core, other teams submitted PRs. We reviewed for API consistency and accessibility. Monthly design system council to prioritize requests."
- "Hardest part: one team had deeply customized their forms. Migrating them to standard primitives required rethinking their entire form architecture. We ended up supporting their custom form components as a 'community' tier — not officially supported but in the monorepo."
- "What I learned: start with tokens, not components. If every team uses the same tokens, the visual consistency problem is 80% solved even without shared components."

</details>

<details>
<summary>24. Tell me about a time you diagnosed and fixed a persistent CI or build performance problem in a frontend codebase — what were the symptoms, what was the root cause, and what measurable improvement did you achieve?</summary>

**What the interviewer is looking for:**

- Systematic debugging approach — not "I tried random things until it worked"
- Understanding of build tooling internals (bundler behavior, CI caching, dependency resolution)
- Ability to measure before and after — not just "it felt faster"
- Prioritization — fixing the highest-impact bottleneck first

**Suggested structure:**

1. **Symptoms**: What was the observable problem? "CI takes 25 minutes." "Local dev server takes 45 seconds to start." "E2E tests time out intermittently." Be specific with numbers.
2. **Investigation**: How did you identify the root cause?
   - CI: Which step was slow? Did you analyze pipeline timing? Cache hit rates?
   - Build: Did you use `--profile` flags, bundle analyzers, or timing plugins?
   - Tests: Did you identify which tests were slow vs the overhead was in setup?
3. **Root cause**: Be specific. Examples:
   - "No remote caching — every PR rebuilt everything from scratch"
   - "A single test file was importing the entire app bundle instead of the component under test"
   - "TypeScript type checking ran on the full monorepo even for single-app changes"
   - "Playwright browsers were being downloaded fresh on every CI run instead of cached"
4. **Fix**: What did you change? Show the before/after configuration if possible.
5. **Result**: Concrete numbers. "CI dropped from 25 minutes to 8 minutes." "Dev server startup went from 45 seconds to 2 seconds." "E2E suite went from 12 minutes to 4 minutes with sharding."

**Example outline to personalize:**

- "Our monorepo CI was taking 22 minutes on every PR. The team was context-switching while waiting, and it was killing review velocity."
- "I broke down the pipeline timing: 3 min install, 8 min build, 6 min test, 5 min E2E. Install was fine. Build was the biggest single step."
- "Root cause: Turborepo was rebuilding all 6 packages on every PR because `turbo.json` didn't have `inputs` configured — any file change (including README edits) invalidated the cache for every package."
- "Fix: Added explicit `inputs` to `turbo.json`, set up Vercel Remote Cache, and added `--filter='...[origin/main]'` to only run affected tasks."
- "Result: Average CI time dropped to 6 minutes. PRs that only touched one app ran in under 4 minutes. Cache hit rate went from 0% to 73%."

</details>

<details>
<summary>25. Describe a time you had to make a build-vs-buy decision for frontend infrastructure tooling (monorepo tool, design system, testing framework, CI setup) — what factors drove your decision, and how did it play out?</summary>

**What the interviewer is looking for:**

- Decision-making framework — not just "we picked X because it's popular"
- Understanding of total cost of ownership (not just initial setup cost)
- Ability to evaluate options against team-specific constraints (team size, expertise, existing tooling)
- Honest reflection on whether the decision was right in hindsight

**Suggested structure:**

1. **Context**: What was the decision? What were the options? What constraints existed (team size, timeline, existing expertise, budget)?
2. **Evaluation criteria**: What factors mattered most? Common ones:
   - **Team expertise** — Can the team maintain it? Is there existing knowledge?
   - **Migration cost** — What's the switching cost from current state?
   - **Long-term maintenance** — Who maintains the "build" option? Does the "buy" option have a sustainable business model?
   - **Flexibility vs convention** — Does the team need customization or would guardrails help?
   - **Community and ecosystem** — Can you hire people who know this tool? Are there plugins for your use cases?
3. **Decision and rationale**: What did you choose and why? Be explicit about what you traded off.
4. **Outcome**: Did the decision hold up? What surprised you? Would you make the same choice today?

**Example outline to personalize:**

- "We needed a monorepo tool for 3 React apps and 8 shared packages. Options: Nx, Turborepo, or a custom solution with npm workspaces + shell scripts."
- "Custom scripts were the status quo — `build.sh` that ran `tsc` and `vite build` in dependency order. It worked but had no caching, no affected detection, and broke whenever someone added a new package."
- "Evaluation: Nx had the best features (graph visualization, generators, DTE) but the team found it opinionated and complex — nobody wanted to learn Nx-specific configuration on top of Vite + TypeScript. Turborepo was simpler but less powerful. Custom scripts were zero overhead but scaling poorly."
- "We chose Turborepo. The deciding factor was that it augmented our existing `package.json` scripts instead of replacing them. Migration took 2 days instead of 2 weeks."
- "6 months later: right decision for our team size (8 devs). We hit one limitation — no source-level dependency detection — which caused a cache miss bug that took a day to diagnose. For a larger team, I'd reconsider Nx for its graph intelligence."

</details>
