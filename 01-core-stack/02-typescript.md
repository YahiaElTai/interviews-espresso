# TypeScript

> **31 questions** — 15 theory, 16 practical

- Structural typing vs nominal typing and branded types
- Strict mode: strictNullChecks, noImplicitAny, strictFunctionTypes — what each enables and migration strategy
- Type erasure, runtime limitations, and runtime validation (Zod, io-ts)
- Generics: constraints, defaults, generic functions/interfaces/classes
- Conditional types, `infer`, `never`, and distributive behavior
- Mapped types, key remapping, and utility types (Partial, Required, Pick, Omit, Record)
- Type operators: keyof, typeof at the type level, indexed access types (T[K]), and combining them for type-safe property access
- Discriminated unions and exhaustiveness checking
- Type narrowing: typeof, instanceof, `in`, custom type guards, assertion functions
- Type assertion patterns: satisfies, as const, explicit annotations — when to use each
- Enums vs union types: string enums, const enums, numeric pitfalls, and when unions are better
- Function overloads: when to use overloads vs union parameters vs generics
- Module resolution strategies (node10, node16, bundler) and ESM/CJS interop
- Type safety gaps: any vs unknown, type assertions, soundness holes, and any propagation in strict codebases
- Error handling typing: Result/Either patterns, discriminated union errors vs thrown exceptions, typing catch blocks, and why TS lacks checked exceptions
- Production tsconfig.json: target, module, moduleResolution, paths, skipLibCheck, outDir — Node.js-specific choices and common pitfalls (ESM target, path aliases in compiled output)

---

## Foundational

<details>
<summary>1. Why does TypeScript use structural typing instead of nominal typing like Java or C# — what are the practical consequences of "if the shape matches, it's compatible," when does this cause unexpected type compatibility, and how do branded types let you opt into nominal behavior when you need it?</summary>

TypeScript uses structural typing because it's designed to model how JavaScript actually works. In JS, you pass objects around based on what properties they have, not what constructor created them. TypeScript mirrors this: two types are compatible if their shapes match, regardless of their names.

**Practical consequences:**

```typescript
interface User { id: string; name: string }
interface Product { id: string; name: string }

function greetUser(user: User) { console.log(user.name); }

const product: Product = { id: "p1", name: "Widget" };
greetUser(product); // No error — shapes match
```

This is usually a feature (great for working with plain objects, JSON, duck typing), but it becomes a problem when two types have the same shape but different semantic meaning. A `UserId` and `ProductId` are both strings, but you never want to pass one where the other is expected.

**Branded types opt into nominal behavior:**

```typescript
type UserId = string & { readonly __brand: unique symbol };
type ProductId = string & { readonly __brand: unique symbol };

function createUserId(id: string): UserId { return id as UserId; }
function createProductId(id: string): ProductId { return id as ProductId; }

function getUser(id: UserId) { /* ... */ }

const userId = createUserId("u1");
const productId = createProductId("p1");

getUser(userId);     // OK
getUser(productId);  // Error — ProductId is not assignable to UserId
getUser("raw-id");   // Error — string is not assignable to UserId
```

The `& { readonly __brand: unique symbol }` intersection adds a phantom property that only exists at the type level (erased at runtime). Since each `unique symbol` is distinct, the two branded types are structurally incompatible even though they're both strings at runtime. This gives you nominal-style safety for domain identifiers, currency amounts, validated strings, etc.

</details>

<details>
<summary>2. Why does TypeScript's type system disappear entirely at runtime (type erasure) — what are the practical implications of having no runtime type information, why does this create a dangerous gap at system boundaries (API responses, user input, database results), and how do libraries like Zod and io-ts bridge this gap?</summary>

TypeScript compiles to plain JavaScript — all type annotations, interfaces, and generics are stripped during compilation. The runtime has zero knowledge of your types. This is by design: TypeScript's goal is to be a superset of JavaScript with no runtime overhead, not a new language with its own runtime.

**The system boundary problem:**

TypeScript can only guarantee types for code it compiles. Data crossing system boundaries — API responses, user input, database queries, message queue payloads — enters as `unknown` at runtime even if you've typed it. If you cast it with `as`, TypeScript trusts you blindly:

```typescript
interface User { id: string; name: string; email: string }

// Dangerous — no runtime check, TypeScript just trusts the cast
const user = (await fetch("/api/user")).json() as User;
// If the API returns { id: 123, name: null }, you now have
// a "User" with a number id and null name — silent corruption
```

This is where most TypeScript production bugs originate. The types say one thing, the data says another, and nothing catches the mismatch.

**Runtime validation with Zod:**

Zod (and io-ts) solve this by defining schemas that exist at runtime AND generate TypeScript types:

```typescript
import { z } from "zod";

const UserSchema = z.object({
  id: z.string(),
  name: z.string(),
  email: z.string().email(),
});

// Infer the TS type from the schema — single source of truth
type User = z.infer<typeof UserSchema>;

// Now validation actually happens at runtime
const result = UserSchema.safeParse(await response.json());
if (result.success) {
  // result.data is typed as User AND validated
  console.log(result.data.email);
} else {
  // Structured error telling you exactly what's wrong
  console.error(result.error.issues);
}
```

**The key insight**: validate at the boundaries, trust types internally. Every place where data enters your system (HTTP handlers, queue consumers, config loading, database results) should have runtime validation. Everything inside those boundaries can rely on TypeScript's compile-time checks.

</details>

<details>
<summary>3. How do any and unknown differ in TypeScript's type system — why does any break type safety by propagating through expressions, what are the common soundness holes in TypeScript (type assertions, index signatures, function parameter bivariance), and how does any sneak into a strict codebase despite strict mode being enabled?</summary>

`any` disables type checking entirely — it's assignable to every type AND every type is assignable to it. `unknown` is the type-safe counterpart: everything is assignable to `unknown`, but you can't do anything with an `unknown` value until you narrow it.

```typescript
// any — no checks, no errors, no safety
const a: any = "hello";
a.nonExistent.method(); // No error at compile time, crash at runtime
const num: number = a;  // No error — any flows into number silently

// unknown — must narrow before use
const u: unknown = "hello";
u.toString();           // Error — Object is of type 'unknown'
if (typeof u === "string") {
  u.toUpperCase();      // OK — narrowed to string
}
```

**Why `any` is dangerous — propagation:**

```typescript
function getConfig(): any { return JSON.parse("{}"); }

const config = getConfig();
const port: number = config.server.port; // No error — any propagates
// port is typed as number but could be undefined, string, anything
```

Once `any` enters an expression, it infects everything it touches. The return type of `config.server.port` is `any`, and assigning `any` to `number` is allowed. This silently breaks the type guarantees of every downstream consumer.

**Common soundness holes (even in strict mode):**

1. **Type assertions (`as`)**: `const x = someValue as SpecificType` — TypeScript trusts you completely, no runtime check.
2. **Index signatures**: `const obj: Record<string, number> = {}; obj["missing"]` returns `number`, not `number | undefined` (unless `noUncheckedIndexedAccess` is enabled).
3. **Function parameter bivariance**: Method syntax `method(x: Dog): void` allows both covariant and contravariant parameter types by default (fixed by `strictFunctionTypes`, but only for function syntax, not method syntax).

**How `any` sneaks into strict codebases:**

- **Untyped dependencies**: `npm install some-lib` with no `@types/some-lib` — all imports are `any`
- **JSON.parse()**: Returns `any` by default
- **`error` in catch blocks**: Was `any` before TS 4.4 (`useUnknownInCatchVariables`)
- **Generic inference fallback**: When TypeScript can't infer a generic, it sometimes falls back to `any`
- **Explicit `any` in library type definitions**: Even well-typed libraries sometimes use `any` in edge cases (e.g., `Array.prototype.find` on certain overloads)

**Defense**: Use `@typescript-eslint/no-explicit-any` and `@typescript-eslint/no-unsafe-*` rules to catch `any` at lint time, not just compile time.

</details>

## Conceptual Depth

<details>
<summary>4. What does TypeScript's strict mode actually enable — explain what strictNullChecks, noImplicitAny, and strictFunctionTypes each catch that would otherwise be silent bugs, why would you enable them individually vs all at once, and what's the practical migration strategy for a large existing codebase that has none of these flags?</summary>

`"strict": true` is a shorthand that enables a family of flags. The three most impactful:

**`strictNullChecks`** — makes `null` and `undefined` their own types instead of being assignable to everything:

```typescript
// Without: string can be null, no error accessing .length on null
function getLength(s: string) { return s.length; } // s could be null at runtime

// With: must handle nullability explicitly
function getLength(s: string | null) {
  if (s === null) return 0;
  return s.length; // safe — narrowed to string
}
```

This is the single most impactful flag. It catches the "billion dollar mistake" — null reference errors — at compile time.

**`noImplicitAny`** — requires explicit types when TypeScript can't infer them:

```typescript
// Without: params default to any silently
function process(data) { return data.items.map(i => i.name); } // all any

// With: Error — Parameter 'data' implicitly has an 'any' type
function process(data: { items: Array<{ name: string }> }) { /* ... */ }
```

Without this, functions without annotations silently become `any`-typed, creating invisible holes in your type coverage.

**`strictFunctionTypes`** — enforces contravariant parameter checking for function types:

```typescript
type Handler = (event: MouseEvent) => void;
const handler: Handler = (event: Event) => {}; // Error with strictFunctionTypes
// Without: this would be allowed, but handler might access MouseEvent-specific
// properties that don't exist on a generic Event
```

**Individual vs all at once:**

For a new project, always use `"strict": true`. For migrating an existing codebase, enable flags individually because each one produces a different category of errors and you want to fix them in focused PRs, not a 500-file changeset.

**Migration strategy for a large codebase:**

1. **Start with `noImplicitAny`** — highest signal-to-noise ratio, surfaces the worst gaps
2. **Then `strictNullChecks`** — the most errors but the most valuable. Use `// @ts-expect-error` temporarily for complex cases; track them as tech debt
3. **Then `strictFunctionTypes`** — usually fewer errors, mostly in callback/event handler code
4. **Finally `strict: true`** — enables remaining flags (`strictBindCallApply`, `strictPropertyInitialization`, `noImplicitThis`, `useUnknownInCatchVariables`, `alwaysStrict`)

At each stage: enable the flag, fix all errors (or suppress with `@ts-expect-error` + a tracking issue), ship, then move to the next flag. Each PR should be reviewable in isolation.

</details>

<details>
<summary>5. How do generics work in TypeScript beyond basic syntax — when and why would you use constraints (extends), defaults, and how do generic functions, interfaces, and classes differ in how they bind type parameters? What are the common mistakes with generics (over-constraining, unnecessary type parameters)?</summary>

Generics let you write code that works with multiple types while preserving type relationships. The key is understanding when type parameters are bound.

**Constraints (`extends`) — restrict what types are allowed:**

```typescript
// Without constraint: can't access .id because T could be anything
function getId<T>(entity: T) { return entity.id; } // Error

// With constraint: T must have an id property
function getId<T extends { id: string }>(entity: T): string {
  return entity.id; // OK — guaranteed by constraint
}
```

**Defaults — provide a fallback type:**

```typescript
interface ApiResponse<T = unknown> {
  data: T;
  status: number;
}

const generic: ApiResponse = { data: "anything", status: 200 }; // T defaults to unknown
const typed: ApiResponse<User> = { data: user, status: 200 };   // T is User
```

**Binding differences — when the type parameter is resolved:**

```typescript
// Generic function: T is bound at each CALL SITE
function identity<T>(x: T): T { return x; }
identity("hello"); // T = string
identity(42);      // T = number

// Generic interface: T is bound when the interface is USED as a type
interface Repository<T> {
  findById(id: string): T;
  save(entity: T): void;
}
const userRepo: Repository<User> = /* ... */; // T = User for all methods

// Generic class: T is bound at INSTANTIATION
class Cache<T> {
  private store = new Map<string, T>();
  get(key: string): T | undefined { return this.store.get(key); }
  set(key: string, value: T): void { this.store.set(key, value); }
}
const cache = new Cache<User>(); // T = User for all methods
```

