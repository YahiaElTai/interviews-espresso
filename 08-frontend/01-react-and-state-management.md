# React & State Management

> **25 questions** — 12 theory, 9 practical, 4 experience

- Reconciliation algorithm: virtual DOM diffing heuristics, key-based identity, element type comparison, O(n) tradeoffs
- Re-render rules: what triggers renders, subtree re-rendering by default, state/props/context changes, comparison with fine-grained reactivity (Solid, Svelte)
- State management: Context vs Redux vs Zustand vs React Query
- React Query as server state cache (stale-while-revalidate, deduplication, background refetching)
- React Server Components and the server/client boundary
- Advanced patterns: compound components, render props, custom hooks
- Forms: controlled vs uncontrolled components, React Hook Form vs Formik, validation strategies, form state complexity
- Error boundaries: class-based error catching, fallback UI, granularity strategies, limitations (no hooks equivalent, no async errors)
- Hooks rules and mental model: why call order is fixed, closure stale-value traps, rules of hooks enforcement
- useEffect mental model: dependency arrays, cleanup timing, effect vs event distinction, common pitfalls (missing deps, infinite loops)
- Performance profiling: React DevTools Profiler, React.memo, useMemo, useCallback
- Code splitting: React.lazy, Suspense, route-based and component-based splitting

---

## Foundational

<details>
<summary>1. How does React's reconciliation algorithm work — what heuristics does the virtual DOM diffing use to avoid O(n^3) tree comparison, why does React diff by element type and key instead of doing deep structural comparison, and what practical consequences do these heuristics have for how you structure component trees?</summary>

A naive tree diff comparing every node to every other node is O(n^3), which is unusable for real UIs. React reduces this to O(n) with two heuristics:

**Heuristic 1: Different element types produce different trees.** If a `<div>` becomes a `<span>`, React tears down the entire old subtree and builds a new one from scratch — no attempt to reuse children. This is correct >99% of the time because different element types almost always produce structurally different output.

**Heuristic 2: Keys identify which children are the same across renders.** When diffing a list, React matches children by `key` rather than by position. Without keys, inserting an item at the beginning causes every child to unmount and remount. With stable keys, React knows which items moved and only performs the minimal DOM operations.

**Why not deep structural comparison?** Because the cost of comparing arbitrary subtree shapes exceeds the cost of occasionally over-rendering. React optimizes for the common case (small incremental updates) rather than the rare case (radically different trees that happen to share some structure).

**Practical consequences:**

- **Don't change component types conditionally** at the same position in the tree. If you swap between `<UserProfile>` and `<AdminProfile>` at the same spot, React destroys and recreates the entire subtree. If they share state/structure, use a single component with a prop.
- **Always use stable, unique keys on lists.** Using array index as key causes bugs when items are reordered, inserted, or deleted — React reuses the wrong component instances and their state persists incorrectly.
- **Moving a component in the tree unmounts it.** If you conditionally render a component in different branches of an `if/else`, React sees two different tree positions and destroys/recreates it each time. Lift it above the conditional to preserve state.
- **Wrapping a component in a new parent element resets its children.** If you go from `<div><Input /></div>` to `<section><Input /></section>`, the `Input` unmounts because the parent element type changed.

</details>

## Conceptual Depth

<details>
<summary>2. How do React's re-render rules actually work — what triggers a re-render, why does React re-render the entire subtree by default instead of only the changed component, and how does this "coarse-grained" rendering model compare to the fine-grained reactivity of frameworks like Solid or Svelte in terms of performance tradeoffs and developer experience?</summary>

**What triggers a re-render:**

1. **`setState` / `useState` setter** — the component that called it re-renders.
2. **Parent re-renders** — all children re-render by default, regardless of whether their props changed.
3. **Context value changes** — every consumer of that context re-renders.

Note: Changing a ref does NOT trigger a re-render. Changing props alone doesn't trigger a re-render — the parent re-rendering is what triggers the child.

**Why re-render the entire subtree?**

React uses a coarse-grained model because it's simpler and more predictable. When a component re-renders, React doesn't know which children depend on which pieces of state without actually rendering them. Checking every child's props for changes (shallow comparison) has its own cost, and in practice most subtrees are small enough that re-rendering is cheap. React bets on the reconciler being fast enough that skipping the check is cheaper than performing it for every component.

The virtual DOM diff then ensures only actual DOM changes are applied, so the real cost is the JavaScript execution of render functions — not DOM mutations.

**Comparison with fine-grained reactivity (Solid, Svelte):**

| Aspect | React (coarse-grained) | Solid/Svelte (fine-grained) |
|---|---|---|
| Update granularity | Re-runs entire component render function | Updates only the specific DOM nodes that depend on the changed signal/store |
| Runtime cost | More JS execution per update, but batched and optimized by the reconciler | Minimal JS per update — no diffing, direct DOM mutation |
| Mental model | Simple — "when state changes, the component re-renders" | Requires understanding the reactivity graph — which signals connect to which effects |
| Optimization burden | Developer must opt into `React.memo`, `useMemo`, `useCallback` to skip work | Automatic — no memoization APIs needed |
| Component functions | Called on every render (closures recreated) | Called once (setup function), only reactive expressions re-execute |
| Ecosystem maturity | Massive ecosystem, battle-tested at scale | Growing but smaller ecosystems |

**Practical tradeoff:** React's model is easier to reason about and harder to break with subtle reactivity bugs, but you pay for it with optimization boilerplate on hot paths. Fine-grained frameworks deliver better default performance but have a steeper mental model and smaller ecosystems.

</details>

