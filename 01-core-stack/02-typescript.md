# TypeScript

> **42 questions** — 20 theory, 22 practical

- Structural typing vs nominal typing and branded types
- Strict mode: strictNullChecks, noImplicitAny, strictFunctionTypes — what each enables and migration strategy
- Type erasure, runtime limitations, and runtime validation (Zod, io-ts)
- Generics: constraints, defaults, generic functions/interfaces/classes
- Conditional types, `infer`, `never`, and distributive behavior
- Template literal types: string manipulation at the type level, real-world usage in library APIs and event systems
- Mapped types, key remapping, and utility types (Partial, Required, Pick, Omit, Record)
- Type operators: keyof, typeof at the type level, indexed access types (T[K]), and combining them for type-safe property access
- Discriminated unions and exhaustiveness checking
- Type narrowing: typeof, instanceof, `in`, custom type guards, assertion functions
- Type assertion patterns: satisfies, as const, explicit annotations — when to use each
- Enums vs union types: string enums, const enums, numeric pitfalls, and when unions are better
- Function overloads: when to use overloads vs union parameters vs generics
- Decorators (legacy and TC39 stage 3): reflect-metadata, DI patterns (NestJS), and the migration path between decorator versions
- Declaration files (.d.ts), DefinitelyTyped, module augmentation for extending third-party types
- Project references, composite mode, and incremental compilation for monorepos
- Module resolution strategies (node10, node16, bundler) and ESM/CJS interop
- Build tooling: tsc vs esbuild vs swc, ts-node vs tsx for development, type-checking vs transpile-only tradeoffs
- Type safety gaps: any vs unknown, type assertions, soundness holes, and any propagation in strict codebases
- Error handling typing: Result/Either patterns, discriminated union errors vs thrown exceptions, typing catch blocks, and why TS lacks checked exceptions
- Production tsconfig.json: target, module, moduleResolution, paths, skipLibCheck, outDir — Node.js-specific choices and common pitfalls (ESM target, path aliases in compiled output)
- JavaScript runtime quirks TypeScript doesn't catch: typeof null, NaN equality, floating-point precision, coercion traps, parseInt gotchas, var hoisting vs let/const TDZ

---

## Foundational

<details>
<summary>1. Why does TypeScript use structural typing instead of nominal typing like Java or C# — what are the practical consequences of "if the shape matches, it's compatible," when does this cause unexpected type compatibility, and how do branded types let you opt into nominal behavior when you need it?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>2. Why does TypeScript's type system disappear entirely at runtime (type erasure) — what are the practical implications of having no runtime type information, why does this create a dangerous gap at system boundaries (API responses, user input, database results), and how do libraries like Zod and io-ts bridge this gap?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>3. How do any and unknown differ in TypeScript's type system — why does any break type safety by propagating through expressions, what are the common soundness holes in TypeScript (type assertions, index signatures, function parameter bivariance), and how does any sneak into a strict codebase despite strict mode being enabled?</summary>

<!-- Answer will be added later -->

</details>

## Conceptual Depth