The key difference: generic functions get fresh type parameters per call, while generic interfaces/classes lock in the type for all members.

**Common mistakes:**

1. **Unnecessary type parameters** — if the type parameter is only used once, you don't need a generic:
```typescript
// Bad: T is only used in the parameter, not related to return
function print<T>(x: T): void { console.log(x); }
// Better: just use unknown
function print(x: unknown): void { console.log(x); }
```

2. **Over-constraining** — making constraints more restrictive than needed:
```typescript
// Over-constrained: requires the full User type
function getName<T extends User>(entity: T): string { return entity.name; }
// Better: only require what you use
function getName<T extends { name: string }>(entity: T): string { return entity.name; }
```

3. **Not using the constraint** — adding `extends` but then using `as` to cast anyway, which defeats the purpose.

**Rule of thumb**: Use generics when you need to express a relationship between inputs and outputs (return type depends on parameter type, or two parameters must share a type). If you just need "accepts anything," use `unknown`.

</details>

<details>
<summary>6. How do conditional types work in TypeScript — what does the `Type extends Condition ? A : B` pattern enable, how does `infer` extract types from complex structures, why does `never` behave unexpectedly in conditional types, and what is distributive behavior over union types and when would you disable it?</summary>

Conditional types are TypeScript's "if/else" at the type level. They enable types that change shape based on other types.

**Basic pattern:**

```typescript
type IsString<T> = T extends string ? true : false;

type A = IsString<string>;  // true
type B = IsString<number>;  // false
```

**`infer` — pattern matching to extract types:**

`infer` declares a type variable inside the `extends` clause that TypeScript fills in by matching the structure:

```typescript
// Extract the return type of a function
type ReturnOf<T> = T extends (...args: any[]) => infer R ? R : never;

type A = ReturnOf<() => string>;        // string
type B = ReturnOf<(x: number) => boolean>; // boolean

// Extract the element type of an array
type ElementOf<T> = T extends (infer E)[] ? E : never;

type C = ElementOf<string[]>;   // string
type D = ElementOf<number[]>;   // number

// Extract the resolved type of a Promise
type Awaited<T> = T extends Promise<infer U> ? Awaited<U> : T;

type E = Awaited<Promise<Promise<string>>>; // string (recursive unwrap)
```

**`never` behaves unexpectedly — it vanishes:**

`never` represents the empty union (no members). When passed as a bare type parameter to a distributive conditional type, there's nothing to distribute over, so the result is `never` — not either branch:

```typescript
type IsString<T> = T extends string ? "yes" : "no";

type X = IsString<never>; // never — not "yes" or "no"!
```

This trips people up when filtering types. If your conditional type might receive `never`, guard against it:

```typescript
type SafeIsString<T> = [T] extends [never] ? "empty" : T extends string ? "yes" : "no";
```

**Distributive behavior — conditional types distribute over unions:**

When `T` is a naked (unwrapped) type parameter and you pass a union, the conditional type is applied to each member individually:

```typescript
type NonNullable<T> = T extends null | undefined ? never : T;

type A = NonNullable<string | null | undefined>;
// Distributes: (string extends null|undefined ? never : string) | (null extends null|undefined ? never : null) | ...
// Result: string
```

**Disabling distribution** — wrap the type parameter in a tuple:

```typescript
// Distributive: checks each union member
type Dist<T> = T extends string ? "str" : "other";
type A = Dist<string | number>; // "str" | "other"

// Non-distributive: checks the union as a whole
type NoDist<T> = [T] extends [string] ? "str" : "other";
type B = NoDist<string | number>; // "other" (string | number doesn't extend string)
```

Disable distribution when you want to check the entire union as a unit rather than checking each member separately — common when building type equality checks or testing if a type is `never`.

</details>

<details>
<summary>7. How do mapped types work in TypeScript — what is key remapping with `as`, how do you build custom mapped types, and how do the built-in utility types (Partial, Required, Pick, Omit, Record) use mapped types under the hood? When would you build a custom mapped type vs compose existing utility types?</summary>

Mapped types iterate over a set of keys and produce a new type by transforming each property. They're TypeScript's "loop" at the type level.

**Basic syntax:**

```typescript
type Mapped<T> = {
  [K in keyof T]: T[K] // iterate over keys, preserve value types
};
```

**How built-in utility types work under the hood:**

```typescript
// Partial — make all properties optional
type Partial<T> = { [K in keyof T]?: T[K] };

// Required — make all properties required
type Required<T> = { [K in keyof T]-?: T[K] }; // -? removes optionality

// Readonly
type Readonly<T> = { readonly [K in keyof T]: T[K] };

// Pick — select specific keys
type Pick<T, K extends keyof T> = { [P in K]: T[P] };

// Record — create a type with specific keys and value type
type Record<K extends keyof any, V> = { [P in K]: V };

// Omit — Pick all keys EXCEPT the specified ones (uses Exclude helper)
type Omit<T, K extends keyof any> = Pick<T, Exclude<keyof T, K>>;
```

**Key remapping with `as` — transform keys during iteration:**

```typescript
// Prefix all keys with "get"
type Getters<T> = {
  [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K]
};

interface User { name: string; age: number }
type UserGetters = Getters<User>;
// { getName: () => string; getAge: () => number }

// Filter out keys by remapping to never
type OnlyStrings<T> = {
  [K in keyof T as T[K] extends string ? K : never]: T[K]
};

type StringProps = OnlyStrings<{ name: string; age: number; email: string }>;
// { name: string; email: string }
```

**Custom mapped type example — make specific keys required, rest optional:**

```typescript
type RequireKeys<T, K extends keyof T> = Required<Pick<T, K>> & Partial<Omit<T, K>>;

interface Config {
  host: string;
  port: number;
  debug?: boolean;
  logLevel?: string;
}

// host and port required, everything else optional
type MinimalConfig = RequireKeys<Config, "host" | "port">;
```

**When to build custom vs compose existing:**

- **Compose existing** when you can express it as a combination of Pick/Omit/Partial/Required — it's more readable and maintainable
- **Build custom** when you need key remapping, key filtering by value type, template literal key transformations, or conditional value transformations — things that can't be expressed by composing utility types

</details>

<details>
<summary>8. Why are discriminated unions one of TypeScript's most powerful patterns — how does a shared literal discriminant field enable exhaustiveness checking, how does the `never` trick in a switch default catch missing cases at compile time, and when are discriminated unions a better fit than class hierarchies or overloads?</summary>

Discriminated unions combine union types with a shared literal property (the discriminant) so TypeScript can narrow the type in each branch and verify you've handled all cases.

**The pattern:**

```typescript
type RequestState =
  | { status: "idle" }
  | { status: "loading" }
  | { status: "success"; data: string[] }
  | { status: "error"; error: Error };

function render(state: RequestState) {
  switch (state.status) {
    case "idle": return "Ready";
    case "loading": return "Loading...";
    case "success": return state.data.join(", "); // TS knows data exists
    case "error": return state.error.message;      // TS knows error exists
  }
}
```

TypeScript narrows the union based on the discriminant value. In the `"success"` branch, it knows `state` is `{ status: "success"; data: string[] }`, so accessing `.data` is safe.

**The `never` trick for exhaustiveness:**

```typescript
function assertNever(x: never): never {
  throw new Error(`Unexpected value: ${x}`);
}

function render(state: RequestState): string {
  switch (state.status) {
    case "idle": return "Ready";
    case "loading": return "Loading...";
    case "success": return state.data.join(", ");
    // case "error" is missing!
    default: return assertNever(state);
    // Compile error: Argument of type '{ status: "error"; error: Error }'
    // is not assignable to parameter of type 'never'
  }
}
```

After handling all cases, the type in `default` should be `never` (no remaining variants). If you miss a case, the remaining variant type can't be assigned to `never`, and you get a compile error. When someone adds a new variant later, every switch statement that uses this pattern immediately shows an error.

**When discriminated unions beat class hierarchies and overloads:**

- **Over class hierarchies**: Unions are plain data (serializable, no `instanceof` dependency). They work naturally with Redux/reducers, API responses, and state machines. Class hierarchies require runtime prototype chains, can't be serialized to JSON directly, and force OOP patterns where they may not fit.
- **Over overloads**: When different input shapes produce different output shapes, discriminated unions make the relationship explicit and checkable. Overloads hide the relationship inside implementation signatures.
- **For state modeling**: When an entity can be in one of N states with different associated data per state, discriminated unions make invalid states unrepresentable. A single interface with optional fields can't express "data exists only when status is success."

</details>

<details>
<summary>9. How does TypeScript narrow types within control flow — what are the mechanisms (typeof, instanceof, `in`, truthiness), when do you need a custom type guard function (paramName is Type) vs an assertion function (asserts paramName is Type), and what are the pitfalls where narrowing silently fails?</summary>

Type narrowing is how TypeScript refines a broad type to a more specific one within a code block based on runtime checks.

**Built-in narrowing mechanisms:**

```typescript
function process(value: string | number | null | { kind: "a" } | { kind: "b"; data: string }) {
  // typeof — works for primitives
  if (typeof value === "string") {
    value.toUpperCase(); // narrowed to string
  }

  // truthiness — eliminates null/undefined/0/""/false
  if (value) {
    // narrowed to string | number | { kind: "a" } | { kind: "b"; data: string }
  }

  // instanceof — works for class instances
  if (value instanceof Date) { /* narrowed to Date */ }

  // "in" operator — checks for property existence
  if (typeof value === "object" && value !== null && "data" in value) {
    // narrowed to { kind: "b"; data: string }
  }

  // equality — narrows both sides
  if (value === null) { /* narrowed to null */ }
}
```

**Custom type guard — when built-in checks aren't enough:**

```typescript
interface Fish { swim(): void }
interface Bird { fly(): void }

// Type guard: returns boolean but annotates the narrowing effect
function isFish(animal: Fish | Bird): animal is Fish {
  return "swim" in animal;
}

const pet: Fish | Bird = getAnimal();
if (isFish(pet)) {
  pet.swim(); // narrowed to Fish
} else {
  pet.fly();  // narrowed to Bird
}
```

Use a type guard when: the check is complex (multi-property validation), you want to reuse the check, or TypeScript can't infer the narrowing from inline code.

**Assertion function — narrows by throwing on failure:**

```typescript
function assertIsString(value: unknown): asserts value is string {
  if (typeof value !== "string") {
    throw new Error(`Expected string, got ${typeof value}`);
  }
}

function process(input: unknown) {
  assertIsString(input);
  // After assertion, input is narrowed to string for the rest of the scope
  input.toUpperCase(); // OK
}
```

The difference: a type guard returns a boolean (use in `if` conditions), an assertion function throws on failure and narrows for the remaining scope (use for precondition checks at the top of functions).

**Pitfalls where narrowing silently fails:**

1. **Narrowing doesn't survive callbacks:**
```typescript
function process(value: string | null) {
  if (value !== null) {
    setTimeout(() => {
      value.toUpperCase(); // Error in strict — value could be null
      // TS can't guarantee value wasn't reassigned before callback runs
    }, 100);
  }
}
```

2. **Truthiness narrows out 0 and empty string:**
```typescript
function process(value: number | null) {
  if (value) {
    // This eliminates null BUT ALSO 0 — which might be a valid number
  }
  // Better: explicit null check
  if (value !== null) { /* 0 is preserved */ }
}
```

3. **Type guards can lie:**
```typescript
function isString(x: unknown): x is string {
  return true; // Always says "yes" — TypeScript trusts you
}
const num = 42;
if (isString(num)) {
  num.toUpperCase(); // No compile error, runtime crash
}
```
TypeScript has no way to verify that a type guard's implementation matches its annotation. A lying type guard is worse than no type guard.

4. **Property access doesn't narrow the parent object:**
```typescript
type A = { kind: "a"; data: string };
type B = { kind: "b" };
type U = A | B;

function process(item: U) {
  const kind = item.kind;
  if (kind === "a") {
    item.data; // Error — item is still U, not narrowed to A
    // Must check item.kind directly, not through a variable
  }
}
```

</details>