<details>
<summary>3. When choosing between Context, Redux, Zustand, and React Query for state management — what type of state is each best suited for, what are the real performance and complexity tradeoffs (especially Context's re-render problem at scale), and when is reaching for an external state library overkill vs essential?</summary>

The key insight is that "state" is not monolithic — different categories of state have different access patterns, update frequencies, and sharing needs.

**Context API**

- **Best for:** Low-frequency, app-wide values — theme, locale, auth status, feature flags.
- **Tradeoff:** When a context value changes, **every consumer re-renders**, regardless of which part of the value they use. There's no selector mechanism. If you put a frequently-updating object in context (e.g., form state, real-time data), every component consuming that context re-renders on every change.
- **Mitigation:** Split contexts by update frequency. Put theme in one context, user in another. Use `useMemo` on the value object to prevent unnecessary reference changes.
- **When it's enough:** If the data changes rarely (auth, theme) and consumer count is small, Context is the right tool. No library needed.

**Redux (with Redux Toolkit)**

- **Best for:** Complex client state with many inter-dependent updates, state that needs middleware (logging, persistence, undo), teams that benefit from strict unidirectional data flow.
- **Tradeoff:** Significant boilerplate even with RTK. Selectors + `useSelector` give fine-grained subscriptions (only re-render when selected slice changes), which is a real advantage over Context. But it introduces concepts (slices, actions, reducers, thunks) that add learning curve and code surface.
- **When it's essential:** Large apps with complex client-side state machines (e.g., multi-step editors, collaborative features with local state). When you need time-travel debugging or action replay.

**Zustand**

- **Best for:** The same use cases as Redux, but with far less boilerplate. Fine-grained selectors, no provider needed, minimal API surface.
- **Tradeoff:** Less opinionated — no enforced patterns for async, no built-in devtools middleware (though available as plugin). Teams used to Redux's structure may miss the guardrails.
- **When to choose over Redux:** When you need an external store but Redux's ceremony is overkill. Zustand is effectively "Redux minus the boilerplate" for most cases.

```typescript
import { create } from 'zustand';

interface CartStore {
  items: CartItem[];
  addItem: (item: CartItem) => void;
  total: () => number;
}

const useCartStore = create<CartStore>((set, get) => ({
  items: [],
  addItem: (item) => set((state) => ({ items: [...state.items, item] })),
  total: () => get().items.reduce((sum, i) => sum + i.price, 0),
}));

// Fine-grained subscription — only re-renders when items.length changes
const count = useCartStore((state) => state.items.length);
```

**React Query (TanStack Query)**

- **Best for:** Server state — data fetched from APIs that needs caching, background refetching, deduplication, and stale-while-revalidate semantics.
- **Tradeoff:** It's not a general state manager. It manages the cache of server responses. Trying to use it for client-only state is a misuse.
- **When it's essential:** Any app that fetches data from APIs. Replaces manual `useEffect` + `useState` + loading/error state boilerplate.

**Decision framework:**

| State type | Tool |
|---|---|
| Server data (API responses) | React Query |
| Simple app-wide values (theme, auth) | Context |
| Complex client state with many consumers | Zustand or Redux |
| Local component state | `useState` / `useReducer` |

**When external libraries are overkill:** If your app is a handful of pages that fetch some data and have simple UI state, `useState` + Context + React Query covers everything. You don't need Zustand or Redux until you have complex client-side state that multiple unrelated components need to read and write.

</details>

<details>
<summary>4. Why should server state be treated differently from client state — how does React Query's stale-while-revalidate model work, what do its deduplication and background refetching features solve that manual useEffect + useState patterns don't, and when does React Query add unnecessary complexity?</summary>

**Why server state is fundamentally different:**

Client state (UI toggles, form inputs, selected tab) is owned by the client — you're the source of truth. Server state (user list, product catalog, order history) is owned by a remote server — your local copy is always potentially stale. This distinction matters because server state needs caching, synchronization, and invalidation strategies that client state doesn't.

**Stale-while-revalidate model:**

1. On first fetch, React Query caches the response and marks it as **fresh**.
2. After `staleTime` elapses, the data is marked **stale** but still served from cache.
3. When a component mounts or the window refocuses, React Query serves the stale cache **immediately** (instant UI) while triggering a background refetch.
4. When the refetch completes, the UI seamlessly updates with fresh data.
5. After `gcTime` (garbage collection time, default 5 minutes), unused cache entries are deleted.

```typescript
const { data, isLoading, isStale } = useQuery({
  queryKey: ['users'],
  queryFn: () => fetch('/api/users').then(r => r.json()),
  staleTime: 30_000,  // data stays fresh for 30s — no refetch on mount
  gcTime: 300_000,    // keep in cache 5min after last subscriber unmounts
});
```

**What deduplication solves:**

If 5 components mount simultaneously and all call `useQuery({ queryKey: ['users'] })`, React Query fires **one** network request and shares the result. With manual `useEffect`, you'd fire 5 identical requests unless you build your own deduplication layer.

**What background refetching solves:**

Manual patterns fetch once on mount and never update. React Query refetches on window focus, network reconnection, and interval (configurable). This means users see current data without explicit refresh actions.

**What manual useEffect + useState looks like (and why it's worse):**

```typescript
// Manual approach — you must handle every state yourself
const [users, setUsers] = useState<User[]>([]);
const [loading, setLoading] = useState(true);
const [error, setError] = useState<Error | null>(null);

useEffect(() => {
  let cancelled = false;
  setLoading(true);
  fetch('/api/users')
    .then(r => r.json())
    .then(data => { if (!cancelled) { setUsers(data); setLoading(false); } })
    .catch(err => { if (!cancelled) { setError(err); setLoading(false); } });
  return () => { cancelled = true; };
}, []);
// No caching, no deduplication, no background refetch, no stale-while-revalidate
```

Every component doing this creates its own isolated fetch. No shared cache. No staleness model. No automatic retry. You end up rebuilding half of React Query poorly.

**When React Query adds unnecessary complexity:**

- Apps that fetch data once and never refetch (static content, config loaded at startup). A simple `useEffect` is fine.
- When the backend pushes updates via WebSocket and you manage the state yourself.
- Tiny prototypes or demos where caching semantics don't matter.
- When the data is truly local (no server) — React Query isn't a client state manager.

</details>

<details>
<summary>5. What are React Server Components and what problem do they solve -- how does the server/client boundary work in practice, what can and can't you do in a Server Component vs a Client Component, and what constraints does the boundary impose on data flow and component composition?</summary>

**What problem RSCs solve:**

Before RSCs, all React components ship JavaScript to the client, even those that just fetch data and render static HTML. This means larger bundles and unnecessary hydration. RSCs let you keep data-fetching and rendering logic on the server, sending only the rendered output (not the component code) to the client. The client bundle shrinks because server-only code never ships.

**How the server/client boundary works:**

- By default, components in Next.js App Router are **Server Components**.
- Adding `'use client'` at the top of a file marks it (and everything it imports) as a **Client Component**.
- The boundary is at the file level, not the component level. You can't have a Server Component and Client Component in the same file.
- Server Components can import and render Client Components. Client Components **cannot** import Server Components — but they can accept them as `children` or other props (the "donut pattern").

**What you can do in a Server Component:**

- Directly `await` data fetches (database queries, API calls) — no `useEffect` needed.
- Access server-only resources (file system, environment variables, secrets).
- Import large libraries that never ship to the client (markdown parsers, syntax highlighters).
- Render other Server Components or Client Components.

**What you can't do in a Server Component:**

- Use hooks (`useState`, `useEffect`, `useRef`, etc.).
- Attach event handlers (`onClick`, `onChange`, etc.).
- Use browser APIs (`window`, `document`, `localStorage`).
- Use context (`useContext`).

**What you can do in a Client Component:**

- Everything traditional React supports — hooks, event handlers, browser APIs.
- Receive Server Component output as `children` props.

**Data flow constraints:**

- Props passed from Server to Client Components must be **serializable** — plain objects, arrays, strings, numbers, booleans. No functions, classes, Dates, Maps, or Sets.
- This means you can't pass an `onClick` handler from a Server Component to a Client Component inline. The Client Component must define its own handlers.

```tsx
// ServerPage.tsx (Server Component — no 'use client')
import { ClientCounter } from './ClientCounter';

export default async function ServerPage() {
  const data = await db.query('SELECT * FROM items'); // direct DB access

  return (
    <div>
      <h1>Items: {data.length}</h1>
      {/* Pass serializable data to client component */}
      <ClientCounter initialCount={data.length} />
    </div>
  );
}

// ClientCounter.tsx
'use client';
import { useState } from 'react';

export function ClientCounter({ initialCount }: { initialCount: number }) {
  const [count, setCount] = useState(initialCount);
  return <button onClick={() => setCount(c => c + 1)}>Count: {count}</button>;
}
```

**The composition constraint (donut pattern):**

Client Components can't import Server Components, but they can render them via `children`:

```tsx
'use client';
export function ClientWrapper({ children }: { children: React.ReactNode }) {
  const [open, setOpen] = useState(false);
  return open ? <div>{children}</div> : <button onClick={() => setOpen(true)}>Show</button>;
}

// In a Server Component:
<ClientWrapper>
  <ServerRenderedContent />  {/* This works — passed as children */}
</ClientWrapper>
```

</details>

<details>
<summary>6. When do advanced component patterns (compound components, render props, custom hooks) genuinely improve a codebase vs add unnecessary abstraction — what specific problem does each pattern solve, how do they compare in terms of API ergonomics and composability, and what signals tell you a pattern is being overengineered?</summary>

**Compound Components**

- **Problem solved:** Components that have related subparts sharing implicit state, where the consumer needs control over structure/layout.
- **Example:** `<Tabs>`, `<Select>`, `<Accordion>` — the parent manages which item is active, but the consumer controls rendering order and which subcomponents to include.
- **When it helps:** The alternative (a single component with arrays of config objects) becomes rigid. Compound components give JSX-level composition while keeping shared state internal.
- **When it hurts:** Simple components with fixed structure. A `<Card>` with a title and body doesn't need compound components — props or slots are simpler.

**Render Props**

- **Problem solved:** Sharing stateful logic where the consumer controls rendering. Before hooks, this was the primary way to share behavior between components.
- **Example:** A `<MouseTracker render={({ x, y }) => <div>{x},{y}</div>} />` that tracks position and lets the consumer decide what to render.
- **When it helps:** Almost never in modern React. Custom hooks replaced render props for behavior sharing. The remaining valid use case is library APIs where you need to provide render customization (e.g., `react-window`'s `children` as render prop for virtualized rows).
- **When it hurts:** When a custom hook would work. Render props create deeper nesting ("wrapper hell") and are harder to compose.

**Custom Hooks**

- **Problem solved:** Extracting and sharing stateful logic without changing the component hierarchy.
- **Example:** `useDebounce`, `useLocalStorage`, `useAuth`, `useWebSocket`.
- **When it helps:** Whenever two or more components share the same stateful logic, or when a component's logic is complex enough to benefit from separation.
- **When it hurts:** Premature extraction. A `useToggle` hook that wraps a single `useState(false)` adds indirection without meaningful reuse or clarity.

**Comparison:**

| Pattern | Composition | Readability | Reusability | Modern relevance |
|---|---|---|---|---|
| Compound components | Excellent — JSX-level flexibility | Good if familiar | Within the component family | High for UI component libraries |
| Render props | Moderate — nesting gets deep | Poor at scale | Cross-component | Low — replaced by hooks |
| Custom hooks | Excellent — just function calls | Good | Across any component | Primary pattern for logic reuse |

**Signals of overengineering:**

- **One consumer.** If only one component uses the abstraction, inline the logic. Extract when a second consumer appears.
- **The abstraction leaks.** If consumers need to understand implementation details to use the pattern correctly, the abstraction isn't earning its keep.
- **Forced flexibility.** Compound components for a component that will always render the same structure. Render props when children-as-function is never customized.
- **Indirection without payoff.** If you need to read 4 files to understand what a component does, and the pattern doesn't enable meaningful reuse, it's over-abstracted.

</details>

<details>
<summary>7. How are hooks implemented internally in React's Fiber tree — why does React use a linked list to store hook state, why must hooks be called in the same order every render (and what breaks if they aren't), and how do closure stale-value traps occur in useEffect and event handlers?</summary>

**Linked list implementation:**

Each Fiber node (React's internal representation of a component instance) has a `memoizedState` property that points to the first hook. Each hook is a node in a linked list with a `next` pointer to the subsequent hook. On every render, React walks this list in order, pairing each hook call with its stored state.

```
Fiber {
  memoizedState -> Hook1 { state: 'hello', next: -> }
                     Hook2 { state: 42, next: -> }
                       Hook3 { state: [dep1, dep2], next: null }
}
```

**Why a linked list instead of named keys?**

React chose positional identity (call order) over named identity (like Vue's `ref('key')`) to keep the API minimal. You don't need to name each piece of state — you just call `useState` and React matches it to the right slot by position. The tradeoff is the ordering constraint.

**Why hooks must be called in the same order:**

On the first render, React creates the linked list as hooks are called. On subsequent renders, React walks the existing list, expecting each hook call to match its position. If you put a hook inside a conditional:

```typescript
// BROKEN — hook order changes between renders
if (isLoggedIn) {
  const [name, setName] = useState(''); // Hook 1 (sometimes)
}
const [count, setCount] = useState(0);   // Hook 1 or 2 depending on condition
```

When `isLoggedIn` becomes `false`, the `useState(0)` call lands on position 1, but position 1 stores `name`'s state. React returns the wrong state to the wrong hook. This causes silent, hard-to-debug corruption — not a crash.

The `eslint-plugin-react-hooks` rule enforces this at lint time.

**Closure stale-value traps:**

Every render creates a new closure. Hooks like `useEffect` capture the values from the render they were created in:

```typescript
const [count, setCount] = useState(0);

useEffect(() => {
  const id = setInterval(() => {
    console.log(count); // Always logs the value from when the effect ran
  }, 1000);
  return () => clearInterval(id);
}, []); // Empty deps — effect runs once, captures count = 0 forever
```

The callback closes over `count = 0` and never sees updates. This is the "stale closure" problem.

**Fixes:**

1. **Add the dependency:** `[count]` — the effect re-runs on each change (but creates a new interval each time).
2. **Functional update:** `setCount(c => c + 1)` — doesn't need to read `count` from the closure.
3. **Ref pattern:** Store the current value in a ref, read `ref.current` inside the callback. Refs aren't bound by closure timing.

```typescript
const countRef = useRef(count);
countRef.current = count; // Update ref on every render

useEffect(() => {
  const id = setInterval(() => {
    console.log(countRef.current); // Always reads current value
  }, 1000);
  return () => clearInterval(id);
}, []);
```

</details>

<details>
<summary>8. What is the correct mental model for useEffect -- why should effects be thought of as synchronization with external systems rather than lifecycle callbacks, how does cleanup timing actually work (when does React run the cleanup function and why), what is the distinction between effects and event handlers that determines which one you should use, and what causes the common infinite loop pitfall where adding a dependency triggers the effect which updates the dependency?</summary>

**The correct mental model: synchronization, not lifecycle.**

Think of `useEffect` as: "keep this external system in sync with this component's current state." The dependency array tells React "re-sync when these values change." This is fundamentally different from class lifecycle thinking (`componentDidMount`, `componentDidUpdate`, `componentWillUnmount`), which is timeline-based. The sync model is declarative — you describe what should be in sync, not when to do things.

```typescript
// "Keep the document title in sync with the count"
useEffect(() => {
  document.title = `Count: ${count}`;
}, [count]);

// "Keep the WebSocket connection in sync with the roomId"
useEffect(() => {
  const ws = new WebSocket(`/rooms/${roomId}`);
  return () => ws.close();
}, [roomId]);
```

**Cleanup timing:**

1. Component renders with new values.
2. React commits the new DOM.
3. React runs the **cleanup** from the **previous** effect (with the previous render's values).
4. React runs the **new** effect (with the current render's values).

On unmount, React runs the cleanup from the last effect. The cleanup always sees the values from the render that created it (closure behavior).

Why this order? The cleanup tears down the old synchronization before the new one is established. In the WebSocket example, when `roomId` changes from "A" to "B": cleanup closes the connection to room A, then the new effect opens a connection to room B.

**Effects vs event handlers:**

| Aspect | Event handler | useEffect |
|---|---|---|
| Triggered by | User action (click, submit, keypress) | Render + dependency change |
| Purpose | Respond to a specific interaction | Synchronize with external system |
| Example | "When the user clicks Send, post the message" | "When roomId changes, connect to the new room" |

**Rule of thumb:** If the code runs because the user did something, it's an event handler. If the code runs because the component needs to sync with something external based on its current state, it's an effect. Most code should be in event handlers — effects are for side effects that need to stay in sync with reactive values.

**The infinite loop pitfall:**

```typescript
const [data, setData] = useState([]);

useEffect(() => {
  const filtered = items.filter(i => i.active);
  setData(filtered); // This triggers a re-render
}, [items, data]); // data is a dependency, but the effect updates data → infinite loop
```

The loop: effect runs → sets `data` → re-render → `data` changed → effect runs → sets `data` → ...

**Common causes:**

1. **Object/array created inside render as a dependency.** `[items.filter(...)]` creates a new reference every render, so the dependency always "changes."
2. **Effect updates its own dependency.** The effect reads and writes the same state.
3. **Missing `useMemo` on computed values.** A derived value recalculated every render causes a new reference.

**Fixes:** Remove the dependency that the effect updates (often the effect shouldn't exist — derive the value during render instead). Use functional state updates. Memoize objects/arrays that are used as dependencies.

</details>

<details>
<summary>9. Why do React.memo, useMemo, and useCallback exist and what is the correct mental model for when they help vs when they hurt — how does each one prevent unnecessary work, what is the cost of memoization itself (memory, comparison overhead), and what common misuses actually make performance worse?</summary>

**What each one does:**

- **`React.memo(Component)`** — Wraps a component so it skips re-rendering when its props haven't changed (shallow comparison). Prevents the "parent re-renders, child re-renders" default.
- **`useMemo(() => value, [deps])`** — Caches a computed value between renders. Only recomputes when dependencies change.
- **`useCallback(fn, [deps])`** — Caches a function reference between renders. Equivalent to `useMemo(() => fn, [deps])`.

**When they help:**

- `React.memo` helps when a component is expensive to render AND its parent re-renders frequently with unchanged props for that child.
- `useMemo` helps when a computation is genuinely expensive (filtering/sorting large arrays, complex math) or when you need referential stability for an object/array passed as a prop to a `React.memo` child or as a dependency in another hook.
- `useCallback` helps when passing a function to a `React.memo` child — without it, a new function reference on every render defeats the memo.

**The cost of memoization:**

- **Memory:** Every memoized value stays in memory until dependencies change. For `React.memo`, React stores the previous props for comparison.
- **Comparison overhead:** Shallow comparison on every render. For components with many props, this comparison runs on every parent re-render — even when the component would have been cheap to re-render.
- **Code complexity:** More code to read, more dependencies to maintain correctly.

**Common misuses that make performance worse:**

1. **Memoizing cheap components:**
```typescript
// Wasteful — Text is trivially cheap to render
const MemoText = React.memo(({ label }: { label: string }) => <span>{label}</span>);
```
The shallow comparison costs more than just re-rendering the span.

2. **`useCallback` without `React.memo` on the child:**
```typescript
// The callback is stable, but nothing checks if it changed
const handleClick = useCallback(() => doThing(), []);
return <Button onClick={handleClick} />; // Button isn't memoized — re-renders anyway
```

3. **Unstable dependencies defeating memoization:**
```typescript
const MemoChild = React.memo(ChildComponent);

function Parent() {
  // New object reference every render — memo never skips
  return <MemoChild config={{ theme: 'dark' }} />;
}
```
Fix: `const config = useMemo(() => ({ theme: 'dark' }), []);`

4. **Memoizing everything "just in case":**
```typescript
// Overkill — simple addition doesn't need memoization
const total = useMemo(() => a + b, [a, b]);
```

**Mental model:** Don't memoize by default. Profile first, identify the component that's slow, then apply memoization surgically. The React team has said explicitly: most memoization in the wild is unnecessary. React Compiler (in development) aims to automate this entirely.

</details>

<details>
<summary>10. What are the different code splitting strategies in React (route-based vs component-based) — how do React.lazy and Suspense enable lazy loading, what determines where you should place split points for maximum bundle size impact, and when can aggressive splitting actually hurt performance through excessive network requests?</summary>

**How `React.lazy` and `Suspense` work:**

`React.lazy` takes a function that returns a dynamic `import()` and produces a component that loads on first render. `Suspense` provides a fallback UI while the lazy component loads.

```tsx
const Dashboard = React.lazy(() => import('./Dashboard'));

function App() {
  return (
    <Suspense fallback={<LoadingSpinner />}>
      <Dashboard />
    </Suspense>
  );
}
```

When `<Dashboard />` first renders, React shows the fallback, fires the dynamic import, and swaps in the real component when it resolves. The bundler (webpack, Vite) creates a separate chunk for `Dashboard` and everything it imports.

**Route-based splitting:**

Split at the route level — each page is a separate chunk. This is the highest-impact, lowest-risk strategy because users naturally expect a small delay on navigation.

```tsx
const Home = React.lazy(() => import('./pages/Home'));
const Settings = React.lazy(() => import('./pages/Settings'));
const Analytics = React.lazy(() => import('./pages/Analytics'));

function App() {
  return (
    <Suspense fallback={<PageLoader />}>
      <Routes>
        <Route path="/" element={<Home />} />
        <Route path="/settings" element={<Settings />} />
        <Route path="/analytics" element={<Analytics />} />
      </Routes>
    </Suspense>
  );
}
```

**Component-based splitting:**

Split heavy components within a page — modals, charts, rich text editors, admin panels. Useful when a page has features that most users never interact with.

```tsx
const HeavyChart = React.lazy(() => import('./HeavyChart'));

function Dashboard() {
  const [showChart, setShowChart] = useState(false);
  return (
    <div>
      <button onClick={() => setShowChart(true)}>Show Analytics</button>
      {showChart && (
        <Suspense fallback={<ChartSkeleton />}>
          <HeavyChart />
        </Suspense>
      )}
    </div>
  );
}
```

**Where to place split points for maximum impact:**

1. **Route boundaries** — always. This is table stakes.
2. **Heavy dependencies** — components that pull in large libraries (chart libraries, markdown editors, PDF renderers). One lazy import can save 200KB+ from the initial bundle.
3. **Conditional UI** — features behind feature flags, admin-only sections, modals that most users never open.
4. **Below the fold** — content not visible on initial viewport load.

**Analyze your bundle** with `webpack-bundle-analyzer` or `source-map-explorer` to find the largest chunks and determine where splitting has the most impact.

**When aggressive splitting hurts:**

- **Too many small chunks** create excessive HTTP requests. Each chunk has overhead (DNS lookup, TLS handshake on HTTP/1.1, request queuing). With HTTP/2 multiplexing this is less severe, but there's still overhead per request.
- **Waterfall loading** — if chunk A lazy-loads chunk B which lazy-loads chunk C, the user waits for three sequential network round trips. Prefetch or preload to parallelize.
- **Splitting tiny components** — the chunk overhead (webpack runtime, module wrapper) can exceed the bytes saved. Don't split components under ~10-20KB.
- **Flash of loading state** — too many Suspense boundaries create a janky experience where the UI flickers between loading states. Use fewer, strategically placed Suspense boundaries.

</details>

<details>
<summary>11. Why do forms in React become complex as they grow -- what are the tradeoffs between controlled and uncontrolled components, when does each approach break down, and why do libraries like React Hook Form and Formik exist instead of just using native form state? What validation strategy considerations (schema-based vs field-level, sync vs async) matter when choosing between them?</summary>

**Why forms get complex:**

Forms accumulate state concerns as they grow: per-field values, per-field errors, touched/dirty tracking, validation timing (on change, on blur, on submit), async validation (checking username availability), conditional fields, array fields (add/remove items), multi-step wizards, and submission state (loading, error, retry). Each concern multiplies the state management burden.

**Controlled vs uncontrolled:**

| Aspect | Controlled (`value` + `onChange`) | Uncontrolled (`ref` / `defaultValue`) |
|---|---|---|
| State lives in | React state | The DOM |
| Re-renders | On every keystroke | None until read |
| Validation | Can validate on every change | Validate on submit or blur |
| Dynamic behavior | Full control — conditionally disable, transform input | Limited without reading DOM |
| Performance | Re-renders the form on each input | No re-renders — reads values on demand |

**When controlled breaks down:** Large forms (30+ fields) re-render the entire form on every keystroke. If the form component is expensive, this causes noticeable lag. You can mitigate with `React.memo` on subcomponents and isolated state per field, but it's significant work.

**When uncontrolled breaks down:** Complex validation, conditional logic, or dynamic field relationships that need to react to current values. You end up imperatively reading refs and managing state outside React's model.

**Why form libraries exist:**

They solve the state management orchestration that neither approach handles well alone:

- **Formik** — Controlled approach. Manages values, errors, touched, and submission state in a single form context. Re-renders on every change (the form or at least subscribed fields).
- **React Hook Form** — Uncontrolled approach using refs internally. Registers inputs via `register()` which attaches refs. Only re-renders when form state (errors, isSubmitting) changes, not on every keystroke. Significantly better performance for large forms.

```typescript
// React Hook Form — minimal re-renders
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';

const schema = z.object({
  email: z.string().email(),
  age: z.number().min(18),
});

type FormData = z.infer<typeof schema>;

function SignupForm() {
  const { register, handleSubmit, formState: { errors } } = useForm<FormData>({
    resolver: zodResolver(schema),
  });

  return (
    <form onSubmit={handleSubmit((data) => console.log(data))}>
      <input {...register('email')} />
      {errors.email && <span>{errors.email.message}</span>}

      <input {...register('age', { valueAsNumber: true })} type="number" />
      {errors.age && <span>{errors.age.message}</span>}

      <button type="submit">Submit</button>
    </form>
  );
}
```

**Validation strategy considerations:**

- **Schema-based (Zod, Yup):** Define validation once, derive TypeScript types from it, validate the entire form as a unit. Better for forms where fields have inter-dependent validation (password/confirm password, date range start < end).
- **Field-level:** Validate each field independently using `validate` functions on individual inputs. Simpler for small forms or when fields are truly independent.
- **Sync vs async:** Most validation is sync (format, required, length). Async validation (check if username is taken) should be debounced and run on blur, not on every keystroke, to avoid flooding the server.

**When to choose which:**

- Small forms (< 5 fields): `useState` + controlled inputs. No library needed.
- Medium forms with validation: React Hook Form + Zod. Best performance-to-complexity ratio.
- Large, complex forms with many dynamic fields: React Hook Form. The uncontrolled approach matters at this scale.
- Legacy codebase already using Formik: Keep Formik unless performance is a problem.

</details>

<details>
<summary>12. Why are error boundaries still class components in React and what are their actual limitations -- what errors do they catch vs miss (async errors, event handlers, SSR), how do you decide on error boundary granularity (page-level vs component-level), and what patterns work around the lack of a hooks-based error boundary API?</summary>

**Why still class components:**

Error boundaries rely on the `componentDidCatch` and `static getDerivedStateFromError` lifecycle methods. The React team hasn't shipped a hooks equivalent (`useErrorBoundary` or similar) because the semantics of "catching errors thrown during render" don't map cleanly to the hooks model — hooks run during render, so a hook catching its own render errors creates circular questions. The community library `react-error-boundary` provides a hooks-friendly wrapper (`useErrorBoundary`) that internally still uses a class component.

**What error boundaries catch:**

- Errors thrown during **rendering** (in the component tree below the boundary).
- Errors thrown in **lifecycle methods** and **constructors**.
- Errors in `useEffect` cleanup (when the cleanup function throws during commit phase — though this is edge-case).

**What they do NOT catch:**

- **Event handlers** — errors in `onClick`, `onChange`, etc. don't occur during render. Use try/catch inside the handler.
- **Async code** — `setTimeout`, `fetch().then()`, `async/await` in effects. Errors are thrown asynchronously, outside React's render cycle.
- **Server-side rendering** — error boundaries don't work during SSR. The server needs its own error handling (try/catch around `renderToString`).
- **Errors in the error boundary itself** — if the boundary's render throws, it bubbles up to the nearest parent boundary.

**Granularity strategy:**

Think of error boundaries like try/catch — too broad loses precision, too narrow adds clutter.

- **Page/route level:** Always have one. Catches unexpected crashes and shows a "something went wrong" page. This is your safety net.
- **Feature/widget level:** Wrap independent features (chat widget, notification panel, analytics dashboard) so one crashing doesn't take down the whole page.
- **Component level:** Only for components with known failure modes (third-party embeds, dynamic content rendering, image galleries with external URLs).

```tsx
// Practical granularity
<AppErrorBoundary fallback={<FullPageError />}>
  <Header />
  <main>
    <ErrorBoundary fallback={<WidgetError name="Chat" />}>
      <ChatWidget />  {/* If chat crashes, rest of page works */}
    </ErrorBoundary>
    <ErrorBoundary fallback={<WidgetError name="Feed" />}>
      <ActivityFeed />
    </ErrorBoundary>
  </main>
</AppErrorBoundary>
```

**Working around the lack of hooks API:**

Use `react-error-boundary` — it provides both the boundary component and a `useErrorBoundary` hook for imperatively triggering error state from event handlers and async code:

```tsx
import { ErrorBoundary, useErrorBoundary } from 'react-error-boundary';

function SaveButton() {
  const { showBoundary } = useErrorBoundary();

  const handleSave = async () => {
    try {
      await saveData();
    } catch (error) {
      showBoundary(error); // Triggers the nearest error boundary
    }
  };

  return <button onClick={handleSave}>Save</button>;
}

// Wrap with ErrorBoundary
<ErrorBoundary fallback={<p>Save failed</p>}>
  <SaveButton />
</ErrorBoundary>
```

This pattern bridges the gap — async and event handler errors can now be caught by error boundaries via the imperative `showBoundary` call.

</details>

## Practical — Performance Profiling & Optimization

<details>
<summary>13. Walk through a performance profiling session using React DevTools Profiler — how do you identify which components are re-rendering unnecessarily, how do you read the flamegraph and ranked chart, what do the commit durations and render reasons tell you, and how do you translate profiler findings into actionable optimizations?</summary>

**Setup:**

1. Install React DevTools browser extension.
2. Open DevTools → Profiler tab.
3. Enable "Record why each component rendered" in Profiler settings (gear icon).
4. Click Record, perform the slow interaction, click Stop.

**Reading the flamegraph:**

The flamegraph shows each commit (a batch of state changes React applied to the DOM). Each bar is a component:

- **Width** represents the time spent rendering that component and its subtree.
- **Color** indicates render duration — gray means the component didn't render (was skipped), yellow/orange means it rendered, and darker colors mean longer render times.
- **Clicking a component** shows its props, state, and hooks — and critically, **why it rendered** (props changed, state changed, parent rendered, hooks changed).

**Reading the ranked chart:**

Switches from tree view to a flat list sorted by render duration (longest first). This immediately answers "which component is the most expensive?" — you don't need to dig through the tree.

**Commit durations:**

The bar chart at the top shows every commit's total duration. Tall bars indicate expensive commits. Click a tall bar to see what rendered during that commit. If most commits are fast but one is slow, that commit is your investigation target.

**Translating findings into optimizations:**

**Finding: Component re-renders because "parent rendered" with unchanged props.**
→ Wrap with `React.memo`. Ensure props are referentially stable (use `useMemo`/`useCallback` in the parent).

**Finding: Expensive component renders on every keystroke in a sibling.**
→ Lift the fast-changing state down into the component that owns it. Or split the expensive component behind `React.memo`.

**Finding: A component re-renders because a context value changed.**
→ Split the context. Put the frequently-changing value in a separate context from the stable values.

**Finding: A component renders fast but renders 500 times in one interaction.**
→ Investigate the state update pattern. Likely a state update in a loop or an effect that triggers cascading re-renders.

**Finding: A hook value changes every render (new object/array reference).**
→ Memoize the computed value with `useMemo`.

**Example workflow:**

```
1. Profile: typing in search input causes 200ms commit
2. Flamegraph: ProductList (120ms) re-renders on every keystroke
3. Render reason: "parent rendered" — props unchanged
4. Fix: React.memo(ProductList) + useMemo on the products array prop
5. Re-profile: commit drops to 15ms
```

**Key principle:** Profile first, optimize second. Never apply `React.memo` or `useMemo` based on intuition — the profiler often reveals the bottleneck is somewhere unexpected.

</details>

<details>
<summary>14. Given a component tree where a parent re-renders frequently and causes expensive child re-renders — show the before/after code applying React.memo, useMemo, and useCallback to fix it, explain why each optimization is necessary in this specific case, and demonstrate a scenario where applying React.memo without stabilizing props actually does nothing?</summary>

**Before — no optimization:**

```tsx
interface Product {
  id: string;
  name: string;
  price: number;
}

function Dashboard() {
  const [searchTerm, setSearchTerm] = useState('');
  const [products] = useState<Product[]>(generateLargeProductList());

  // New array reference on every render
  const filtered = products.filter(p =>
    p.name.toLowerCase().includes(searchTerm.toLowerCase())
  );

  // New function reference on every render
  const handleProductClick = (id: string) => {
    console.log('Selected:', id);
  };

  return (
    <div>
      <input
        value={searchTerm}
        onChange={(e) => setSearchTerm(e.target.value)}
        placeholder="Search..."
      />
      <ProductList products={filtered} onProductClick={handleProductClick} />
    </div>
  );
}

// Expensive component — renders 1000+ product cards
function ProductList({ products, onProductClick }: {
  products: Product[];
  onProductClick: (id: string) => void;
}) {
  return (
    <div>
      {products.map(p => (
        <ProductCard key={p.id} product={p} onClick={() => onProductClick(p.id)} />
      ))}
    </div>
  );
}

function ProductCard({ product, onClick }: { product: Product; onClick: () => void }) {
  // Simulate expensive render
  expensiveCalculation(product);
  return <div onClick={onClick}>{product.name} - ${product.price}</div>;
}
```

Every keystroke in the search input causes `Dashboard` to re-render → `ProductList` re-renders → all `ProductCard`s re-render. Even cards whose data hasn't changed re-render because of React's default subtree behavior.

**After — optimized:**

```tsx
function Dashboard() {
  const [searchTerm, setSearchTerm] = useState('');
  const [products] = useState<Product[]>(generateLargeProductList());

  // 1. useMemo: only recompute when searchTerm or products change
  const filtered = useMemo(
    () => products.filter(p =>
      p.name.toLowerCase().includes(searchTerm.toLowerCase())
    ),
    [products, searchTerm]
  );

  // 2. useCallback: stable function reference across renders
  const handleProductClick = useCallback((id: string) => {
    console.log('Selected:', id);
  }, []);

  return (
    <div>
      <input
        value={searchTerm}
        onChange={(e) => setSearchTerm(e.target.value)}
        placeholder="Search..."
      />
      <MemoProductList products={filtered} onProductClick={handleProductClick} />
    </div>
  );
}

// 3. React.memo: skip re-render when props are shallowly equal
const MemoProductList = React.memo(function ProductList({ products, onProductClick }: {
  products: Product[];
  onProductClick: (id: string) => void;
}) {
  return (
    <div>
      {products.map(p => (
        <MemoProductCard key={p.id} product={p} onClick={onProductClick} />
      ))}
    </div>
  );
});

const MemoProductCard = React.memo(function ProductCard({ product, onClick }: {
  product: Product;
  onClick: (id: string) => void;
}) {
  expensiveCalculation(product);
  return <div onClick={() => onClick(product.id)}>{product.name} - ${product.price}</div>;
});
```

**Why each optimization is necessary:**

- **`useMemo` on `filtered`:** Without it, `filter()` creates a new array reference every render, even if the result is identical. `React.memo` on `ProductList` would see a "new" `products` prop every time and re-render anyway.
- **`useCallback` on `handleProductClick`:** Without it, a new function reference is created every render. Same problem — `React.memo` sees a "new" `onProductClick` prop.
- **`React.memo` on `ProductList` and `ProductCard`:** Skips re-rendering when props are shallowly equal. This is the optimization that actually prevents the expensive work — `useMemo` and `useCallback` just make it possible by stabilizing the props.

**Scenario where React.memo does nothing:**

```tsx
const MemoChild = React.memo(function Child({ config }: { config: { theme: string } }) {
  return <div className={config.theme}>Content</div>;
});

function Parent() {
  const [count, setCount] = useState(0);

  return (
    <div>
      <button onClick={() => setCount(c => c + 1)}>Click: {count}</button>
      {/* New object literal on every render — React.memo comparison always fails */}
      <MemoChild config={{ theme: 'dark' }} />
    </div>
  );
}
```

Every click re-renders `MemoChild` despite the memo wrapper. The `config` object is created inline, so it's a new reference every render. Shallow comparison (`{ theme: 'dark' } !== { theme: 'dark' }`) returns false. The memo is useless without `useMemo(() => ({ theme: 'dark' }), [])` on the config.

</details>

## Practical — State Management & Component Patterns

<details>
<summary>15. Build a compound component (e.g., a Tabs component with Tabs, TabList, Tab, TabPanel) in TypeScript — show the full implementation using Context to share state between the compound parts, explain the component API design decisions, and demonstrate how this pattern gives consumers flexible composition while keeping internal state encapsulated?</summary>

```tsx
import { createContext, useContext, useState, ReactNode } from 'react';

// --- Internal Context ---
interface TabsContextValue {
  activeIndex: number;
  setActiveIndex: (index: number) => void;
}

const TabsContext = createContext<TabsContextValue | null>(null);

function useTabsContext() {
  const context = useContext(TabsContext);
  if (!context) {
    throw new Error('Tabs compound components must be used within <Tabs>');
  }
  return context;
}

// --- Compound Components ---
interface TabsProps {
  defaultIndex?: number;
  children: ReactNode;
}

function Tabs({ defaultIndex = 0, children }: TabsProps) {
  const [activeIndex, setActiveIndex] = useState(defaultIndex);

  return (
    <TabsContext.Provider value={{ activeIndex, setActiveIndex }}>
      <div role="tablist-container">{children}</div>
    </TabsContext.Provider>
  );
}

function TabList({ children }: { children: ReactNode }) {
  return <div role="tablist" style={{ display: 'flex', gap: '4px' }}>{children}</div>;
}

interface TabProps {
  index: number;
  children: ReactNode;
}

function Tab({ index, children }: TabProps) {
  const { activeIndex, setActiveIndex } = useTabsContext();
  const isActive = activeIndex === index;

  return (
    <button
      role="tab"
      aria-selected={isActive}
      onClick={() => setActiveIndex(index)}
      style={{ fontWeight: isActive ? 'bold' : 'normal' }}
    >
      {children}
    </button>
  );
}

interface TabPanelProps {
  index: number;
  children: ReactNode;
}

function TabPanel({ index, children }: TabPanelProps) {
  const { activeIndex } = useTabsContext();
  if (activeIndex !== index) return null;

  return <div role="tabpanel">{children}</div>;
}

// Attach subcomponents to Tabs for clean imports
Tabs.List = TabList;
Tabs.Tab = Tab;
Tabs.Panel = TabPanel;

export { Tabs };
```

**Consumer usage — flexible composition:**

```tsx
function SettingsPage() {
  return (
    <Tabs defaultIndex={0}>
      <Tabs.List>
        <Tabs.Tab index={0}>Profile</Tabs.Tab>
        <Tabs.Tab index={1}>Security</Tabs.Tab>
        <Tabs.Tab index={2}>Billing</Tabs.Tab>
      </Tabs.List>

      {/* Consumer controls layout — can add whatever they want between list and panels */}
      <div className="divider" />

      <Tabs.Panel index={0}>
        <ProfileSettings />
      </Tabs.Panel>
      <Tabs.Panel index={1}>
        <SecuritySettings />
      </Tabs.Panel>
      <Tabs.Panel index={2}>
        <BillingSettings />
      </Tabs.Panel>
    </Tabs>
  );
}
```

**API design decisions:**

1. **Context for implicit state sharing.** The consumer never passes `activeIndex` or `onChange` to each `Tab` and `TabPanel` — the `Tabs` parent owns state and distributes it via context. This keeps the consumer API clean.

2. **Explicit `index` prop instead of automatic indexing.** Some implementations use `React.Children.map` to inject indices automatically. Explicit indices are more predictable — reordering JSX children doesn't silently break tab/panel associations, and consumers can skip indices or conditionally render tabs.

3. **Subcomponent attachment (`Tabs.List`, `Tabs.Tab`).** Single import, discoverable API. The consumer imports `Tabs` and dot-notation reveals all parts. Alternative: named exports (`import { Tabs, Tab, TabList, TabPanel }`) — equally valid, but dot-notation signals "these belong together."

4. **Context guard (`useTabsContext` throws).** Using `<Tab>` outside `<Tabs>` throws a clear error instead of silently rendering nothing. Fail fast with a helpful message.

5. **Composition freedom.** The consumer can put arbitrary elements between `TabList` and `TabPanel`s, wrap panels in layout components, or conditionally render tabs. A config-based API (`<Tabs items={[...]} />`) wouldn't allow this flexibility.

</details>

<details>
<summary>16. Set up React Query for a feature that lists and creates items -- show the TypeScript code for the query hook, mutation hook, and cache invalidation strategy, and explain how staleTime and gcTime affect caching behavior and when you'd tune each one.</summary>

```tsx
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';

// --- Types ---
interface Todo {
  id: string;
  title: string;
  completed: boolean;
}

interface CreateTodoInput {
  title: string;
}

// --- API layer ---
const todosApi = {
  list: async (): Promise<Todo[]> => {
    const res = await fetch('/api/todos');
    if (!res.ok) throw new Error('Failed to fetch todos');
    return res.json();
  },
  create: async (input: CreateTodoInput): Promise<Todo> => {
    const res = await fetch('/api/todos', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(input),
    });
    if (!res.ok) throw new Error('Failed to create todo');
    return res.json();
  },
};

// --- Query hook ---
function useTodos() {
  return useQuery({
    queryKey: ['todos'],
    queryFn: todosApi.list,
    staleTime: 30_000, // Data stays fresh for 30s — no refetch on component mount
  });
}

// --- Mutation hook ---
function useCreateTodo() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: todosApi.create,
    onSuccess: () => {
      // Invalidate the todos query — triggers a refetch
      queryClient.invalidateQueries({ queryKey: ['todos'] });
    },
  });
}

// --- Usage in a component ---
function TodoPage() {
  const { data: todos, isLoading, error } = useTodos();
  const createTodo = useCreateTodo();
  const [title, setTitle] = useState('');

  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();
    createTodo.mutate({ title }, {
      onSuccess: () => setTitle(''),
    });
  };

  if (isLoading) return <div>Loading...</div>;
  if (error) return <div>Error: {error.message}</div>;

  return (
    <div>
      <form onSubmit={handleSubmit}>
        <input value={title} onChange={e => setTitle(e.target.value)} />
        <button type="submit" disabled={createTodo.isPending}>
          {createTodo.isPending ? 'Adding...' : 'Add Todo'}
        </button>
      </form>
      <ul>
        {todos?.map(todo => (
          <li key={todo.id}>{todo.title}</li>
        ))}
      </ul>
    </div>
  );
}
```

**Cache invalidation strategy:**

`onSuccess` calls `invalidateQueries({ queryKey: ['todos'] })`. This marks the `['todos']` cache entry as stale and immediately triggers a background refetch if any component is subscribed. The list updates with the server's response, which includes the newly created item with its server-generated ID.

Alternative: Instead of invalidating, you could manually update the cache with `setQueryData` to add the new item without a refetch. This is faster (no network round trip) but means the list might drift from the server if another user also added items. Invalidation is safer for correctness.

**`staleTime` and `gcTime` explained:**

- **`staleTime`** (default: 0) — How long data stays "fresh" after fetching. While fresh, React Query serves from cache and **does not refetch** on mount, window focus, or network reconnect. After staleTime expires, data is marked stale, and the next trigger causes a background refetch (while still showing the stale cache immediately).

- **`gcTime`** (default: 5 minutes) — How long **inactive** cache entries are kept in memory. When the last component using a query unmounts, the cache entry becomes inactive. After gcTime, it's garbage collected. If a component remounts before gcTime, it gets the cached data instantly (stale or not).

**When to tune each:**

| Scenario | staleTime | gcTime |
|---|---|---|
| Rarely-changing data (config, feature flags) | `Infinity` or `600_000` (10min) | Long (match staleTime) |
| Frequently-changing data (notifications, chat) | `0` (always refetch on mount) | Default or short |
| User-specific dashboard | `30_000` (30s) | Default |
| Data the user just navigated away from and will return to | Default staleTime | Longer gcTime (keep cache warm) |
| Static reference data (countries list, currencies) | `Infinity` | `Infinity` |

Setting `staleTime: 0` (default) means every mount triggers a background refetch, which is correct for data that changes frequently but wasteful for stable data. Setting it higher reduces unnecessary network requests at the cost of showing slightly older data.

</details>

<details>
<summary>17. Implement optimistic updates with React Query for a mutation that could fail -- show the TypeScript code using onMutate, onError, and onSettled callbacks, demonstrate what happens visually when the server rejects the update, and explain why the rollback pattern requires saving a snapshot of the previous cache state.</summary>

```tsx
import { useMutation, useQueryClient } from '@tanstack/react-query';

interface Todo {
  id: string;
  title: string;
  completed: boolean;
}

function useToggleTodo() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: async (todoId: string) => {
      const res = await fetch(`/api/todos/${todoId}/toggle`, { method: 'PATCH' });
      if (!res.ok) throw new Error('Server rejected the update');
      return res.json() as Promise<Todo>;
    },

    onMutate: async (todoId: string) => {
      // 1. Cancel in-flight refetches so they don't overwrite our optimistic update
      await queryClient.cancelQueries({ queryKey: ['todos'] });

      // 2. Snapshot the current cache for rollback
      const previousTodos = queryClient.getQueryData<Todo[]>(['todos']);

      // 3. Optimistically update the cache
      queryClient.setQueryData<Todo[]>(['todos'], (old) =>
        old?.map(todo =>
          todo.id === todoId ? { ...todo, completed: !todo.completed } : todo
        )
      );

      // 4. Return the snapshot — React Query passes this to onError
      return { previousTodos };
    },

    onError: (_error, _todoId, context) => {
      // Rollback to the snapshot
      if (context?.previousTodos) {
        queryClient.setQueryData(['todos'], context.previousTodos);
      }
    },

    onSettled: () => {
      // Always refetch after mutation settles to ensure server/client consistency
      queryClient.invalidateQueries({ queryKey: ['todos'] });
    },
  });
}

// Usage
function TodoItem({ todo }: { todo: Todo }) {
  const toggleTodo = useToggleTodo();

  return (
    <label style={{ opacity: toggleTodo.isPending ? 0.6 : 1 }}>
      <input
        type="checkbox"
        checked={todo.completed}
        onChange={() => toggleTodo.mutate(todo.id)}
      />
      {todo.title}
    </label>
  );
}
```

**What happens visually when the server rejects:**

1. User clicks checkbox → `onMutate` fires → checkbox instantly appears toggled (optimistic).
2. Network request goes to server.
3. Server returns 400/500 → `onError` fires → cache reverts to snapshot → checkbox snaps back to original state.
4. `onSettled` fires → background refetch ensures cache matches server truth.

The user sees the checkbox toggle, then untoggle ~200-500ms later. You can enhance UX by showing a toast ("Failed to update — reverted") or a brief animation indicating the rollback.

**Why the snapshot is necessary:**

The rollback needs to know what the cache looked like **before** the optimistic update. Without the snapshot, you can't undo the change. Consider:

- You can't just "re-toggle" because other updates might have happened between `onMutate` and `onError`.
- Multiple concurrent mutations might be in flight. Mutation A optimistically updates, then mutation B optimistically updates on top of A's change. If A fails, you need A's specific previous state — not just "undo the last thing."
- The `onError` callback only knows the mutation errored — it doesn't know what the original cache state was unless you explicitly saved it.

`onMutate` returns the context object, which React Query stores and passes as the third argument to `onError`. This is the rollback mechanism.

**Why `cancelQueries` matters:**

Without canceling in-flight refetches, this race condition can occur:

1. `onMutate` optimistically updates cache.
2. An already-in-flight refetch completes with the old server data.
3. The refetch overwrites the optimistic update.
4. The user sees the checkbox flicker back before the mutation even completes.

`cancelQueries` prevents this by aborting any pending refetches for that query key.

</details>

<details>
<summary>18. Build a custom hook that manages a WebSocket connection — show the TypeScript implementation that handles connection lifecycle, reconnection logic, cleanup on unmount, and avoids stale closure bugs when accessing current state inside event handlers. Explain the specific closure trap and how the ref pattern or useEffectEvent solves it?</summary>

```tsx
import { useEffect, useRef, useState, useCallback } from 'react';

interface UseWebSocketOptions {
  url: string;
  onMessage: (data: unknown) => void;
  reconnectAttempts?: number;
  reconnectInterval?: number;
}

interface UseWebSocketReturn {
  send: (data: string) => void;
  status: 'connecting' | 'connected' | 'disconnected' | 'reconnecting';
  lastError: Event | null;
}

function useWebSocket({
  url,
  onMessage,
  reconnectAttempts = 5,
  reconnectInterval = 3000,
}: UseWebSocketOptions): UseWebSocketReturn {
  const [status, setStatus] = useState<UseWebSocketReturn['status']>('connecting');
  const [lastError, setLastError] = useState<Event | null>(null);

  const wsRef = useRef<WebSocket | null>(null);
  const retriesRef = useRef(0);
  const reconnectTimerRef = useRef<ReturnType<typeof setTimeout>>();

  // Ref to always have the latest onMessage callback without re-running the effect
  const onMessageRef = useRef(onMessage);
  onMessageRef.current = onMessage;

  const connect = useCallback(() => {
    // Clean up existing connection
    if (wsRef.current) {
      wsRef.current.close();
    }

    const ws = new WebSocket(url);
    wsRef.current = ws;

    ws.onopen = () => {
      setStatus('connected');
      retriesRef.current = 0; // Reset retry count on successful connection
    };

    ws.onmessage = (event) => {
      // Read from ref — always gets the latest callback, no stale closure
      onMessageRef.current(JSON.parse(event.data));
    };

    ws.onerror = (error) => {
      setLastError(error);
    };

    ws.onclose = () => {
      setStatus('disconnected');

      // Reconnect logic
      if (retriesRef.current < reconnectAttempts) {
        setStatus('reconnecting');
        reconnectTimerRef.current = setTimeout(() => {
          retriesRef.current += 1;
          connect();
        }, reconnectInterval);
      }
    };
  }, [url, reconnectAttempts, reconnectInterval]);

  useEffect(() => {
    connect();

    return () => {
      // Cleanup: close WebSocket and cancel pending reconnect
      clearTimeout(reconnectTimerRef.current);
      if (wsRef.current) {
        wsRef.current.close();
      }
    };
  }, [connect]);

  const send = useCallback((data: string) => {
    if (wsRef.current?.readyState === WebSocket.OPEN) {
      wsRef.current.send(data);
    }
  }, []);

  return { send, status, lastError };
}
```

**Usage:**

```tsx
function ChatRoom({ roomId }: { roomId: string }) {
  const [messages, setMessages] = useState<Message[]>([]);

  const { send, status } = useWebSocket({
    url: `wss://api.example.com/rooms/${roomId}`,
    onMessage: (data) => {
      // This callback changes every render (new closure over messages)
      // but the ref pattern ensures the WebSocket always calls the latest version
      setMessages(prev => [...prev, data as Message]);
    },
  });

  return (
    <div>
      <span>Status: {status}</span>
      {messages.map(m => <div key={m.id}>{m.text}</div>)}
      <button onClick={() => send(JSON.stringify({ text: 'Hello' }))}>Send</button>
    </div>
  );
}
```

**The stale closure trap:**

Without the ref pattern, the `onMessage` callback is captured by the effect's closure at the time the effect runs. If the effect has an empty dependency array (or doesn't re-run when `onMessage` changes), the WebSocket's `onmessage` handler holds a reference to the **original** `onMessage` from the first render. Any state the callback closes over (like `messages`) is permanently stale.

```typescript
// BROKEN — stale closure
useEffect(() => {
  const ws = new WebSocket(url);
  ws.onmessage = (event) => {
    onMessage(event.data); // Captures onMessage from first render forever
  };
  return () => ws.close();
}, [url]); // onMessage not in deps — stale. Adding it to deps would reconnect on every render.
```

If you add `onMessage` to the dependency array, the WebSocket disconnects and reconnects on every render (since `onMessage` is a new function reference each time). Neither option works.

**The ref pattern solves this** by storing the latest `onMessage` in a ref (`onMessageRef.current = onMessage`) on every render. The WebSocket handler reads from `ref.current`, which always points to the latest version. The effect doesn't need `onMessage` in its dependencies because refs are mutable containers outside the closure system.

**`useEffectEvent` (experimental) is the React-blessed solution:**

```typescript
// When stable — replaces the manual ref pattern
const onWsMessage = useEffectEvent((data: unknown) => {
  onMessage(data); // Always reads latest onMessage, never stale
});

useEffect(() => {
  const ws = new WebSocket(url);
  ws.onmessage = (event) => onWsMessage(JSON.parse(event.data));
  return () => ws.close();
}, [url]);
```

`useEffectEvent` wraps a function so it always sees the latest values without being a dependency. It's the official escape hatch for "I need current values in an effect without re-running the effect."

</details>

## Practical — Forms & Modern APIs

<details>
<summary>19. Build a multi-step form with validation using React Hook Form and a schema validation library (Zod or Yup) -- show the TypeScript code for the form setup, per-step validation, error display, and submission handling. Explain why React Hook Form's uncontrolled approach performs better than Formik's controlled approach for large forms, and what breaks if you mix controlled and uncontrolled patterns.</summary>

```tsx
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';
import { useState } from 'react';

// --- Schemas per step ---
const personalSchema = z.object({
  firstName: z.string().min(1, 'Required'),
  lastName: z.string().min(1, 'Required'),
  email: z.string().email('Invalid email'),
});

const addressSchema = z.object({
  street: z.string().min(1, 'Required'),
  city: z.string().min(1, 'Required'),
  zipCode: z.string().regex(/^\d{5}$/, 'Must be 5 digits'),
});

const paymentSchema = z.object({
  cardNumber: z.string().regex(/^\d{16}$/, 'Must be 16 digits'),
  expiry: z.string().regex(/^\d{2}\/\d{2}$/, 'Format: MM/YY'),
});

// Combined schema for final submission
const fullSchema = personalSchema.merge(addressSchema).merge(paymentSchema);
type FormData = z.infer<typeof fullSchema>;

// Step schemas mapped by index
const stepSchemas = [personalSchema, addressSchema, paymentSchema] as const;

// --- Step Components ---
function PersonalStep({ register, errors }: StepProps) {
  return (
    <div>
      <h3>Personal Info</h3>
      <div>
        <input {...register('firstName')} placeholder="First Name" />
        {errors.firstName && <span className="error">{errors.firstName.message}</span>}
      </div>
      <div>
        <input {...register('lastName')} placeholder="Last Name" />
        {errors.lastName && <span className="error">{errors.lastName.message}</span>}
      </div>
      <div>
        <input {...register('email')} placeholder="Email" />
        {errors.email && <span className="error">{errors.email.message}</span>}
      </div>
    </div>
  );
}

function AddressStep({ register, errors }: StepProps) {
  return (
    <div>
      <h3>Address</h3>
      <input {...register('street')} placeholder="Street" />
      {errors.street && <span className="error">{errors.street.message}</span>}
      <input {...register('city')} placeholder="City" />
      {errors.city && <span className="error">{errors.city.message}</span>}
      <input {...register('zipCode')} placeholder="ZIP Code" />
      {errors.zipCode && <span className="error">{errors.zipCode.message}</span>}
    </div>
  );
}

function PaymentStep({ register, errors }: StepProps) {
  return (
    <div>
      <h3>Payment</h3>
      <input {...register('cardNumber')} placeholder="Card Number" />
      {errors.cardNumber && <span className="error">{errors.cardNumber.message}</span>}
      <input {...register('expiry')} placeholder="MM/YY" />
      {errors.expiry && <span className="error">{errors.expiry.message}</span>}
    </div>
  );
}

interface StepProps {
  register: ReturnType<typeof useForm<FormData>>['register'];
  errors: ReturnType<typeof useForm<FormData>>['formState']['errors'];
}

// --- Main Form ---
function MultiStepForm() {
  const [step, setStep] = useState(0);
  const {
    register,
    handleSubmit,
    trigger,
    formState: { errors, isSubmitting },
  } = useForm<FormData>({
    resolver: zodResolver(fullSchema),
    mode: 'onTouched', // Validate on blur, then on change after first error
  });

  const steps = [PersonalStep, AddressStep, PaymentStep];
  const CurrentStep = steps[step];

  // Validate only the current step's fields before advancing
  const handleNext = async () => {
    const currentSchema = stepSchemas[step];
    const fieldsToValidate = Object.keys(currentSchema.shape) as (keyof FormData)[];

    const isValid = await trigger(fieldsToValidate);
    if (isValid) setStep(s => s + 1);
  };

  const onSubmit = async (data: FormData) => {
    await fetch('/api/checkout', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(data),
    });
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <div className="step-indicator">Step {step + 1} of {steps.length}</div>

      <CurrentStep register={register} errors={errors} />

      <div className="buttons">
        {step > 0 && (
          <button type="button" onClick={() => setStep(s => s - 1)}>Back</button>
        )}
        {step < steps.length - 1 ? (
          <button type="button" onClick={handleNext}>Next</button>
        ) : (
          <button type="submit" disabled={isSubmitting}>
            {isSubmitting ? 'Submitting...' : 'Submit'}
          </button>
        )}
      </div>
    </form>
  );
}
```

**Per-step validation with `trigger`:**

React Hook Form's `trigger` method validates specific fields without submitting. We extract the field names from the current step's Zod schema and pass them to `trigger`. This validates only the visible step, not the entire form. Fields on future steps haven't been filled yet and shouldn't block navigation.

**Why React Hook Form's uncontrolled approach is faster:**

- **Formik** uses controlled components — every keystroke calls `setState`, which re-renders the form component (and potentially all its children). A form with 30 fields re-renders 30 times for 30 keystrokes on a single field.
- **React Hook Form** stores values in refs via `register()`. Input changes go straight to the DOM — no `setState`, no re-render. React Hook Form only triggers a re-render when form-level state changes (errors appearing/disappearing, `isSubmitting` toggling). For a 30-field form, typing in one field causes zero re-renders until the user triggers validation.

This difference is negligible for small forms but dramatic for large forms with complex validation, conditional fields, or expensive child components.

**What breaks when mixing controlled and uncontrolled:**

```tsx
// BROKEN — mixing patterns
const { register, control } = useForm<FormData>();

// This input is uncontrolled via register
<input {...register('name')} />

// This input is controlled via useState — RHF doesn't know about it
const [email, setEmail] = useState('');
<input value={email} onChange={e => setEmail(e.target.value)} />
```

React Hook Form can't validate, track errors, or include `email` in the submitted data because it's not registered. You get split state — half managed by RHF, half by local state. For inputs that need programmatic control in RHF, use the `Controller` component or `useController` hook, which integrates controlled components into RHF's system.

</details>

## Practical — Debugging & Troubleshooting

<details>
<summary>20. You're adopting React Server Components and hitting hydration mismatch errors and unexpected 'use client' boundaries -- show a concrete example of a component tree that causes these errors, walk through how to identify which components are server vs client using error messages and React DevTools, and show the before/after code restructuring that correctly places the 'use client' boundary so server-fetched data flows to client interactive elements without serialization issues.</summary>

**Concrete example that causes errors:**

```tsx
// ProductPage.tsx (Server Component — no 'use client')
import { ProductDetails } from './ProductDetails';

export default async function ProductPage({ params }: { params: { id: string } }) {
  const product = await db.products.findById(params.id);

  return (
    <div>
      <h1>{product.name}</h1>
      <ProductDetails product={product} />
    </div>
  );
}

// ProductDetails.tsx — PROBLEM: uses hooks but no 'use client'
import { useState } from 'react';

export function ProductDetails({ product }: { product: Product }) {
  const [quantity, setQuantity] = useState(1);
  // Error: useState can only be used in Client Components.
  // Add 'use client' directive at the top of the file.
  return (
    <div>
      <p>{product.description}</p>
      <p>Updated: {product.updatedAt.toLocaleDateString()}</p> {/* Serialization error */}
      <input type="number" value={quantity} onChange={e => setQuantity(+e.target.value)} />
    </div>
  );
}
```

**Two errors here:**

1. **Missing `'use client'`** — `ProductDetails` uses `useState` but isn't marked as a Client Component. React throws: "You're importing a component that needs useState. It only works in a Client Component."

2. **Serialization issue** — `product.updatedAt` is a `Date` object. Props crossing the server/client boundary must be serializable. Date objects aren't serializable — you'll get: "Only plain objects, and a few built-ins, can be passed to Client Components."

**How to identify server vs client in error messages and DevTools:**

- Next.js error overlay shows the exact file and line, with a message like "useState only works in Client Components" and a link to the offending import.
- React DevTools (with RSC support) shows server-rendered components with a different badge or grayed out — they don't appear in the hooks inspector because they have no client-side instance.
- Check the import chain: any file imported by a `'use client'` file is also client. Any file without `'use client'` that doesn't import hooks/browser APIs is server by default.

**After — correctly structured:**

```tsx
// ProductPage.tsx (Server Component)
import { ProductInfo } from './ProductInfo';
import { QuantitySelector } from './QuantitySelector';

export default async function ProductPage({ params }: { params: { id: string } }) {
  const product = await db.products.findById(params.id);

  return (
    <div>
      <h1>{product.name}</h1>
      {/* Server Component — no interactivity needed, renders static HTML */}
      <ProductInfo
        description={product.description}
        updatedAt={product.updatedAt.toISOString()} // Serialize Date to string
      />
      {/* Client Component — only the interactive part crosses the boundary */}
      <QuantitySelector productId={product.id} price={product.price} />
    </div>
  );
}

// ProductInfo.tsx (Server Component — no 'use client', no hooks)
export function ProductInfo({ description, updatedAt }: {
  description: string;
  updatedAt: string; // Receives pre-serialized string
}) {
  return (
    <div>
      <p>{description}</p>
      <p>Updated: {new Date(updatedAt).toLocaleDateString()}</p>
    </div>
  );
}

// QuantitySelector.tsx
'use client';
import { useState } from 'react';

export function QuantitySelector({ productId, price }: {
  productId: string;
  price: number;  // Only serializable primitives cross the boundary
}) {
  const [quantity, setQuantity] = useState(1);

  return (
    <div>
      <input type="number" value={quantity} onChange={e => setQuantity(+e.target.value)} />
      <p>Total: ${(price * quantity).toFixed(2)}</p>
      <button onClick={() => addToCart(productId, quantity)}>Add to Cart</button>
    </div>
  );
}
```

**Key restructuring principles applied:**

1. **Push `'use client'` as deep as possible.** Only `QuantitySelector` needs interactivity. `ProductInfo` stays as a Server Component — its code never ships to the client bundle.

2. **Serialize at the boundary.** Convert `Date` to ISO string before passing to client. The client converts back if needed. Pass only primitives and plain objects across the boundary.

3. **Split interactive from static.** The original `ProductDetails` mixed display and interaction. Separating them lets the static part stay server-rendered (zero JS cost) while only the interactive part hydrates.

4. **Hydration mismatches** occur when server HTML doesn't match client render. Common causes: using `Date.now()`, `Math.random()`, or browser-only APIs during render. Fix: move non-deterministic values to `useEffect` (client-only) or pass them as props from the server.

</details>

<details>
<summary>21. A useEffect callback is logging stale state values even though the state has clearly updated — walk through why this happens (the closure captures the value at render time), how to identify stale closure bugs using console logging and the React DevTools hooks inspector, and show the different fix patterns (dependency array correction, ref pattern, functional state updates) with tradeoffs of each?</summary>

**Why this happens — the closure model:**

Every render creates a new scope. When `useEffect` runs, its callback closes over the state values from that specific render. If the effect's dependency array doesn't include the state variable, the effect doesn't re-run when that state changes — so the callback permanently sees the value from the render when it was created.

```tsx
function Counter() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    const id = setInterval(() => {
      console.log('Count is:', count); // Always logs 0 — stale closure
    }, 1000);
    return () => clearInterval(id);
  }, []); // Empty deps — effect runs once, closes over count = 0

  return <button onClick={() => setCount(c => c + 1)}>Count: {count}</button>;
}
```

The button updates `count` to 1, 2, 3... but the interval always logs 0 because it captured the initial closure.

**How to identify stale closure bugs:**

1. **Console logging pattern:** Add a log inside the effect callback AND outside it in the render body. If the render log shows the updated value but the effect log shows the old value, the effect has a stale closure.

```tsx
console.log('Render sees count:', count);  // Shows 5
useEffect(() => {
  console.log('Effect sees count:', count); // Shows 0 — stale
}, []);
```

2. **React DevTools hooks inspector:** Select the component → look at the Hooks section. `useState` shows the current value (e.g., 5). If your effect is logging 0, there's a mismatch between what the hook holds and what the effect sees.

3. **ESLint `exhaustive-deps` rule:** The `react-hooks/exhaustive-deps` ESLint rule warns when a dependency is missing from the array. This catches most stale closure bugs at lint time.

**Fix patterns and tradeoffs:**

**1. Dependency array correction — add the missing dependency:**

```tsx
useEffect(() => {
  const id = setInterval(() => {
    console.log('Count is:', count); // Now sees current count
  }, 1000);
  return () => clearInterval(id);
}, [count]); // Re-runs effect every time count changes
```

- **Tradeoff:** The interval is torn down and recreated on every count change. For intervals/subscriptions, this means a new timer starts each time. Usually fine, but can cause timing drift or unnecessary reconnections for WebSockets.
- **When to use:** Default fix. Use this unless the teardown/setup cost is a problem.

**2. Functional state updates — avoid reading state entirely:**

```tsx
useEffect(() => {
  const id = setInterval(() => {
    setCount(prev => {
      console.log('Count is:', prev); // prev is always current
      return prev + 1;
    });
  }, 1000);
  return () => clearInterval(id);
}, []); // No dependency on count — effect runs once
```

- **Tradeoff:** Only works when you need to update the state based on its previous value. Doesn't help if you need to read state without updating it (e.g., logging, conditional logic).
- **When to use:** When the effect's job is to increment/transform state. The most common and cleanest fix for timer-based updates.

**3. Ref pattern — escape the closure system:**

```tsx
const countRef = useRef(count);
countRef.current = count; // Update on every render