<details>
<summary>4. What does TypeScript's strict mode actually enable — explain what strictNullChecks, noImplicitAny, and strictFunctionTypes each catch that would otherwise be silent bugs, why would you enable them individually vs all at once, and what's the practical migration strategy for a large existing codebase that has none of these flags?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>5. How do generics work in TypeScript beyond basic syntax — when and why would you use constraints (extends), defaults, and how do generic functions, interfaces, and classes differ in how they bind type parameters? What are the common mistakes with generics (over-constraining, unnecessary type parameters)?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>6. How do conditional types work in TypeScript — what does the `Type extends Condition ? A : B` pattern enable, how does `infer` extract types from complex structures, why does `never` behave unexpectedly in conditional types, and what is distributive behavior over union types and when would you disable it?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>7. How do mapped types work in TypeScript — what is key remapping with `as`, how do you build custom mapped types, and how do the built-in utility types (Partial, Required, Pick, Omit, Record) use mapped types under the hood? When would you build a custom mapped type vs compose existing utility types?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>8. Why are discriminated unions one of TypeScript's most powerful patterns — how does a shared literal discriminant field enable exhaustiveness checking, how does the `never` trick in a switch default catch missing cases at compile time, and when are discriminated unions a better fit than class hierarchies or overloads?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>9. How does TypeScript narrow types within control flow — what are the mechanisms (typeof, instanceof, `in`, truthiness), when do you need a custom type guard function (paramName is Type) vs an assertion function (asserts paramName is Type), and what are the pitfalls where narrowing silently fails?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>10. Why do TypeScript developers often prefer union types over enums — what are the pitfalls of numeric enums (reverse mapping, widening), when do const enums help and what are their limitations with --isolatedModules, and when are string enums genuinely the better choice?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>11. When should you use function overloads in TypeScript vs union parameter types vs generics — what specific problem do overloads solve that the alternatives can't, what are the implementation signature rules, and what are the common mistakes (too many overloads, implementation signature mismatches)?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>12. How do TypeScript decorators work — what's the difference between the legacy (experimentalDecorators) and TC39 stage 3 decorators, how does reflect-metadata enable dependency injection frameworks like NestJS, and what are the limitations (decorator evaluation order, decorated type inference)?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>13. Why does TypeScript have multiple module resolution strategies (node10, node16, bundler) — what problem does each solve, how does node16 resolution differ for ESM vs CJS imports, and what are the common interop headaches when mixing ESM and CJS packages in a Node.js TypeScript project?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>14. Why does TypeScript need project references and composite mode for monorepos — what problem does incremental compilation solve when you have multiple packages, how do the build and references fields in tsconfig.json work together, and when are project references overkill?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>15. Why does TypeScript offer three assertion patterns — satisfies, as const, and explicit type annotations — what does each one do differently to the inferred type, when should you use each, and what goes wrong when you use `as` (type assertion) where you should have used `satisfies`?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>16. How do template literal types work in TypeScript — what string manipulation is possible at the type level (uppercase, lowercase, split, join patterns), how do libraries use them to create type-safe event emitters or route parsers, and when are template literal types genuinely useful vs an over-engineered party trick?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>17. How do TypeScript's type operators (keyof, typeof at the type level, and indexed access types T[K]) work together to enable type-safe property access — show how combining keyof with indexed access creates the pattern behind Pick and Record, how typeof extracts types from runtime values, and what are the common mistakes when using these operators (e.g., typeof on a type vs a value)?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>18. Why are there so many TypeScript build tools (tsc, esbuild, swc) and development runners (ts-node, tsx) — what does each one actually do differently, what is the "transpile-only" approach and what type safety do you lose with it, and how do you decide which combination to use for a production backend service vs a development workflow?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>19. How do you type error handling in TypeScript — why does TypeScript type catch blocks as unknown (and why did it used to be any), what are the tradeoffs between Result/Either return types vs thrown exceptions for error handling, how do discriminated union error types work as an alternative to exception hierarchies, and why doesn't TypeScript have checked exceptions like Java?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>20. What JavaScript runtime quirks does TypeScript NOT protect you from — why does `typeof null === "object"` break type guard assumptions, how does `NaN !== NaN` cause subtle bugs in comparisons, what goes wrong with floating-point arithmetic in financial calculations (`0.1 + 0.2 !== 0.3`), and what other JS runtime traps (equality coercion, parseInt edge cases, var hoisting vs let/const TDZ in closures) should a TypeScript developer still watch for?</summary>

<!-- Answer will be added later -->

</details>

## Practical — Type System Patterns

<details>
<summary>21. Build a generic repository interface in TypeScript that constrains the entity type to have an id field — show how the constraint prevents invalid usage, how you'd add generic methods (findById, create, update) that preserve type safety, and what happens if a consumer passes a type that doesn't satisfy the constraint</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>22. Write a generic pipe or compose function in TypeScript that correctly infers the return type through a chain of functions — show how TypeScript infers the intermediate and final types, what limitations you hit with variadic generics, and how this pattern appears in real libraries (e.g., middleware chains, validation pipelines)</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>23. Write `infer`-based conditional types that extract the return type of a Promise (unwrapping nested Promises) and extract function parameter types — show how `infer` works in each case and explain what happens when you pass a union type to each</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>24. Create a recursive conditional/mapped type DeepPartial that recursively makes all properties optional — explain the recursion mechanics, show edge cases (arrays, optional properties), and demonstrate how to test that recursive types don't infinite-loop</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>25. Build a mapped type that transforms an interface's method signatures — for example, take a service interface and produce an async client type where every method returns a Promise. Show the mapped type, demonstrate key remapping with 'as' to filter or rename keys, and explain when you'd use this pattern in production (e.g., generating client SDKs, mocking interfaces)</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>26. Show how to compose Pick, Omit, and Record to build a type-safe event handler map — given a set of event names and their payload types, construct a handler registry type where each event maps to a correctly-typed callback, and demonstrate what TypeScript catches at compile time if a handler has the wrong signature</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>27. Implement a discriminated union for a state machine (e.g., request states: idle, loading, success, error) with proper exhaustiveness checking — show the never trick in a switch default, demonstrate how this catches missing cases at compile time, and show the pattern for type-safe reducer functions</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>28. Write a custom type guard function that validates an unknown API response into a typed shape, then write an assertion function that throws if the input doesn't match — show how each works in control flow, when to use one over the other, and what happens if your type guard lies (returns true for invalid data)</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>29. Show real-world scenarios where you'd use satisfies vs as const vs an explicit type annotation — demonstrate how satisfies validates a value matches a type without widening it, how as const creates literal types and readonly tuples, and what goes wrong when you use `as` instead (silent type assertion bugs)</summary>