<details>
<summary>10. Why do TypeScript developers often prefer union types over enums — what are the pitfalls of numeric enums (reverse mapping, widening), when do const enums help and what are their limitations with --isolatedModules, and when are string enums genuinely the better choice?</summary>

**Why unions are often preferred:**

```typescript
// Union — zero runtime overhead, just a type
type Status = "active" | "inactive" | "pending";

// Enum — generates runtime JavaScript object
enum Status { Active = "active", Inactive = "inactive", Pending = "pending" }
```

Unions are pure types (erased at compile time), work naturally with string literals, have excellent IDE autocomplete, and need no import. They're simpler and fit TypeScript's philosophy of not adding runtime constructs.

**Numeric enum pitfalls:**

```typescript
enum Direction { Up, Down, Left, Right } // 0, 1, 2, 3

// Reverse mapping — the compiled object maps both ways
// Direction[0] === "Up" and Direction["Up"] === 0
// This means the runtime object is larger than expected

// Widening — any number is assignable to a numeric enum
function move(dir: Direction) { /* ... */ }
move(999); // No error! TypeScript allows any number
// This completely defeats the purpose of an enum
```

The widening issue is the killer problem. A numeric enum that accepts any number provides almost no type safety.

**`const` enums — inlined at compile time:**

```typescript
const enum HttpStatus {
  OK = 200,
  NotFound = 404,
  ServerError = 500,
}

// Compiled output: the enum disappears, values are inlined
const status = 200; // instead of HttpStatus.OK
```

This eliminates the runtime object and reverse mapping, but has a major limitation: **`--isolatedModules` (required by most bundlers and `ts-jest`) forbids `const` enums** because each file is compiled independently and can't inline values from other files. Since most modern TypeScript projects use `isolatedModules`, `const` enums are effectively unusable across module boundaries.

**When string enums are genuinely better:**

String enums make sense when you need:

1. **Runtime iteration over values** — e.g., populating a dropdown from enum values, or validating that a string is a member of the set:
```typescript
enum Role { Admin = "admin", User = "user", Guest = "guest" }

// You can iterate over values at runtime
const validRoles = Object.values(Role);
function isValidRole(s: string): s is Role {
  return validRoles.includes(s as Role);
}
```

2. **Namespace-like grouping** — when you want the enum to act as both a type and a namespace (e.g., `Role.Admin` as a value and `Role` as a type).

3. **Third-party API alignment** — when an external API defines enum-like constants and you want a 1:1 mapping.

**Practical recommendation:** Default to union types. Use string enums only when you need runtime access to the set of values. Avoid numeric enums entirely. If you want an object with union-type values, use `as const`:

```typescript
const Status = {
  Active: "active",
  Inactive: "inactive",
  Pending: "pending",
} as const;

type Status = typeof Status[keyof typeof Status]; // "active" | "inactive" | "pending"
// Best of both: union type + runtime object for iteration
```

</details>

<details>
<summary>11. When should you use function overloads in TypeScript vs union parameter types vs generics — what specific problem do overloads solve that the alternatives can't, what are the implementation signature rules, and what are the common mistakes (too many overloads, implementation signature mismatches)?</summary>

**The specific problem overloads solve**: when the return type depends on which parameter type was passed, and you want TypeScript to narrow the return type at each call site.

```typescript
// Overloads: return type varies based on input type
function parse(input: string): Document;
function parse(input: Buffer): Document;
function parse(input: string | Buffer): Document {
  // implementation handles both
  return typeof input === "string" ? parseString(input) : parseBuffer(input);
}

const doc1 = parse("html");    // TypeScript knows: Document
const doc2 = parse(buffer);    // TypeScript knows: Document
```

A more compelling case — different return types:

```typescript
function createElement(tag: "input"): HTMLInputElement;
function createElement(tag: "div"): HTMLDivElement;
function createElement(tag: string): HTMLElement;
function createElement(tag: string): HTMLElement {
  return document.createElement(tag);
}

const input = createElement("input"); // HTMLInputElement, not just HTMLElement
```

**When to use each alternative:**

- **Union parameters**: When the return type is the same regardless of which parameter type is passed. As shown in the TypeScript docs, if overloads differ only in parameter type, prefer unions — they're simpler and compose better (e.g., passing values through to other functions).
- **Generics**: When the return type mirrors or derives from the input type — `function identity<T>(x: T): T`. The relationship is structural, not a fixed mapping.
- **Overloads**: When there's a fixed mapping between specific input types and specific return types that generics can't express.

**Implementation signature rules:**

1. The implementation signature is **not callable directly** — callers only see the overload signatures
2. The implementation signature must be **compatible with all overloads** — it's typically the widest union
3. Overloads are resolved **top-to-bottom** — put the most specific signatures first

**Common mistakes:**

1. **Too many overloads** when a union would suffice:
```typescript
// Bad — overloads add complexity with no type-narrowing benefit
function log(msg: string): void;
function log(msg: number): void;
function log(msg: string | number): void { console.log(msg); }

// Better
function log(msg: string | number): void { console.log(msg); }
```

2. **Implementation signature too narrow** — if the implementation doesn't cover all overload types, you get a compile error.

3. **Overloads that could be a generic**:
```typescript
// Overloads that just echo the type back — use a generic instead
function wrap(x: string): string[];
function wrap(x: number): number[];
function wrap(x: string | number) { return [x]; }

// Better
function wrap<T>(x: T): T[] { return [x]; }
```

**Rule of thumb**: If the return type depends on a finite set of specific input types, use overloads. If the return type structurally relates to the input type, use generics. If the return type doesn't change, use unions.

</details>

<details>
<summary>12. Why does TypeScript have multiple module resolution strategies (node10, node16, bundler) — what problem does each solve, how does node16 resolution differ for ESM vs CJS imports, and what are the common interop headaches when mixing ESM and CJS packages in a Node.js TypeScript project?</summary>

Each strategy mirrors how a different runtime or tool resolves imports. Using the wrong one means TypeScript finds types that don't match what the runtime actually loads.

**`node10` (formerly `node`)** — mirrors Node.js's original CJS resolution:
- Tries adding `.js`, `.ts`, `/index.js`, `/index.ts` to bare specifiers
- Looks up `node_modules` with `main` field in package.json
- No support for `exports` field, no ESM semantics
- **Problem it solves**: Legacy CJS projects. Still the default if you don't set it explicitly, which causes confusion.

**`node16` / `nodenext`** — mirrors Node.js's dual ESM/CJS resolution:
- Respects `package.json` `exports` field and conditions (`import` vs `require`)
- In ESM files (`.mts` or `.ts` in a `"type": "module"` package): **requires file extensions** on relative imports (`import "./utils.js"`, not `"./utils"`)
- In CJS files (`.cts` or `.ts` in a `"type": "commonjs"` package): extensionless imports still work
- **Problem it solves**: Accurate modeling of how Node.js 16+ actually resolves modules, including the ESM/CJS dual system

**`bundler`** — mirrors modern bundler resolution (Vite, esbuild, webpack):
- Supports `exports` field and `import` conditions like `node16`
- But relaxes the extension requirement — extensionless imports work
- No CJS/ESM distinction in resolution — bundlers handle both uniformly
- **Problem it solves**: Projects using bundlers that don't enforce Node.js's strict ESM rules

**How `node16` differs for ESM vs CJS:**

```typescript
// In an ESM file (.mts or "type": "module")
import { helper } from "./utils.js";   // Must include .js extension
import pkg from "some-package";         // Resolves via "import" condition in exports

// In a CJS file (.cts or "type": "commonjs")
import { helper } from "./utils";      // Extensionless OK
import pkg from "some-package";         // Resolves via "require" condition in exports
```

**Common interop headaches:**

1. **`ERR_REQUIRE_ESM`** — Your CJS code tries to `require()` an ESM-only package (e.g., `node-fetch` v3, `chalk` v5). Fix: use dynamic `import()`, switch your project to ESM, or pin an older CJS version of the dependency.

2. **`ERR_MODULE_NOT_FOUND`** — Missing file extension in ESM. TypeScript compiles `import "./utils"` to `import "./utils"` in JS output, and Node.js ESM won't resolve it. Fix: write `import "./utils.js"` in your `.ts` file (yes, `.js`, even though the source is `.ts`).

3. **Default export mismatch** — CJS modules have `module.exports`, not `export default`. When importing a CJS module in ESM, the entire `module.exports` becomes the default. This means `import { named } from "cjs-pkg"` may not work — you might need `import pkg from "cjs-pkg"; const { named } = pkg;`.

4. **`exports` field blocking deep imports** — A package with `"exports": { ".": "./dist/index.js" }` blocks `import "pkg/internal"`. With `node10` this worked (no `exports` support), but `node16` respects it.

**Practical recommendation**: Use `"moduleResolution": "node16"` for pure Node.js projects (it matches what the runtime actually does). Use `"bundler"` for projects using Vite, esbuild, or webpack. Avoid `node10` for new projects — it's outdated and will silently resolve things differently than your runtime.

</details><details>
<summary>13. Why does TypeScript offer three assertion patterns — satisfies, as const, and explicit type annotations — what does each one do differently to the inferred type, when should you use each, and what goes wrong when you use `as` (type assertion) where you should have used `satisfies`?</summary>

Each pattern has a different effect on the inferred type of a value:

**Explicit type annotation — widens to the declared type:**

```typescript
type Config = { port: number; host: string };

const config: Config = { port: 3000, host: "localhost" };
// Type of config: Config
// TypeScript forgets the literal values — config.port is `number`, not `3000`
```

Use when: you want to enforce a type contract and don't need literal types.

**`as const` — narrows to the most specific literal type:**

```typescript
const config = { port: 3000, host: "localhost" } as const;
// Type: { readonly port: 3000; readonly host: "localhost" }
// All properties are readonly, all values are literal types
```

Use when: you want literal types (for discriminants, route maps, etc.) and immutability. Essential for the `const object + typeof` pattern (as shown in question 10 for enum alternatives).

**`satisfies` — validates against a type WITHOUT widening:**

```typescript
type Config = { port: number; host: string };

const config = { port: 3000, host: "localhost" } satisfies Config;
// Type: { port: number; host: string } — validated against Config
// But TypeScript retains the narrower inferred types for autocomplete
// config.host is still string (not widened to Config["host"])
```

The real power shows when the target type has unions:

```typescript
type Color = Record<string, string | number[]>;

const palette = {
  red: "#ff0000",
  green: [0, 255, 0],
} satisfies Color;

palette.red.toUpperCase();  // OK — TypeScript knows red is string
palette.green.map(x => x);  // OK — TypeScript knows green is number[]

// With annotation instead:
const palette2: Color = { red: "#ff0000", green: [0, 255, 0] };
palette2.red.toUpperCase();  // Error — red is string | number[]
```

`satisfies` validates the value matches the type but preserves the narrower inferred type for each property. An annotation would widen everything to the declared type.

**What goes wrong with `as` instead of `satisfies`:**

```typescript
type Config = { port: number; host: string; debug: boolean };

// as — LIES to the compiler, no validation
const config = { port: 3000, host: "localhost" } as Config;
// No error! debug is missing, but TypeScript trusts you
// config.debug is typed as boolean but is actually undefined at runtime

// satisfies — validates the value
const config2 = { port: 3000, host: "localhost" } satisfies Config;
// Error: Property 'debug' is missing
```

`as` is a type assertion — it overrides TypeScript's inference and says "trust me." It performs no structural validation. `satisfies` actually checks that the value matches the type and reports errors if it doesn't.

**Decision table:**

| Pattern | Validates shape? | Preserves literals? | Use case |
|---|---|---|---|
| `: Type` | Yes | No (widens) | Function params, class properties, variable contracts |
| `as const` | No | Yes (narrows to literals + readonly) | Constants, lookup tables, discriminant values |
| `satisfies Type` | Yes | Yes (partially — keeps inferred types) | Config objects, maps where you want both validation and narrow types |
| `as Type` | No | No | Last resort — casting external data (prefer Zod), test mocks |