useEffect(() => {
  const id = setInterval(() => {
    console.log('Count is:', countRef.current); // Always current
  }, 1000);
  return () => clearInterval(id);
}, []); // Effect runs once, reads current value via ref
```

- **Tradeoff:** Mutable ref is outside React's reactive model — you must remember to keep it in sync. It's an escape hatch, not a default pattern.
- **When to use:** When you need to read (not update) current state inside a long-lived callback (WebSocket handler, interval, third-party library callback) without re-running the effect. This is the pattern used in the WebSocket hook (question 18).

**Summary of when to use each:**

| Pattern | Best for | Limitation |
|---|---|---|
| Add to deps | Most cases | Tears down/recreates effect on each change |
| Functional update | Updating state based on previous value | Can't just read state without updating |
| Ref | Reading current state in long-lived callbacks | Manual sync, outside React's model |

</details>

---

## Experience-Based Questions

These questions test real-world experience. Prepare by mapping them to your own projects and situations.

<details>
<summary>22. Tell me about a time you had to significantly improve the performance of a React application — what were the symptoms, how did you profile and identify the bottlenecks, what changes did you make, and what was the measurable impact?</summary>

**What the interviewer is looking for:**

- Systematic approach to performance problems (measure first, then optimize — not guessing).
- Familiarity with profiling tools (React DevTools Profiler, Chrome Performance tab, Lighthouse).
- Ability to identify root causes vs symptoms.
- Quantifiable results, not vague "it felt faster."

**Suggested structure (STAR):**

1. **Situation:** What was the app, what was the performance problem, how did users experience it?
2. **Task:** What was your role, what was the target (e.g., reduce input lag, improve TTI)?
3. **Action:** How did you profile, what did you find, what specific changes did you make?
4. **Result:** What were the measurable improvements?

**Key points to hit:**

- Start with user-reported symptoms (lag, jank, slow page loads) — not technical details.
- Mention the specific profiling tool and what it revealed (e.g., "React Profiler showed the product list re-rendering 200ms on every keystroke because the parent form state triggered full subtree re-renders").
- Describe concrete fixes: `React.memo`, state restructuring (colocating state), code splitting, virtualization (`react-window` for long lists), debouncing expensive operations.
- Quantify: "Reduced interaction-to-paint from 400ms to 60ms" or "Cut initial bundle from 1.2MB to 380KB."

**Example outline to personalize:**

> Our dashboard had a search input above a product table with 500+ rows. Users reported noticeable lag when typing. I opened React DevTools Profiler and recorded a typing session — the flamegraph showed the entire `ProductTable` (120ms render) re-rendering on every keystroke because the search state lived in the parent `Dashboard` component. I moved the search input into its own component with local state, wrapped `ProductTable` in `React.memo`, and memoized the filtered results with `useMemo`. For the table itself, I added `react-window` to virtualize rows. Typing lag dropped from ~400ms to unnoticeable (<16ms), and the table renders only the visible 20 rows instead of all 500.

</details>

<details>
<summary>23. Describe a time you chose a state management approach for a complex feature or application — what were the requirements, what options did you evaluate, what did you pick and why, and would you make the same choice today?</summary>

**What the interviewer is looking for:**

- Thoughtful evaluation of tradeoffs — not just "we used Redux because it's popular."
- Understanding of different state categories (server state vs client state vs UI state).
- Ability to match tools to requirements, not requirements to tools.
- Willingness to revisit decisions — the "would you choose the same today?" tests growth.

**Suggested structure:**

1. **Context:** What was the feature/app, what were the state management needs (how many components shared state, how frequently it updated, server vs client state)?
2. **Options evaluated:** What did you consider and what were the pros/cons for your specific case?
3. **Decision:** What did you pick and what was the deciding factor?
4. **Outcome:** How well did it work in practice? Any surprises?
5. **Retrospective:** Would you choose the same today? What would you change?

**Key points to hit:**

- Show you categorized the state first (server cache vs shared client state vs local UI state) rather than picking one tool for everything.
- Mention concrete evaluation criteria: bundle size impact, learning curve for the team, performance characteristics, debugging experience.
- Be honest about tradeoffs — every choice has downsides. Interviewers distrust "it was perfect."

**Example outline to personalize:**

> We were building a multi-step order configuration tool where users could customize products, see live pricing, and save drafts. The state was complex: product config (deeply nested, frequently updated), pricing (server-calculated, needed caching), and UI state (which step, validation errors, modal visibility).
>
> I evaluated: (1) Redux Toolkit — powerful but heavy ceremony for our 3-person team, (2) Zustand — lightweight with selectors, (3) React Query + local state — separate server state from client state.
>
> We chose React Query for pricing data (server state with caching and background refetch) and Zustand for the product configuration (complex client state that multiple components read from with fine-grained selectors). UI state stayed in local `useState`.
>
> It worked well — the separation kept each concern clean. The one thing I'd change: we initially put draft-saving logic in Zustand middleware but later moved it to React Query mutations, which handled loading/error states better. Today I'd start with that separation from day one.

</details>

<details>
<summary>24. Tell me about a time you debugged a subtle React bug that was hard to reproduce — what were the symptoms, how did you narrow down the root cause, and what was the fix?</summary>

**What the interviewer is looking for:**

- Systematic debugging methodology (not "I guessed until it worked").
- Ability to narrow down from symptoms to root cause.
- Understanding of React-specific gotchas (stale closures, key-related bugs, race conditions in effects, hydration mismatches).
- Clear communication of a technical investigation.

**Suggested structure:**

1. **Symptoms:** What did users or QA report? Why was it hard to reproduce?
2. **Investigation:** How did you narrow it down? What tools did you use?
3. **Root cause:** What was actually wrong? Why was it subtle?
4. **Fix:** What did you change and how did you verify it?
5. **Prevention:** Did you add tests, linting rules, or patterns to prevent recurrence?

**Key points to hit:**

- Describe the reproduction difficulty (timing-dependent, data-dependent, environment-specific).
- Show tool usage: React DevTools, browser DevTools, console logging with timestamps, adding `key` props to force remounts for isolation, React strict mode double-render to surface issues.
- Name the React-specific concept at the root (stale closure, missing key, effect race condition, uncontrolled-to-controlled switch).

**Example outline to personalize:**

> Users reported that a filter dropdown occasionally showed the wrong selected value after navigating between pages. It only happened when switching quickly — slow navigation worked fine. I couldn't reproduce it consistently until I added console logs with timestamps to the filter component's `useEffect` and `onChange` handler.
>
> The logs revealed a race condition: navigating to a new page triggered a `useEffect` that fetched filter options and called `setSelectedFilter(defaultValue)`. But if the user had already selected a filter before the fetch completed, the `setSelectedFilter` in the effect overwrote their selection. The effect's cleanup didn't cancel the in-flight fetch.
>
> The fix was switching to React Query for the filter options (which handles cancellation and deduplication) and only setting the default filter if no user selection existed. I also added a regression test that simulated rapid navigation with pending fetches. The key lesson: `useEffect` fetches without cancellation are a race condition waiting to happen.

</details>

<details>
<summary>25. Describe a time you introduced a new pattern, library, or architectural change to a React codebase — how did you evaluate it, how did you get team buy-in, and how did the migration go?</summary>

**What the interviewer is looking for:**

- Technical judgment — ability to evaluate new tools critically, not just adopt what's trending.
- Influence without authority — how you convince a team, not just mandate a change.
- Migration pragmatism — incremental adoption vs big bang, handling the messy middle.
- Honest assessment of outcomes, including what didn't go well.

**Suggested structure:**

1. **Context:** What was the pain point that motivated the change?
2. **Evaluation:** What alternatives did you consider? What criteria did you use (performance, DX, bundle size, team familiarity, maintenance burden)?
3. **Buy-in:** How did you propose it? Proof of concept, RFC/doc, tech talk, pair session?
4. **Migration:** How did you execute — incremental or all-at-once? How did you handle the coexistence period?
5. **Outcome:** What improved? What was harder than expected?

**Key points to hit:**

- Start with the problem, not the solution. "We had X problem" is stronger than "I wanted to use Y library."
- Show you evaluated tradeoffs, not just features. Bundle size, learning curve, long-term maintenance, community support.
- Describe the buy-in approach concretely: a working prototype in a branch, a comparison doc with benchmarks, or a lunch-and-learn demo.
- Be honest about migration friction — coexisting old and new patterns, team members struggling with the new approach, unexpected edge cases.

**Example outline to personalize:**

> Our codebase had inconsistent data fetching — some components used `useEffect` + `useState`, others used a custom `useFetch` hook, and a few used SWR. Caching was ad-hoc, and we had duplicate requests on nearly every page.
>
> I evaluated React Query vs SWR vs continuing with our custom hook. I built a small proof-of-concept branch converting one data-heavy page to React Query, measured the network requests (dropped from 12 to 4 due to deduplication), and documented the comparison in a short RFC with code examples and bundle size impact.
>
> I presented it in our weekly tech sync. The main concern was learning curve — so I offered to pair with each team member on their first conversion and wrote a migration guide with before/after examples for our common patterns.
>
> We migrated incrementally — new features used React Query, and we converted existing pages during related refactors rather than a dedicated migration sprint. After ~6 weeks, 80% of data fetching was on React Query. The remaining 20% in rarely-touched pages stayed on the old pattern. The hardest part was establishing cache invalidation conventions — we initially over-invalidated, causing unnecessary refetches, and had to tune `staleTime` per query type.

</details>