<!-- Answer will be added later -->

</details>

## Practical — Configuration & Tooling

<details>
<summary>30. Configure a production tsconfig.json for a Node.js backend service — explain your choices for target, module, moduleResolution, paths (alias mapping), strict flags, skipLibCheck, and declaration output. Show how the same codebase's tsconfig differs when targeting Node 18 vs Node 20, and what breaks if you get module/moduleResolution wrong</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>31. Set up TypeScript project references for a monorepo with 3 packages (shared types, API server, worker) — show the tsconfig.json for each package and the root, demonstrate how `tsc --build` handles incremental compilation, explain the composite and declarationMap settings, and explain what breaks if composite is missing or references are misconfigured (stale .d.ts files, broken cross-package imports)</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>32. Set up runtime validation at an API boundary using Zod — define a schema that matches a TypeScript type, use it to validate incoming request bodies, show how to infer the TypeScript type from the Zod schema (z.infer), and demonstrate what happens when validation fails vs when you skip validation and trust incoming data</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>33. Configure a TypeScript project to correctly resolve both ESM and CJS dependencies — show the tsconfig moduleResolution setting, package.json type field, and the specific errors you get when resolution is wrong (ERR_REQUIRE_ESM, ERR_MODULE_NOT_FOUND), and how to fix each</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>34. Write a declaration file (.d.ts) for an untyped JavaScript library, then use module augmentation to add a custom property to Express's Request interface — show both techniques, explain when you'd use DefinitelyTyped (@types/) vs writing your own, what the `declare module` syntax does, and what errors you get when types are missing or augmentation doesn't take effect (duplicate identifier, module not found)</summary>

<!-- Answer will be added later -->

</details>

## Practical — Migration & Refactoring

<details>
<summary>35. Walk through migrating a JavaScript codebase to TypeScript strict mode incrementally — show the tsconfig changes at each stage (allowJs → checkJs → strict flags one by one), how to use // @ts-expect-error for gradual migration, and what the most common strict mode violations are and how to fix each</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>36. Refactor a function that uses union parameter types into proper overloads, then show an alternative using generics — compare the three approaches (union params, overloads, generics) for the same function, demonstrate where each approach gives better type inference to callers, and when overloads cause more harm than good</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>37. Set up TypeScript decorators for dependency injection in a NestJS-style pattern — show a class decorator, method decorator, and parameter decorator, demonstrate how reflect-metadata stores type information at runtime, explain what changes with TC39 stage 3 decorators (no reflect-metadata, different API), and what breaks if you mix legacy and TC39 decorators in the same project or forget to configure reflect-metadata</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>38. Given a TypeScript codebase with several any types that have leaked through — walk through finding them (strict compiler flags, ESLint rules), show how to systematically replace any with unknown and add proper narrowing, and demonstrate how a single any in a utility function propagates to infect callers' types</summary>

<!-- Answer will be added later -->

</details>

---

## Experience-Based Questions

These questions test real-world experience. Prepare by mapping them to your own projects and situations.

<details>
<summary>39. Tell me about a time you migrated a codebase to TypeScript or significantly tightened its type safety — what was the scope, what challenges did you face, and what was the impact on developer productivity and bug rates?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>40. Describe a time you built a complex generic type or utility type to solve a real problem — what was the use case, how did you design it, and how did it improve the codebase?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>41. Tell me about a time a type safety gap caused a production bug — what was the gap (any leak, missing validation, incorrect assertion), how did you discover it, and what did you change to prevent it from recurring?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>42. Describe a time you configured TypeScript for a large project or monorepo — what decisions did you make about strict mode, project references, module resolution, and what tradeoffs did you encounter?</summary>

<!-- Answer framework will be added later -->

</details>