</details><details>
<summary>14. How do TypeScript's type operators (keyof, typeof at the type level, and indexed access types T[K]) work together to enable type-safe property access — show how combining keyof with indexed access creates the pattern behind Pick and Record, how typeof extracts types from runtime values, and what are the common mistakes when using these operators (e.g., typeof on a type vs a value)?</summary>

These three operators are the building blocks for most advanced TypeScript patterns.

**`keyof` — extracts the keys of a type as a union:**

```typescript
interface User { id: string; name: string; age: number }

type UserKeys = keyof User; // "id" | "name" | "age"
```

**Indexed access types `T[K]` — looks up the type of a property:**

```typescript
type UserId = User["id"];           // string
type UserNameOrAge = User["name" | "age"]; // string | number
type AnyUserProp = User[keyof User];       // string | number
```

**`typeof` at the type level — extracts the type from a runtime value:**

```typescript
const config = { port: 3000, host: "localhost", debug: false };

type Config = typeof config;
// { port: number; host: string; debug: boolean }

// Combined with as const for literal types:
const ROLES = ["admin", "user", "guest"] as const;
type Role = typeof ROLES[number]; // "admin" | "user" | "guest"
```

**How they combine to create Pick and Record:**

```typescript
// Pick uses keyof constraint + indexed access
type Pick<T, K extends keyof T> = {
  [P in K]: T[P]  // P iterates over selected keys, T[P] gets the value type
};

// Record uses keyof any (string | number | symbol) for key constraint
type Record<K extends keyof any, V> = {
  [P in K]: V
};

// The type-safe property getter pattern:
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}

const user: User = { id: "1", name: "Alice", age: 30 };
const name = getProperty(user, "name");    // string — T[K] resolves to User["name"]
const age = getProperty(user, "age");      // number
getProperty(user, "email");                // Error — "email" not in keyof User
```

This is the fundamental pattern: `K extends keyof T` constrains the key, `T[K]` gives the matching value type.

**Common mistakes:**

1. **`typeof` on a type instead of a value:**
```typescript
interface User { name: string }
type T = typeof User; // Error — User is a type, not a value
// typeof operates on values. Use the type directly: type T = User
```

2. **Confusing runtime `typeof` with type-level `typeof`:**
```typescript
const x = "hello";
if (typeof x === "string") { }    // Runtime typeof — returns "string" at runtime
type T = typeof x;                 // Type-level typeof — evaluates to string type
// They look the same but operate in completely different contexts
```

3. **Forgetting `keyof` returns a union, not an array:**
```typescript
type Keys = keyof User; // "id" | "name" | "age" — a union type
// You can't iterate over it at runtime. For runtime keys, use Object.keys()
```

4. **Index access on optional properties without handling undefined:**
```typescript
interface Config { debug?: boolean }
type Debug = Config["debug"]; // boolean | undefined — the optionality carries through
```

</details>

<details>
<summary>15. How do you type error handling in TypeScript — why does TypeScript type catch blocks as unknown (and why did it used to be any), what are the tradeoffs between Result/Either return types vs thrown exceptions for error handling, how do discriminated union error types work as an alternative to exception hierarchies, and why doesn't TypeScript have checked exceptions like Java?</summary>

**Why `catch` is `unknown` (and was `any`):**

Before TS 4.4, `catch (e)` typed `e` as `any` — a pragmatic default since anything can be thrown in JavaScript (strings, numbers, objects, not just `Error`). But `any` meant you could access `e.message` without checking, leading to runtime crashes when non-Error values were thrown.

TS 4.4 added `useUnknownInCatchVariables` (included in `strict`), making catch variables `unknown`:

```typescript
try {
  JSON.parse(input);
} catch (e) {
  // e is unknown — must narrow before use
  if (e instanceof Error) {
    console.error(e.message); // safe
  } else {
    console.error("Non-error thrown:", e);
  }
}
```

**Result/Either pattern vs thrown exceptions:**

```typescript
// Result pattern — errors as values
type Result<T, E = Error> =
  | { ok: true; data: T }
  | { ok: false; error: E };

function parseConfig(raw: string): Result<Config, "invalid_json" | "missing_field"> {
  try {
    const parsed = JSON.parse(raw);
    if (!parsed.host) return { ok: false, error: "missing_field" };
    return { ok: true, data: parsed as Config };
  } catch {
    return { ok: false, error: "invalid_json" };
  }
}

const result = parseConfig(input);
if (result.ok) {
  console.log(result.data.host); // type-safe access
} else {
  // result.error is "invalid_json" | "missing_field" — exhaustively handleable
}
```

**Tradeoffs:**

| | Result/Either | Thrown Exceptions |
|---|---|---|
| **Type safety** | Full — error types visible in signatures | None — `throws` not part of function type |
| **Caller awareness** | Forced to handle (can't access `.data` without checking `.ok`) | Easy to forget `try/catch` |
| **Composition** | Chainable with map/flatMap, explicit error propagation | Implicit propagation up the stack |
| **Performance** | No stack unwinding, cheap | Stack trace creation has cost |
| **Ergonomics** | Verbose for deeply nested calls | Clean happy path with `try/catch` |
| **Ecosystem fit** | Not standard in Node.js ecosystem | Node.js APIs, libraries all throw |

**Discriminated union error types:**

```typescript
type AppError =
  | { type: "not_found"; resource: string; id: string }
  | { type: "validation"; fields: string[] }
  | { type: "unauthorized"; reason: string };

function handleError(error: AppError): string {
  switch (error.type) {
    case "not_found": return `${error.resource} ${error.id} not found`;
    case "validation": return `Invalid fields: ${error.fields.join(", ")}`;
    case "unauthorized": return `Access denied: ${error.reason}`;
  }
}
```

This gives you exhaustiveness checking (as covered in question 8), specific data per error type, and no class hierarchy needed. Each error variant carries exactly the data relevant to it.

**Why TypeScript doesn't have checked exceptions:**

1. **JavaScript can throw anything** — not just Error instances. There's no way to statically verify what a function throws.
2. **Structural typing can't model it** — checked exceptions would need nominal "exception types" that don't fit TypeScript's structural system.
3. **Composition breaks down** — a function calling three functions that each throw different exceptions would need to declare all of them, creating the same verbose signature problem Java has.
4. **Design philosophy** — TypeScript aims to type JavaScript as it is, not impose new runtime semantics. Checked exceptions would require changes to how JavaScript works, not just types.

**Practical recommendation**: Use thrown exceptions for truly exceptional cases (network failures, programmer errors). Use Result types for expected failure modes in domain logic (validation, business rules) where the caller should handle each case. Use discriminated union errors when you have a known set of error types with different associated data.

</details>## Practical — Type System Patterns

<details>
<summary>16. Build a generic repository interface in TypeScript that constrains the entity type to have an id field — show how the constraint prevents invalid usage, how you'd add generic methods (findById, create, update) that preserve type safety, and what happens if a consumer passes a type that doesn't satisfy the constraint</summary>

```typescript
// Base constraint: every entity must have a string id
interface Entity {
  id: string;
}

interface Repository<T extends Entity> {
  findById(id: string): Promise<T | null>;
  findAll(): Promise<T[]>;
  create(data: Omit<T, "id">): Promise<T>;      // id is generated, not provided
  update(id: string, data: Partial<Omit<T, "id">>): Promise<T>;
  delete(id: string): Promise<void>;
}
```

**Type safety in action:**

```typescript
interface User extends Entity {
  name: string;
  email: string;
  role: "admin" | "user";
}

class UserRepository implements Repository<User> {
  async findById(id: string): Promise<User | null> { /* ... */ }
  async findAll(): Promise<User[]> { /* ... */ }

  async create(data: Omit<User, "id">): Promise<User> {
    // data is { name: string; email: string; role: "admin" | "user" }
    // id is excluded — TypeScript enforces this
    const user: User = { id: generateId(), ...data };
    return user;
  }

  async update(id: string, data: Partial<Omit<User, "id">>): Promise<User> {
    // data is { name?: string; email?: string; role?: "admin" | "user" }
    // Can't accidentally update the id
    return { /* ... */ } as User;
  }

  async delete(id: string): Promise<void> { /* ... */ }
}

const repo = new UserRepository();

// Type-safe usage:
await repo.create({ name: "Alice", email: "a@b.com", role: "admin" }); // OK
await repo.create({ name: "Alice" }); // Error — missing email and role
await repo.update("1", { role: "guest" }); // Error — "guest" not in role union
```

**What happens with an invalid type:**

```typescript
// No id field
interface LogEntry {
  timestamp: Date;
  message: string;
}

class LogRepository implements Repository<LogEntry> {
// Error: Type 'LogEntry' does not satisfy the constraint 'Entity'.
//   Property 'id' is missing in type 'LogEntry' but required in type 'Entity'.
}

// Wrong id type
interface Product {
  id: number; // number, not string
  name: string;
}

class ProductRepository implements Repository<Product> {
// Error: Type 'Product' does not satisfy the constraint 'Entity'.
//   Types of property 'id' are incompatible.
}
```

The `T extends Entity` constraint catches invalid types at the point of use, with clear error messages explaining exactly what's missing or incompatible. This is the generic constraint pattern from question 5 applied to a real-world interface.

</details>

<details>
<summary>17. Write a generic pipe or compose function in TypeScript that correctly infers the return type through a chain of functions — show how TypeScript infers the intermediate and final types, what limitations you hit with variadic generics, and how this pattern appears in real libraries (e.g., middleware chains, validation pipelines)</summary>

A type-safe `pipe` needs overloads because TypeScript can't express "the output type of function N is the input type of function N+1" with a single generic signature for arbitrary lengths.

**Pipe with overloads (the practical approach):**

```typescript
function pipe<A, B>(value: A, fn1: (a: A) => B): B;
function pipe<A, B, C>(value: A, fn1: (a: A) => B, fn2: (b: B) => C): C;
function pipe<A, B, C, D>(value: A, fn1: (a: A) => B, fn2: (b: B) => C, fn3: (c: C) => D): D;
function pipe<A, B, C, D, E>(
  value: A, fn1: (a: A) => B, fn2: (b: B) => C, fn3: (c: C) => D, fn4: (d: D) => E
): E;
function pipe(value: unknown, ...fns: Function[]): unknown {
  return fns.reduce((acc, fn) => fn(acc), value);
}

// TypeScript infers every intermediate type:
const result = pipe(
  "  hello world  ",
  (s: string) => s.trim(),          // string → string
  (s) => s.split(" "),              // string → string[] (inferred)
  (arr) => arr.length,              // string[] → number (inferred)
);
// result: number — fully inferred through the chain
```

**Why variadic generics don't fully solve this:**

TypeScript's variadic tuple types (`[...T]`) can describe "an array of types," but they can't express the chaining constraint: "element N's output matches element N+1's input." You'd need dependent types or higher-kinded types to express this generically, neither of which TypeScript has.

```typescript
// This DOESN'T work — no way to connect adjacent function types
function pipe<T extends [...any[]]>(value: T[0], ...fns: ???): ???
```

So real libraries use overloads (typically 10-20 overloads to cover common arities) with a fallback to `unknown` for longer chains.

**Where this pattern appears in real libraries:**

**Express/Koa middleware chains** — each middleware transforms the context:
```typescript
// Conceptually: pipe(request, auth, validate, handle)
// Each middleware adds to the context (req.user, req.body parsed, etc.)
app.use(authMiddleware);    // adds req.user
app.use(validateBody(schema)); // adds req.validatedBody
app.get("/users", handler);    // handler sees the enriched type
```

**Zod transform pipelines:**
```typescript
const schema = z.string()
  .transform((s) => s.trim())           // string → string
  .transform((s) => parseInt(s, 10))    // string → number
  .refine((n) => n > 0);               // number → number (validated)
// Zod infers the final output type: number
```

**fp-ts pipe:**
```typescript
import { pipe } from "fp-ts/function";
// Uses the same overload approach — defines overloads up to 20 functions
const result = pipe(
  5,
  (n: number) => n * 2,
  (n) => `Value: ${n}`,
); // "Value: 10"
```

**Limitation**: Once you exceed the number of overloads (usually 10-20 functions), TypeScript falls back to `unknown` and you lose type inference. In practice, chains longer than 5-6 steps usually indicate the need for intermediate variables anyway.

</details>

<details>
<summary>18. Write `infer`-based conditional types that extract the return type of a Promise (unwrapping nested Promises) and extract function parameter types — show how `infer` works in each case and explain what happens when you pass a union type to each</summary>

**Deep Promise unwrapping:**

```typescript
// Recursively unwrap nested Promises
type DeepAwaited<T> = T extends Promise<infer U> ? DeepAwaited<U> : T;

type A = DeepAwaited<Promise<string>>;                    // string
type B = DeepAwaited<Promise<Promise<number>>>;           // number
type C = DeepAwaited<Promise<Promise<Promise<boolean>>>>; // boolean
type D = DeepAwaited<string>;                             // string (not a Promise, returned as-is)
```

How it works: `T extends Promise<infer U>` pattern-matches `T` against `Promise<something>`. If it matches, `U` captures the inner type, and the type recurses. If not, `T` is the base case.

This is how TypeScript's built-in `Awaited<T>` works (added in TS 4.5), with additional handling for thenables.

**Extract function parameter types:**

```typescript
// Extract all parameters as a tuple
type Params<T> = T extends (...args: infer P) => any ? P : never;

type A = Params<(x: string, y: number) => void>;  // [x: string, y: number]
type B = Params<() => void>;                        // []
type C = Params<(req: Request) => Response>;        // [req: Request]

// Extract a specific parameter by index
type FirstParam<T> = T extends (first: infer F, ...rest: any[]) => any ? F : never;

type D = FirstParam<(x: string, y: number) => void>; // string

// Extract return type (same as built-in ReturnType<T>)
type Ret<T> = T extends (...args: any[]) => infer R ? R : never;

type E = Ret<(x: string) => number>;  // number
```

**What happens with union types — distributive behavior:**

Since these are conditional types with a naked type parameter, they distribute over unions (as explained in question 6):

```typescript
// Promise unwrapping distributes:
type X = DeepAwaited<Promise<string> | Promise<number>>;
// Distributes to: DeepAwaited<Promise<string>> | DeepAwaited<Promise<number>>
// Result: string | number

// Parameter extraction distributes:
type Y = Params<((x: string) => void) | ((x: number, y: boolean) => void)>;
// Distributes to: [x: string] | [x: number, y: boolean]

// Return type distributes:
type Z = Ret<(() => string) | (() => number)>;
// Result: string | number
```

Each union member is processed independently, and the results are unioned together. This is usually the desired behavior — if you have a union of functions, you get a union of their parameter tuples or return types. If you wanted to check the union as a whole instead, wrap in a tuple to disable distribution: `[T] extends [Promise<infer U>]` (as covered in question 6).

</details><details>
<summary>19. Show how to compose Pick, Omit, and Record to build a type-safe event handler map — given a set of event names and their payload types, construct a handler registry type where each event maps to a correctly-typed callback, and demonstrate what TypeScript catches at compile time if a handler has the wrong signature</summary>

**Step 1 — Define event payload types:**

```typescript
interface EventPayloads {
  "user:created": { userId: string; email: string };
  "user:deleted": { userId: string; reason: string };
  "order:placed": { orderId: string; total: number; items: string[] };
  "order:cancelled": { orderId: string };
}
```

**Step 2 — Build the handler map type using mapped types:**

```typescript
// Each event maps to a callback that receives its specific payload
type EventHandlerMap = {
  [E in keyof EventPayloads]: (payload: EventPayloads[E]) => void;
};

// Equivalent to:
// {
//   "user:created": (payload: { userId: string; email: string }) => void;
//   "user:deleted": (payload: { userId: string; reason: string }) => void;
//   "order:placed": (payload: { orderId: string; total: number; items: string[] }) => void;
//   "order:cancelled": (payload: { orderId: string }) => void;
// }
```

**Step 3 — Compose with Pick/Omit/Partial for flexibility:**

```typescript
// Only user events
type UserEventHandlers = Pick<EventHandlerMap, "user:created" | "user:deleted">;

// Everything except order events
type NonOrderHandlers = Omit<EventHandlerMap, "order:placed" | "order:cancelled">;

// All handlers optional (for partial registration)
type OptionalHandlerMap = Partial<EventHandlerMap>;
```

**Step 4 — Build a type-safe event emitter:**

```typescript
class TypedEventEmitter {
  private handlers: Partial<EventHandlerMap> = {};

  on<E extends keyof EventPayloads>(
    event: E,
    handler: (payload: EventPayloads[E]) => void
  ): void {
    this.handlers[event] = handler as any; // internal cast, external API is safe
  }

  emit<E extends keyof EventPayloads>(event: E, payload: EventPayloads[E]): void {
    const handler = this.handlers[event];
    if (handler) (handler as any)(payload);
  }
}

const emitter = new TypedEventEmitter();
```

**What TypeScript catches at compile time:**

```typescript
// 1. Wrong payload type for handler
emitter.on("user:created", (payload) => {
  console.log(payload.email);   // OK — TypeScript knows payload has email
  console.log(payload.total);   // Error — 'total' doesn't exist on user:created payload
});

// 2. Wrong payload when emitting
emitter.emit("order:placed", { orderId: "1", total: "free", items: [] });
// Error — 'total' should be number, not string

// 3. Missing required payload fields
emitter.emit("order:placed", { orderId: "1" });
// Error — missing 'total' and 'items'

// 4. Invalid event name
emitter.on("user:updated", () => {});
// Error — "user:updated" is not in keyof EventPayloads

// 5. Handler signature mismatch in a full registry
const handlers: EventHandlerMap = {
  "user:created": (p) => console.log(p.email),
  "user:deleted": (p) => console.log(p.reason),
  "order:placed": (p) => console.log(p.total),
  // Error — missing "order:cancelled" handler
};
```

The key technique: the mapped type `[E in keyof EventPayloads]: (payload: EventPayloads[E]) => void` uses indexed access (as covered in question 14) to look up each event's payload type, creating a 1:1 mapping between event names and their handler signatures.

</details>

<details>
<summary>20. Implement a discriminated union for a state machine (e.g., request states: idle, loading, success, error) with proper exhaustiveness checking — show the never trick in a switch default, demonstrate how this catches missing cases at compile time, and show the pattern for type-safe reducer functions</summary>

**The state union — each state carries exactly the data it needs:**

```typescript
type RequestState<T> =
  | { status: "idle" }
  | { status: "loading"; startedAt: number }
  | { status: "success"; data: T; fetchedAt: number }
  | { status: "error"; error: Error; retryCount: number };
```

**Exhaustiveness checking with the `never` trick:**

```typescript
function assertNever(x: never): never {
  throw new Error(`Unhandled state: ${JSON.stringify(x)}`);
}

function getStatusMessage<T>(state: RequestState<T>): string {
  switch (state.status) {
    case "idle":
      return "Ready to fetch";
    case "loading":
      return `Loading since ${new Date(state.startedAt).toISOString()}`;
    case "success":
      return `Loaded ${JSON.stringify(state.data)}`;
    case "error":
      return `Failed: ${state.error.message} (retries: ${state.retryCount})`;
    default:
      return assertNever(state); // state is `never` here — all cases handled
  }
}
```

**What happens when you add a new state and forget to handle it:**

```typescript
// Add a new state variant:
type RequestState<T> =
  | { status: "idle" }
  | { status: "loading"; startedAt: number }
  | { status: "success"; data: T; fetchedAt: number }
  | { status: "error"; error: Error; retryCount: number }
  | { status: "cancelled"; cancelledBy: string };  // NEW

// Now in the switch, default gets:
// Argument of type '{ status: "cancelled"; cancelledBy: string }'
// is not assignable to parameter of type 'never'.
// Every switch using assertNever immediately shows a compile error.
```

**Type-safe reducer pattern:**

```typescript
// Actions — also a discriminated union
type RequestAction<T> =
  | { type: "FETCH_START" }
  | { type: "FETCH_SUCCESS"; data: T }
  | { type: "FETCH_ERROR"; error: Error }
  | { type: "RESET" };

function requestReducer<T>(
  state: RequestState<T>,
  action: RequestAction<T>
): RequestState<T> {
  switch (action.type) {
    case "FETCH_START":
      return { status: "loading", startedAt: Date.now() };

    case "FETCH_SUCCESS":
      return { status: "success", data: action.data, fetchedAt: Date.now() };

    case "FETCH_ERROR": {
      const retryCount = state.status === "error" ? state.retryCount + 1 : 0;
      return { status: "error", error: action.error, retryCount };
    }

    case "RESET":
      return { status: "idle" };

    default:
      return assertNever(action);
  }
}
```

**Why this pattern is powerful:**

1. **Invalid states are unrepresentable** — you can't have `status: "idle"` with a `data` field. Each variant's shape is enforced.
2. **Narrowing works automatically** — in the `"success"` branch, TypeScript knows `state.data` exists (as covered in question 8).
3. **Adding states/actions forces handling** — the `assertNever` in both the state renderer and the reducer will catch unhandled cases at compile time.
4. **Runtime safety net** — if somehow an unhandled value reaches `assertNever` at runtime (e.g., from an API), it throws with a descriptive error instead of silently proceeding.

</details>

<details>
<summary>21. Write a custom type guard function that validates an unknown API response into a typed shape, then write an assertion function that throws if the input doesn't match — show how each works in control flow, when to use one over the other, and what happens if your type guard lies (returns true for invalid data)</summary>

**Type guard — validates and returns boolean:**

```typescript
interface ApiUser {
  id: string;
  name: string;
  email: string;
  role: "admin" | "user";
}

function isApiUser(data: unknown): data is ApiUser {
  if (typeof data !== "object" || data === null) return false;

  const obj = data as Record<string, unknown>;
  return (
    typeof obj.id === "string" &&
    typeof obj.name === "string" &&
    typeof obj.email === "string" &&
    (obj.role === "admin" || obj.role === "user")
  );
}

// Usage in control flow — narrows inside the if block
async function fetchUser(id: string): Promise<ApiUser | null> {
  const response = await fetch(`/api/users/${id}`);
  const data: unknown = await response.json();

  if (isApiUser(data)) {
    // data is narrowed to ApiUser here
    console.log(data.email.toUpperCase()); // safe
    return data;
  }
  console.warn("Invalid user response:", data);
  return null;
}
```

**Assertion function — throws on failure, narrows the rest of the scope:**

```typescript
function assertApiUser(data: unknown): asserts data is ApiUser {
  if (typeof data !== "object" || data === null) {
    throw new Error(`Expected object, got ${typeof data}`);
  }

  const obj = data as Record<string, unknown>;
  if (typeof obj.id !== "string") throw new Error(`Invalid id: ${obj.id}`);
  if (typeof obj.name !== "string") throw new Error(`Invalid name: ${obj.name}`);
  if (typeof obj.email !== "string") throw new Error(`Invalid email: ${obj.email}`);
  if (obj.role !== "admin" && obj.role !== "user") {
    throw new Error(`Invalid role: ${obj.role}`);
  }
}

// Usage — narrows for the entire remaining scope (no if/else needed)
async function processUser(id: string): Promise<void> {
  const data: unknown = await fetch(`/api/users/${id}`).then((r) => r.json());

  assertApiUser(data); // throws if invalid
  // data is ApiUser for the rest of this function
  console.log(data.name, data.role);
}
```

**When to use each:**

- **Type guard** (`is`): When invalid data is an expected outcome you want to handle gracefully — branching logic, fallbacks, returning null.
- **Assertion function** (`asserts`): When invalid data is a programming error or system failure that should halt execution — precondition checks, middleware validation, test assertions.

**What happens when a type guard lies:**

```typescript
// DANGEROUS — always returns true regardless of actual shape
function isApiUser(data: unknown): data is ApiUser {
  return typeof data === "object" && data !== null; // doesn't check properties!
}

const data: unknown = { id: 123, garbage: true };

if (isApiUser(data)) {
  // TypeScript now believes data is ApiUser
  data.email.toUpperCase(); // No compile error, runtime crash: Cannot read properties of undefined
  data.role === "admin";    // No compile error, runtime: undefined === "admin" → false (silent wrong behavior)
}
```

A lying type guard is worse than `as` because it looks safe — it's wrapped in a runtime check, so code reviewers assume it works. At least `as` is a visible red flag. TypeScript has no way to verify that the guard's implementation matches its annotation (as noted in question 9). In production, prefer Zod schemas (as covered in question 2) over hand-written type guards for complex shapes — Zod generates both the runtime check and the type.

</details>

<details>
<summary>22. Show real-world scenarios where you'd use satisfies vs as const vs an explicit type annotation — demonstrate how satisfies validates a value matches a type without widening it, how as const creates literal types and readonly tuples, and what goes wrong when you use `as` instead (silent type assertion bugs)</summary>

The theory behind these three patterns is covered in question 13. Here are concrete real-world scenarios showing when each one is the right choice.

**Scenario 1 — Route configuration map (`satisfies`):**

```typescript
type RouteConfig = Record<string, { path: string; auth: boolean }>;

// satisfies: validates every entry matches RouteConfig, but preserves the specific keys
const routes = {
  home: { path: "/", auth: false },
  dashboard: { path: "/dashboard", auth: true },
  settings: { path: "/settings", auth: true },
} satisfies RouteConfig;

// TypeScript knows the exact keys — great for autocomplete
routes.dashboard.path; // OK — "dashboard" is a known key
routes.nonexistent;     // Error — "nonexistent" doesn't exist

// With annotation instead:
const routes2: RouteConfig = { home: { path: "/", auth: false } };
routes2.home;        // OK
routes2.nonexistent; // No error! RouteConfig allows any string key
```

`satisfies` catches typos in the values while preserving the specific key set. An annotation widens the keys to `string`.

**Scenario 2 — HTTP status codes (`as const`):**

```typescript
const HTTP_STATUS = {
  OK: 200,
  CREATED: 201,
  BAD_REQUEST: 400,
  NOT_FOUND: 404,
  SERVER_ERROR: 500,
} as const;

// Type: { readonly OK: 200; readonly CREATED: 201; ... }
// Values are literal types (200, not number), properties are readonly

type StatusCode = (typeof HTTP_STATUS)[keyof typeof HTTP_STATUS];
// 200 | 201 | 400 | 404 | 500

// Without as const:
const HTTP_STATUS_WIDE = { OK: 200, NOT_FOUND: 404 };
type WideStatus = (typeof HTTP_STATUS_WIDE)[keyof typeof HTTP_STATUS_WIDE];
// just `number` — useless for type narrowing
```

**Scenario 3 — Tuple route definitions (`as const`):**

```typescript
// as const preserves tuple structure and literal types
const routes = [
  ["GET", "/users", listUsers],
  ["POST", "/users", createUser],
  ["GET", "/users/:id", getUser],
] as const;

// Type: readonly [readonly ["GET", "/users", typeof listUsers], ...]
// Without as const: string[][] — loses all specificity
```

**Scenario 4 — Function parameter contract (explicit annotation):**

```typescript
interface CreateUserInput {
  name: string;
  email: string;
  role: "admin" | "user";
}

// Annotation is the right choice for function parameters and class fields —
// you want the contract, not the narrow type
async function createUser(input: CreateUserInput): Promise<User> {
  // input.role is "admin" | "user" — narrowed enough for the contract
  // Don't need literal type preservation here
}
```

**Scenario 5 — Combining `as const` + `satisfies`:**

```typescript
type Theme = Record<string, { fg: string; bg: string }>;

const themes = {
  light: { fg: "#000", bg: "#fff" },
  dark: { fg: "#fff", bg: "#1a1a1a" },
} as const satisfies Theme;

// Validated against Theme AND preserves literal types and readonly
// themes.light.fg is "#000", not string
// themes.nonexistent is an error (specific keys preserved)
// themes.light.bg = "#000" is an error (readonly)
```

**What goes wrong with `as`:**

```typescript
interface Config {
  host: string;
  port: number;
  ssl: boolean;
}

// Silent missing field — as doesn't validate
const config = { host: "localhost", port: 3000 } as Config;
config.ssl; // typed as boolean, actually undefined at runtime

// Silent wrong type — as doesn't check values
const config2 = { host: "localhost", port: "3000", ssl: "yes" } as Config;
config2.port + 1; // typed as number, actually "30001" (string concat) at runtime

// Extra fields not caught either
const config3 = { host: "localhost", port: 3000, ssl: true, hacker: true } as Config;
// No error — as suppresses excess property checking
```

`as` is a "trust me" override. Use it only when you genuinely know more than the compiler (test mocks, narrowing after external validation). For everything else, `satisfies` or annotations are safer.

</details>

## Practical — Configuration & Tooling

<details>
<summary>23. Configure a production tsconfig.json for a Node.js backend service — explain your choices for target, module, moduleResolution, paths (alias mapping), strict flags, skipLibCheck, and declaration output. Show how the same codebase's tsconfig differs when targeting Node 18 vs Node 20, and what breaks if you get module/moduleResolution wrong</summary>

**Production tsconfig.json for a Node.js 20 backend service:**

```jsonc
{
  "compilerOptions": {
    // Target: ES2023 for Node 20 (supports top-level await, array grouping, etc.)
    "target": "ES2023",

    // Module: NodeNext — tells TS to emit ESM or CJS based on package.json "type"
    "module": "NodeNext",

    // ModuleResolution: NodeNext — matches how Node 20 actually resolves imports
    "moduleResolution": "NodeNext",

    // Strict mode: always. No exceptions for production code.
    "strict": true,

    // Output
    "outDir": "dist",
    "rootDir": "src",
    "declaration": true,        // Generate .d.ts for libraries; skip for pure services
    "declarationMap": true,     // Source maps for .d.ts — enables "go to definition" in consumers
    "sourceMap": true,          // Debugging in production

    // Path aliases — must be mirrored in runtime (e.g., tsc-alias, tsx, or package.json imports)
    "paths": {
      "@src/*": ["./src/*"],
      "@config/*": ["./src/config/*"]
    },
    "baseUrl": ".",

    // Performance
    "skipLibCheck": true,       // Skip type-checking node_modules .d.ts — 2-5x faster builds
    "incremental": true,        // Cache compilation results

    // Safety
    "noUncheckedIndexedAccess": true,  // Record<string, T> returns T | undefined
    "exactOptionalPropertyTypes": true, // undefined !== optional
    "forceConsistentCasingInFileNames": true,
    "isolatedModules": true,     // Required by most bundlers and ts-jest

    // Node.js specific
    "esModuleInterop": true,     // Enables default imports from CJS modules
    "resolveJsonModule": true    // Import .json files
  },
  "include": ["src"],
  "exclude": ["node_modules", "dist"]
}
```

**Node 18 vs Node 20 differences:**

```jsonc
// Node 18
{
  "target": "ES2022",  // Node 18 supports ES2022 (class fields, top-level await, error.cause)
  "lib": ["ES2022"]    // Might need explicit lib if using Array.findLast (ES2023)
}

// Node 20
{
  "target": "ES2023",  // Node 20 supports ES2023 (Array.findLast, Array.toSorted, etc.)
  // No lib needed — ES2023 is included with target
}
```

The main difference is `target`, which controls which syntax features TypeScript emits natively vs downlevels. Setting `ES2023` for Node 18 won't cause syntax errors (Node 18 supports most ES2023 features), but `Array.prototype.toSorted` and similar ES2023 additions may not have type definitions without the right `lib`.

**What breaks with wrong module/moduleResolution:**

```jsonc
// WRONG: module: "CommonJS" + moduleResolution: "NodeNext"
// TS expects CJS emit but NodeNext resolution rules — confusing mismatch

// WRONG: module: "NodeNext" + moduleResolution: "Node10"
// NodeNext module emits ESM, but Node10 resolution doesn't understand
// package.json "exports" field, file extensions in imports, or import conditions.
// Result: TS finds types that don't match what Node actually loads.
```

Specific errors you'll see:

1. `module: "CommonJS"` with a package that only has `"exports": { "import": ... }` — TS can't find the types because it's looking for the `require` condition
2. `moduleResolution: "Node10"` with modern packages — ignores `exports` field, resolves to wrong entry point or fails entirely
3. `module: "ESNext"` without `moduleResolution: "NodeNext"` — TS doesn't enforce `.js` extensions on relative imports, but Node ESM requires them at runtime. Your code compiles but crashes with `ERR_MODULE_NOT_FOUND`

**Path alias gotcha**: `paths` only affects TypeScript's type resolution — the compiled JS still has `@src/...` import paths. You need a runtime resolver: `tsc-alias` (post-compile rewrite), `tsx` (dev), or Node's native `package.json` `"imports"` field (which works with `moduleResolution: "NodeNext"` and requires no extra tooling).

</details><details>
<summary>24. Set up runtime validation at an API boundary using Zod — define a schema that matches a TypeScript type, use it to validate incoming request bodies, show how to infer the TypeScript type from the Zod schema (z.infer), and demonstrate what happens when validation fails vs when you skip validation and trust incoming data</summary>

This builds on the runtime validation concept from question 2 with a complete practical implementation.

**Define the schema — single source of truth for type AND validation:**

```typescript
import { z } from "zod";

// Schema defines structure, constraints, and generates the TypeScript type
const CreateOrderSchema = z.object({
  customerId: z.string().uuid(),
  items: z.array(
    z.object({
      productId: z.string(),
      quantity: z.number().int().positive(),
      unitPrice: z.number().positive(),
    })
  ).min(1, "Order must have at least one item"),
  shippingAddress: z.object({
    street: z.string().min(1),
    city: z.string().min(1),
    postalCode: z.string().regex(/^\d{5}(-\d{4})?$/),
    country: z.string().length(2), // ISO country code
  }),
  notes: z.string().max(500).optional(),
});

// Infer the TypeScript type FROM the schema — never define both manually
type CreateOrderInput = z.infer<typeof CreateOrderSchema>;
// {
//   customerId: string;
//   items: { productId: string; quantity: number; unitPrice: number }[];
//   shippingAddress: { street: string; city: string; postalCode: string; country: string };
//   notes?: string;
// }
```

**Use it at the API boundary (Express example):**

```typescript
import express, { Request, Response } from "express";

const app = express();
app.use(express.json());

app.post("/orders", (req: Request, res: Response) => {
  const result = CreateOrderSchema.safeParse(req.body);

  if (!result.success) {
    // Structured error response with field-level details
    return res.status(400).json({
      error: "Validation failed",
      issues: result.error.issues.map((issue) => ({
        path: issue.path.join("."),
        message: issue.message,
      })),
    });
  }

  // result.data is typed as CreateOrderInput — validated AND typed
  const order = result.data;
  console.log(order.customerId); // string, guaranteed to be a valid UUID
  console.log(order.items[0].quantity); // number, guaranteed to be a positive integer
});
```

**What validation failure looks like:**

```typescript
const badInput = {
  customerId: "not-a-uuid",
  items: [],
  shippingAddress: { street: "", city: "NYC", postalCode: "abc", country: "USA" },
};

const result = CreateOrderSchema.safeParse(badInput);
// result.success === false
// result.error.issues:
// [
//   { code: "invalid_string", path: ["customerId"], message: "Invalid uuid" },
//   { code: "too_small", path: ["items"], message: "Order must have at least one item" },
//   { code: "too_small", path: ["shippingAddress", "street"], message: "String must contain at least 1 character(s)" },
//   { code: "invalid_string", path: ["shippingAddress", "postalCode"], message: "Invalid" },
//   { code: "too_big", path: ["shippingAddress", "country"], message: "String must contain exactly 2 character(s)" }
// ]
```

Every invalid field gets a specific error with its path — the client knows exactly what to fix.

**What happens when you skip validation and trust incoming data:**

```typescript
// DANGEROUS — no validation, just a type assertion
app.post("/orders", async (req: Request, res: Response) => {
  const order = req.body as CreateOrderInput; // TypeScript trusts this blindly

  // If a client sends: { customerId: 123, items: "not-an-array" }
  // TypeScript thinks order.items is an array...
  const total = order.items.reduce((sum, item) => sum + item.quantity * item.unitPrice, 0);
  // Runtime crash: order.items.reduce is not a function

  // Or worse — silent corruption:
  // { customerId: "valid-uuid", items: [{ productId: "p1", quantity: -5, unitPrice: 0 }] }
  // This "passes" type checking but creates an order with negative quantity and zero price
  await saveOrder(order); // corrupt data in your database
});
```

Without runtime validation, you get either crashes (when the shape is wrong) or silent data corruption (when the shape is right but values are invalid). The `as` assertion is the root cause — it tells TypeScript "trust me" when the data hasn't been checked.

**Reusable middleware pattern:**

```typescript
function validateBody<T>(schema: z.ZodSchema<T>) {
  return (req: Request, res: Response, next: express.NextFunction) => {
    const result = schema.safeParse(req.body);
    if (!result.success) {
      return res.status(400).json({ issues: result.error.issues });
    }
    req.body = result.data; // now validated
    next();
  };
}

app.post("/orders", validateBody(CreateOrderSchema), (req, res) => {
  // req.body is validated by this point
});
```

</details>

</details>

<details>
<summary>25. Configure a TypeScript project to correctly resolve both ESM and CJS dependencies — show the tsconfig moduleResolution setting, package.json type field, and the specific errors you get when resolution is wrong (ERR_REQUIRE_ESM, ERR_MODULE_NOT_FOUND), and how to fix each</summary>

This builds on the module resolution theory from question 12 with hands-on configuration.

**The correct configuration for a Node.js ESM project:**

```jsonc
// package.json
{
  "type": "module",  // All .js files are ESM by default
  "engines": { "node": ">=20" },
  "scripts": {
    "build": "tsc",
    "start": "node dist/index.js"
  }
}
```

```jsonc
// tsconfig.json
{
  "compilerOptions": {
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "target": "ES2023",
    "outDir": "dist",
    "rootDir": "src",
    "esModuleInterop": true,  // Needed for default imports from CJS
    "strict": true
  }
}
```

```typescript
// src/index.ts — note the .js extension on relative imports
import { helper } from "./utils.js"; // .js even though the source is .ts
import express from "express";        // CJS package — works via esModuleInterop
import chalk from "chalk";            // ESM-only package — works natively
```

**Error 1: `ERR_REQUIRE_ESM`**

```
Error [ERR_REQUIRE_ESM]: require() of ES Module .../node_modules/chalk/source/index.js not supported
```

**Cause**: Your project is CJS (`"type": "commonjs"` or no type field) and you're importing an ESM-only package (chalk v5, node-fetch v3, etc.). CJS `require()` can't load ESM modules.

**Fixes** (in order of preference):
1. Switch your project to ESM: set `"type": "module"` and `"module": "NodeNext"`
2. Use dynamic `import()` for the ESM dependency:
   ```typescript
   // In a CJS project, dynamic import() works for ESM packages
   const chalk = await import("chalk");
   ```
3. Pin an older CJS version of the dependency (e.g., `chalk@4` instead of `chalk@5`)

**Error 2: `ERR_MODULE_NOT_FOUND`**

```
Error [ERR_MODULE_NOT_FOUND]: Cannot find module '/app/dist/utils' imported from /app/dist/index.js
```

**Cause**: You wrote `import { helper } from "./utils"` (no extension) in your `.ts` file. TypeScript compiled it as-is to `import "./utils"` in the JS output. Node ESM requires explicit file extensions.

**Fix**: Always include `.js` extensions on relative imports when using `moduleResolution: "NodeNext"`:

```typescript
// Wrong — compiles but fails at runtime in ESM
import { helper } from "./utils";

// Correct — .js extension even though the source file is .ts
import { helper } from "./utils.js";
```

This feels wrong but is correct: TypeScript resolves `./utils.js` to `./utils.ts` during compilation, and the emitted JS keeps `./utils.js` which Node can resolve.

**Error 3: `ERR_UNKNOWN_FILE_EXTENSION`**

```
TypeError [ERR_UNKNOWN_FILE_EXTENSION]: Unknown file extension ".ts"
```

**Cause**: You're trying to run `.ts` files directly with `node` in an ESM project. Node doesn't know how to handle `.ts` files.

**Fix**: Use `tsx` for development (`npx tsx src/index.ts`) or compile with `tsc` first and run the `.js` output.

**Error 4: Default import mismatch from CJS packages**

```typescript
import express from "express"; // Works with esModuleInterop
import { Router } from "express"; // Named exports also work

// But some CJS packages with weird exports don't work cleanly:
import pkg from "some-cjs-lib";
const { namedExport } = pkg; // May need to destructure from default
```

**Cause**: CJS modules have `module.exports` (a single export), not `export default` + named exports. `esModuleInterop` creates a compatibility wrapper, but edge cases exist where named imports don't map correctly.

**Practical tip**: When in doubt about a dependency's module format, check its `package.json` for `"type": "module"` and `"exports"` field. If it has `"exports": { "import": ..., "require": ... }`, it's dual-format and will work either way. If it only has `"main"`, it's CJS-only.

</details>

</details>

## Practical — Migration & Refactoring

<details>
<summary>26. Walk through migrating a JavaScript codebase to TypeScript strict mode incrementally — show the tsconfig changes at each stage (allowJs → checkJs → strict flags one by one), how to use // @ts-expect-error for gradual migration, and what the most common strict mode violations are and how to fix each</summary>

This expands on the migration strategy outlined in question 4 with the full step-by-step process.

**Stage 1 — Add TypeScript alongside JavaScript (`allowJs`):**

```jsonc
// tsconfig.json — starting point
{
  "compilerOptions": {
    "allowJs": true,           // Accept .js files in the compilation
    "outDir": "dist",
    "rootDir": "src",
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "esModuleInterop": true
  },
  "include": ["src"]
}
```

At this stage, nothing breaks. JS files are compiled but not type-checked. You can start renaming `.js` to `.ts` file by file, adding types as you go. Start with leaf modules (utilities, types, constants) that have few dependencies.

**Stage 2 — Enable type checking on JS files (`checkJs`):**

```jsonc
{
  "compilerOptions": {
    "allowJs": true,
    "checkJs": true,           // Type-check .js files using JSDoc and inference
    // ...
  }
}
```

This surfaces errors in unconverted JS files. Use `// @ts-ignore` or `// @ts-expect-error` at the top of files that aren't ready:

```javascript
// @ts-nocheck — skip this entire file for now
const config = require("./config");
```

**Stage 3 — Enable strict flags one by one:**

```jsonc
// Round 1: noImplicitAny — the highest-impact flag
{
  "compilerOptions": {
    "noImplicitAny": true
    // ...
  }
}
```

**Common `noImplicitAny` violations and fixes:**

```typescript
// Violation: Parameter 'data' implicitly has an 'any' type
function process(data) { return data.items; }

// Fix: add type annotation
function process(data: { items: string[] }) { return data.items; }

// Temporary fix for complex cases:
// @ts-expect-error — TODO: type this properly (JIRA-123)
function legacyHandler(req, res) { /* ... */ }
```

```jsonc
// Round 2: strictNullChecks — the most errors but most valuable
{
  "compilerOptions": {
    "noImplicitAny": true,
    "strictNullChecks": true
  }
}
```

**Common `strictNullChecks` violations and fixes:**

```typescript
// Violation: Object is possibly 'undefined'
const user = users.find((u) => u.id === id);
console.log(user.name); // Error — find() returns T | undefined

// Fix 1: explicit check
if (!user) throw new Error(`User ${id} not found`);
console.log(user.name); // narrowed

// Fix 2: non-null assertion (only when you're certain)
console.log(user!.name); // suppress — use sparingly

// Violation: Type 'null' is not assignable to type 'string'
let name: string = null; // Error

// Fix: make nullability explicit
let name: string | null = null;
```

```jsonc
// Round 3: strictFunctionTypes
{
  "compilerOptions": {
    "noImplicitAny": true,
    "strictNullChecks": true,
    "strictFunctionTypes": true
  }
}
```

**Common `strictFunctionTypes` violations:**

```typescript
// Violation: callback parameter type is too narrow
type Handler = (event: Event) => void;
const handler: Handler = (event: MouseEvent) => { // Error
  event.clientX; // MouseEvent is narrower than Event
};

// Fix: accept the base type and narrow inside
const handler: Handler = (event: Event) => {
  if (event instanceof MouseEvent) {
    event.clientX;
  }
};
```

```jsonc
// Round 4: replace individual flags with strict: true
{
  "compilerOptions": {
    "strict": true  // Enables all remaining: strictBindCallApply,
                    // strictPropertyInitialization, noImplicitThis,
                    // useUnknownInCatchVariables, alwaysStrict
  }
}
```

**Using `@ts-expect-error` effectively:**

```typescript
// @ts-expect-error — Temporary: migrating to strict null checks (JIRA-456)
const config: Config = getConfig(); // getConfig might return null

// Key difference from @ts-ignore:
// @ts-expect-error fails if there's NO error — so when you fix the underlying
// issue, the suppression itself becomes an error, reminding you to remove it.
// @ts-ignore silently suppresses forever, even after the issue is fixed.
```

**Migration checklist per flag:**

1. Enable the flag
2. Run `tsc --noEmit` to see all errors
3. Fix straightforward errors (usually 70-80%)
4. Add `@ts-expect-error` with JIRA tickets for complex cases
5. PR review, merge
6. Schedule follow-up work for suppressed errors
7. Move to the next flag

</details><details>
<summary>27. Given a TypeScript codebase with several any types that have leaked through — walk through finding them (strict compiler flags, ESLint rules), show how to systematically replace any with unknown and add proper narrowing, and demonstrate how a single any in a utility function propagates to infect callers' types</summary>

This expands on the `any` propagation problem from question 3 with a systematic remediation approach.

**Step 1 — Find all `any` types in the codebase:**

```jsonc
// tsconfig.json — compiler flags that surface any
{
  "compilerOptions": {
    "strict": true,                    // Enables noImplicitAny
    "noImplicitAny": true             // (included in strict, listed for clarity)
  }
}
```

```jsonc
// .eslintrc.json — @typescript-eslint rules that catch explicit any
{
  "rules": {
    "@typescript-eslint/no-explicit-any": "error",       // Flags any written in code
    "@typescript-eslint/no-unsafe-argument": "error",     // Flags passing any to typed params
    "@typescript-eslint/no-unsafe-assignment": "error",   // Flags assigning any to typed vars
    "@typescript-eslint/no-unsafe-call": "error",         // Flags calling any as a function
    "@typescript-eslint/no-unsafe-member-access": "error", // Flags accessing properties on any
    "@typescript-eslint/no-unsafe-return": "error"        // Flags returning any from typed functions
  }
}
```

`noImplicitAny` catches *implicit* any (untyped params). The ESLint rules catch *explicit* any and — critically — any *usage* of values that have the `any` type, even if the `any` came from a dependency.

**Step 2 — Demonstrate how `any` propagates (infection):**

```typescript
// A single any in a utility function...
function parseConfig(raw: string): any {
  return JSON.parse(raw); // JSON.parse returns any
}

// ...infects every caller:
const config = parseConfig('{"port": 3000}');
// config is any — no errors on anything below

const port: number = config.port;           // No error, but could be undefined
const host: string = config.host;           // No error, host doesn't exist
const nested = config.a.b.c.d;             // No error, complete nonsense
const result: boolean[] = config.server;    // No error, could be anything

// It gets worse — any propagates THROUGH typed functions:
function getPort(cfg: { port: number }): number {
  return cfg.port;
}
const p = getPort(config); // No error! any satisfies any parameter type
// p is number (the return type), so the any is now hidden inside a "safe" type
```

The infection path: `JSON.parse` returns `any` -> `parseConfig` returns `any` -> every caller gets `any` -> callers pass `any` into typed functions silently -> downstream code looks type-safe but isn't.

**Step 3 — Systematic replacement with `unknown` + narrowing:**

```typescript
// Before: any return type
function parseConfig(raw: string): any {
  return JSON.parse(raw);
}

// After step 1: unknown return type — forces callers to narrow
function parseConfig(raw: string): unknown {
  return JSON.parse(raw);
}

// After step 2: proper validation (best approach)
import { z } from "zod";

const ConfigSchema = z.object({
  port: z.number(),
  host: z.string(),
  debug: z.boolean().optional(),
});
type Config = z.infer<typeof ConfigSchema>;

function parseConfig(raw: string): Config {
  const parsed: unknown = JSON.parse(raw);
  return ConfigSchema.parse(parsed); // throws on invalid, returns Config
}

// Now callers get real type safety:
const config = parseConfig('{"port": 3000, "host": "localhost"}');
config.port;    // number — guaranteed
config.host;    // string — guaranteed
config.missing; // Error — property doesn't exist on Config
```

**Common sources of `any` leaks and their fixes:**

```typescript
// 1. JSON.parse — returns any
const data = JSON.parse(raw); // any
// Fix: cast to unknown, then validate
const data: unknown = JSON.parse(raw);

// 2. Untyped third-party libraries
import lib from "untyped-lib"; // all imports are any
// Fix: install @types/untyped-lib, or create a local declaration:
// src/types/untyped-lib.d.ts
declare module "untyped-lib" {
  export function doThing(input: string): number;
}

// 3. Error in catch blocks (pre-strict or old code)
catch (error) { // any in old config, unknown with useUnknownInCatchVariables
  console.log(error.message); // No error if any — crash if error isn't Error
}
// Fix: narrow the error
catch (error) {
  if (error instanceof Error) {
    console.log(error.message);
  }
}

// 4. Generic inference fallback
function wrap<T>(value: T) { return { value }; }
wrap(JSON.parse("{}")); // T inferred as any, result.value is any
// Fix: explicitly type or validate the input before passing
const parsed: unknown = JSON.parse("{}");
```

**Prioritization strategy**: Fix `any` leaks in order of blast radius. A utility function called from 50 places is higher priority than a one-off handler. Use the `no-unsafe-*` ESLint rules to find where `any` values are being *consumed* — those are the actual danger points.

</details>

</details>

---

## Experience-Based Questions

These questions test real-world experience. Prepare by mapping them to your own projects and situations.

<details>
<summary>28. Tell me about a time you migrated a codebase to TypeScript or significantly tightened its type safety — what was the scope, what challenges did you face, and what was the impact on developer productivity and bug rates?</summary>

**What the interviewer is looking for:**
- Ability to plan and execute a large-scale technical change incrementally
- Understanding of the practical tradeoffs of TypeScript migration (not just "types are good")
- How you handled team buy-in, backward compatibility, and productivity during the transition
- Measurable outcomes, not just "it felt better"

**Suggested structure (STAR):**

1. **Situation**: What was the codebase? Size, team, language. Why was the migration needed? (Runtime bugs from untyped data, onboarding friction, refactoring fear)
2. **Task**: What was your role? Did you champion it, plan it, execute it? What was the scope — full migration or incremental strictness?
3. **Action**: Walk through the strategy:
   - How you phased the migration (allowJs → file-by-file → strict flags one by one, as in question 26)
   - How you handled the hardest parts (third-party libs without types, complex dynamic patterns, team resistance)
   - Tooling decisions (CI enforcement, ESLint rules, `@ts-expect-error` tracking)
   - How you kept the team productive during migration (not blocking feature work)
4. **Result**: Concrete outcomes — reduction in production bugs, faster PR reviews, onboarding time improvement, developer satisfaction. If you have numbers, use them.

**Key points to hit:**
- Incremental approach is essential — big-bang migrations fail on real codebases
- The hardest part is usually team buy-in and the "messy middle" where half the code is typed
- Strict mode flags should be enabled one at a time, each as its own PR
- Runtime validation at boundaries (Zod) was likely part of the story
- Type coverage metrics helped track progress and maintain momentum

**Example outline to personalize:**
> "At [company], we had a 50K-line Express API in JavaScript. We were averaging 2-3 runtime TypeError bugs per sprint in production, mostly from unexpected null values and wrong payload shapes. I proposed and led the TypeScript migration over 3 months. We started with `allowJs` and `checkJs`, converted files starting from the data layer up, and enabled strict flags incrementally. The hardest challenge was our internal SDK that used dynamic patterns — I wrote custom type definitions for it. By the end, we'd eliminated that category of runtime errors entirely and PR review time dropped because reviewers could trust the types instead of manually tracing data flow."

</details>

</details>

<details>
<summary>29. Describe a time you built a complex generic type or utility type to solve a real problem — what was the use case, how did you design it, and how did it improve the codebase?</summary>

**What the interviewer is looking for:**
- That you've actually needed advanced types to solve a real problem (not just type puzzles)
- Your ability to design types that other developers can understand and use
- Judgment on when complexity is justified vs when simpler alternatives exist
- Understanding of the maintenance cost of complex types

**Suggested structure:**

1. **The problem**: What concrete issue triggered the need? (Repetitive type definitions, type-unsafe API, runtime bugs from wrong usage, boilerplate across similar patterns)
2. **Why simpler approaches weren't enough**: Show you considered alternatives before reaching for generics/conditional types
3. **The design**: Walk through the type, explaining each piece. Focus on the key insight that made it work.
4. **The result**: How it improved the codebase — fewer bugs, less boilerplate, better DX, caught errors at compile time that previously were runtime

**Key points to hit:**
- Start with the developer experience you wanted, then work backward to the type
- Explain the tradeoff: complex types in one place vs simple usage everywhere
- Mention if you added JSDoc or examples so the team could understand it
- If you iterated on the design (started too complex, simplified), that shows maturity

**Example outline to personalize:**
> "We had an event system where 15+ event types each had different payloads, and handlers were typed as `(payload: any) => void`. Every few weeks someone would handle an event with the wrong payload shape. I built a generic event handler map type (similar to the pattern in question 19) — a mapped type that derived handler signatures from a central event payload interface. Adding a new event meant adding one interface entry, and TypeScript would enforce the correct handler signature everywhere. It caught 3 existing bugs on the first compile and made adding new events from a 20-minute task (checking all handlers manually) to a 2-minute task (TypeScript tells you what to fix)."

**Watch out for:**
- Don't pick an example where a simpler solution existed — the interviewer will ask "couldn't you just...?"
- Be ready to whiteboard or explain the type on the spot
- If asked "was it worth the complexity?", have a honest answer about maintenance cost

</details>

</details>

<details>
<summary>30. Tell me about a time a type safety gap caused a production bug — what was the gap (any leak, missing validation, incorrect assertion), how did you discover it, and what did you change to prevent it from recurring?</summary>

**What the interviewer is looking for:**
- Deep understanding of WHERE TypeScript's type safety has limits (system boundaries, `as`, `any` leaks)
- Incident investigation skills — how you traced from symptom to root cause
- Systemic thinking — preventing the class of bug, not just the specific instance
- Honesty about mistakes and what you learned

**Suggested structure:**

1. **The bug**: What happened in production? What was the symptom? (Wrong data in database, crash, silent wrong behavior, API returning unexpected shape)
2. **Discovery**: How did you find it? (Error monitoring, customer report, data audit, failing downstream system)
3. **Root cause — the type safety gap**: Which specific gap? Common ones:
   - `as` assertion on external data (API response, message queue payload) with no runtime validation
   - `any` leaking from a dependency or JSON.parse and flowing through typed code
   - Missing `strictNullChecks` — null value where code assumed non-null
   - Type guard that didn't validate all fields (lying guard, as in question 21)
   - Index signature returning `T` instead of `T | undefined`
4. **The fix**: What did you change? (Added Zod validation at the boundary, replaced `as` with runtime check, enabled strict flags, added ESLint rules)
5. **Prevention**: What systemic change did you make? (CI rule, team guideline, architectural pattern)

**Key points to hit:**
- The root cause is almost always at a system boundary — where TypeScript's compile-time guarantees meet runtime reality
- Show the connection between the TypeScript concept and the real-world failure
- Emphasize the systemic fix, not just the point fix
- If you added monitoring or tests to catch similar gaps, mention it

**Example outline to personalize:**
> "We had a service consuming messages from a queue. The message payload was typed with `as OrderEvent` — no runtime validation. A upstream team added a new optional field and changed the shape of an existing field. Our TypeScript types didn't match the new payload, but since we used `as`, the compiler never flagged it. In production, we were writing corrupted order data to our database for 4 hours before monitoring caught the anomaly. The fix was adding Zod validation at every queue consumer (the boundary pattern from question 2). The systemic change was a team rule: no `as` on external data, enforced by an ESLint rule. We also added payload schema versioning so breaking changes would be caught at the validation layer."

</details>

</details>

<details>
<summary>31. Describe a time you configured TypeScript for a large project or monorepo — what decisions did you make about strict mode, project references, module resolution, and what tradeoffs did you encounter?</summary>

**What the interviewer is looking for:**
- Hands-on experience with TypeScript configuration at scale — not just knowing the flags, but understanding the tradeoffs
- Decision-making process: why you chose specific settings, not just what you chose
- Awareness of monorepo-specific challenges (build order, shared types, IDE performance)
- Pragmatism — strict mode adoption isn't always all-or-nothing

**Suggested structure:**

1. **The context**: What was the project? Monorepo or single repo? How many services/packages? Team size? Existing codebase or greenfield?
2. **Strict mode decisions**: Did you enable full strict from day one, or adopt incrementally (as in question 26)? Why? What was the biggest friction point?
3. **Module resolution & module format**: Which `moduleResolution` and `module` settings? Did you go ESM-first or stick with CJS? What interop issues did you hit (as covered in questions 12 and 25)?
4. **Project references** (if monorepo): Did you use TypeScript project references (`composite`, `references`)? What was the build performance impact? How did you handle shared types across packages?
5. **Path aliases & output structure**: Did you use `paths` for aliases? How did you handle them in compiled output (tsc-alias, bundler, or avoided them)?
6. **Tradeoffs & lessons**: What would you do differently? What surprised you?

**Key points to hit:**
- Show the connection between tsconfig choices and developer experience — slow IDE, confusing errors, broken imports are all config problems
- If monorepo: explain how you balanced per-package configs (each package has its own `tsconfig.json`) vs a shared base config (`tsconfig.base.json` extended by each package)
- Mention tooling decisions that interact with TypeScript: bundler choice (esbuild, swc, tsc), test runner TypeScript support (ts-jest, vitest), linting (`@typescript-eslint`)
- If you hit a specific painful issue (e.g., path aliases not resolving at runtime, `skipLibCheck` hiding real type errors, `declaration` output breaking downstream packages), describe the debugging process

**Example outline to personalize:**
> "I set up TypeScript for a monorepo with 3 Node.js services and 2 shared libraries using npm workspaces. I created a `tsconfig.base.json` at the root with full strict mode, `target: ES2022`, `module: node16`, and `moduleResolution: node16` — targeting Node 20. Each package extended the base and only overrode `outDir`, `rootDir`, and `references`. For the shared libraries, I enabled `composite: true` and `declaration: true` so project references worked — this gave us incremental builds where changing a shared library only recompiled dependent services. The biggest tradeoff was project references requiring `declarationMap` for go-to-definition to work across packages in VS Code, which added build output. We initially used `paths` aliases but dropped them because tsc doesn't rewrite paths in output — we would have needed tsc-alias or a bundler, and for backend services that added unnecessary complexity. The ESM decision was pragmatic: we stayed CJS because two critical dependencies didn't ship proper ESM at the time (covered in question 25). The lesson was that the 'ideal' tsconfig matters less than a consistent, well-documented one — we added comments to every non-obvious setting and a CONTRIBUTING section explaining why we made each choice."

**Watch out for:**
- Don't just list flags — the interviewer wants to hear WHY you chose them and what happened when you got them wrong
- Be ready to discuss what you'd change now vs what you chose then — shows growth
- If asked about project references vs other monorepo build approaches (turborepo, nx), have an opinion on when project references are worth the complexity vs just using a task runner

</details>
