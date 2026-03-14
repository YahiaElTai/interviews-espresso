# Browser & Web Performance

> **25 questions** — 12 theory, 13 practical

- Critical rendering path: DOM, CSSOM, render-blocking resources (async, defer, module scripts), preload scanner
- Browser compositing architecture: layout, paint, composite phases — CSS property triggers (will-change, transform, opacity), GPU-accelerated animations, avoiding forced reflows
- JavaScript execution cost: parsing, compilation, main thread blocking, long tasks, impact on responsiveness (INP)
- Core Web Vitals: LCP, INP, CLS — diagnosis and optimization
- Lab data vs field data (RUM) for performance measurement
- SSR vs CSR vs SSG — tradeoffs and per-route mixing
- Hydration: cost, mismatches, progressive/partial hydration, streaming SSR
- HTTP caching: Cache-Control directives, ETag, conditional requests, CDN caching
- Bundle optimization: tree shaking, code splitting, vendor chunking, bundle analysis
- Image optimization: responsive images, srcset/sizes, lazy loading, WebP/AVIF
- Real user monitoring with web-vitals library and PerformanceObserver
- Resource loading optimization: resource hints (preconnect, preload, prefetch, dns-prefetch), critical CSS inlining, early hints (103)
---

## Foundational

<details>
<summary>1. How does the critical rendering path work end-to-end — what are DOM construction, CSSOM construction, and render tree formation, why are CSS and synchronous JS render-blocking by default, and why does understanding this pipeline matter for every performance optimization decision you make as a senior frontend engineer?</summary>

The critical rendering path is the sequence of steps the browser takes from receiving HTML bytes to painting pixels on screen.

**The pipeline:**

1. **HTML parsing and DOM construction** -- The browser parses HTML top-to-bottom, converting bytes into tokens, tokens into nodes, and nodes into the DOM tree. This is incremental -- the browser builds the DOM as bytes arrive, which is why streaming HTML matters.

2. **CSSOM construction** -- As the parser encounters `<link>` stylesheets or `<style>` blocks, it builds the CSS Object Model. Unlike the DOM, the CSSOM is not incremental -- a stylesheet must be fully parsed before the CSSOM is complete because CSS rules can override earlier rules (cascading).

3. **Render tree formation** -- The browser combines the DOM and CSSOM to create the render tree, which contains only visible nodes (elements with `display: none` are excluded). Each render tree node has computed styles attached.

4. **Layout** -- The browser calculates the exact position and size of every render tree node (box model geometry).

5. **Paint** -- The browser fills in pixels: text, colors, images, borders, shadows. This produces paint records (display lists).

6. **Composite** -- Paint layers are composited together and sent to the GPU for display.

**Why CSS is render-blocking:** The browser refuses to render anything until the CSSOM is complete. Without full CSS, the browser would render unstyled content (FOUC) and then re-render once styles arrive -- a terrible user experience. So stylesheets in `<head>` block rendering until they download and parse.

**Why synchronous JS is render-blocking:** When the HTML parser encounters a `<script>` tag without `async` or `defer`, it must stop DOM construction, wait for the script to download, then execute it. This is because the script might call `document.write()` or modify the DOM, so the parser cannot safely continue. This blocks both DOM construction and rendering.

**The preload scanner** partially mitigates this. While the main parser is blocked waiting for a script to execute, a secondary lightweight parser scans ahead in the HTML looking for resources to fetch (`<script>`, `<link>`, `<img>`). This means resources are discovered and fetched in parallel even though DOM construction is paused.

**Why this matters for every optimization decision:** Every performance technique -- `async`/`defer`, critical CSS inlining, code splitting, preload hints -- maps back to unblocking a specific stage of this pipeline. Without understanding it, you are guessing at optimizations rather than targeting the actual bottleneck.

</details>

<details>
<summary>2. How does the browser's compositing architecture split rendering into layout, paint, and composite phases — why does the browser separate these into distinct stages, what triggers each phase to re-run, and why does understanding which CSS properties trigger layout vs paint vs composite-only changes fundamentally change how you write performant animations and UI updates?</summary>

**Why three phases exist:** The browser separates rendering into layout, paint, and composite because each phase is progressively more expensive. By splitting them, the browser can skip earlier phases when a change only affects later ones -- a composite-only change skips layout and paint entirely.

**What triggers each phase:**

| Phase | What it computes | Triggered by changes to |
|-------|-----------------|------------------------|
| **Layout** | Geometry (position, size) of every element | `width`, `height`, `margin`, `padding`, `top`/`left`, `font-size`, `display`, `flex` properties, reading layout properties like `offsetHeight` |
| **Paint** | Fill pixels (colors, text, shadows, borders) | `color`, `background`, `border-color`, `box-shadow`, `visibility`, `text-decoration` |
| **Composite** | Combine pre-painted layers using the GPU | `transform`, `opacity`, `filter`, `will-change` |

The key insight: each phase triggers everything after it. A layout change triggers layout + paint + composite. A paint change triggers paint + composite. A composite change triggers only compositing.

**Compositor layers and GPU acceleration:** When you use `transform` or `opacity`, the browser promotes the element to its own compositor layer. This layer is a bitmap texture on the GPU. Animating it means only changing how the GPU composites textures together -- the CPU does not re-layout or re-paint anything. This is why `transform: translateX()` animations run at 60fps while `left: Npx` animations often drop frames.

**Forced reflow (layout thrashing):** The most dangerous pattern is reading a layout property (like `offsetHeight`) immediately after writing a style change. The browser must synchronously recalculate layout to return an accurate value, even if it would have batched that change otherwise.

```typescript
// BAD: layout thrashing -- forces synchronous layout on every iteration
elements.forEach((el) => {
  const height = el.offsetHeight; // read -> forces layout
  el.style.width = `${height * 2}px`; // write -> invalidates layout
});

// GOOD: batch reads, then batch writes
const heights = elements.map((el) => el.offsetHeight); // all reads
elements.forEach((el, i) => {
  el.style.width = `${heights[i] * 2}px`; // all writes
});
```

**Practical impact on animations:**

```css
/* BAD: triggers layout on every frame */
.animate-bad {
  transition: left 0.3s, top 0.3s;
}

/* GOOD: composite-only, GPU-accelerated */
.animate-good {
  transition: transform 0.3s;
  will-change: transform; /* hint to promote to own layer */
}
```

**`will-change` gotcha:** It creates a new compositor layer, which consumes GPU memory. Applying `will-change` to dozens of elements wastes memory and can actually hurt performance. Use it only on elements that will animate, and remove it when the animation ends.

</details>

<details>
<summary>3. What are Core Web Vitals (LCP, INP, CLS), why did Google choose these three specific metrics over alternatives like Time to Interactive or First Contentful Paint, and what does each one actually measure from the user's perspective, and how do they connect to real-world user experience and business outcomes?</summary>

Core Web Vitals are three metrics that map to the three fundamental aspects of user-perceived page experience:

| Metric | What it measures (user perspective) | Good threshold | Question it answers |
|--------|-------------------------------------|---------------|-------------------|
| **LCP** (Largest Contentful Paint) | How fast the main content appears | < 2.5s | "Can I see what I came for?" |
| **INP** (Interaction to Next Paint) | How quickly the page responds to input | < 200ms | "Does it feel responsive when I click/type?" |
| **CLS** (Cumulative Layout Shift) | How much the page unexpectedly jumps around | < 0.1 | "Is the page visually stable?" |

**Why these three over alternatives:** FCP measures when *any* content appears (often a spinner), not meaningful content -- LCP is more representative. TTI measured a synthetic "quiet window" with no real user interaction and was noisy in the field -- INP measures actual interaction latency. FID only captured the *first* interaction's delay, ignoring processing and presentation time -- INP captures the worst interaction across the entire session.

**How each metric works technically:**

- **LCP** identifies the largest image, video, or text block in the viewport and records when it finishes rendering. It updates as larger elements appear (the hero image replacing a heading, for example) and finalizes when the user first interacts.
- **INP** tracks every discrete interaction (click, tap, keypress -- not scroll) throughout the page's life, measuring from input event to next paint. The reported value is approximately the worst interaction (p98 to account for outliers).
- **CLS** sums up layout shift scores for unexpected element movements. Each shift score = impact fraction x distance fraction. Only *unexpected* shifts count -- shifts caused by user interaction (clicking a button that expands content) are excluded.

**Business outcomes:** Google uses CWV as a ranking signal. Beyond SEO, studies consistently show that faster LCP increases conversion rates, lower INP reduces bounce rates, and stable CLS reduces rage clicks. These are not vanity metrics -- they directly correlate with revenue.

</details>

## Conceptual Depth

<details>
<summary>4. Why do you need both lab data (Lighthouse, WebPageTest) and field data (RUM via CrUX, web-vitals library) to understand performance — what does each one capture that the other misses, when can lab data actively mislead you about real user experience, and how do you reconcile conflicting results between the two?</summary>

**Lab data** runs tests in controlled environments with fixed device/network settings. **Field data** (Real User Monitoring) captures actual user experiences across diverse devices, networks, and usage patterns.

**What each captures that the other misses:**

| Aspect | Lab data | Field data (RUM) |
|--------|----------|-------------------|
| **Reproducibility** | Deterministic, same conditions each run | Varies by user -- different devices, networks, geography |
| **INP coverage** | Cannot measure real INP (no real user interactions) | Captures actual interaction patterns across the full session |
| **CLS accuracy** | Only measures initial load CLS | Captures CLS from lazy-loaded content, infinite scroll, SPA navigations |
| **Debugging** | Full waterfall, flame charts, screenshots | Attribution data only (element selectors, timing breakdowns) |
| **Device diversity** | Fixed simulated device | Real distribution of low-end Android, iPhones, desktops |
| **Third-party impact** | Often blocked or absent | Full impact of ads, analytics, chat widgets |

**When lab data actively misleads:**

1. **LCP on slow devices** -- Lighthouse simulates a mid-tier phone, but your real users might be on low-end Android devices 3x slower. Lab shows 2.1s LCP, field shows 4.5s.
2. **CLS from dynamic content** -- Lab tests only measure load-time CLS. In the field, CLS accumulates from late-loading ads, cookie banners, and SPA route transitions.
3. **Third-party scripts** -- Lab tests often exclude or under-represent third-party script impact. In production, that analytics bundle and chat widget add 500ms of main-thread blocking.
4. **Caching** -- Lab usually tests cold cache. Your repeat visitors (70%+ of traffic) experience warm-cache performance that lab never measures.

**Reconciling conflicts:**

When lab says "fast" but field says "slow," the field is always right about user experience. Use the conflict as a diagnostic signal:

- **Lab LCP < field LCP** -- Look at device distribution in CrUX. If most users are on slow mobile, your lab device profile is too optimistic. Also check if the LCP element is different in the field (images loaded via JS vs statically in HTML).
- **Lab CLS = 0 but field CLS = 0.3** -- Check for layout shifts after load: lazy images without dimensions, dynamically injected banners, font swaps on slow connections.
- Use lab data to **diagnose and iterate** (it gives you waterfalls and flame charts). Use field data to **measure what matters** (actual user experience at p75).

</details>

<details>
<summary>5. Why do different rendering strategies (SSR, CSR, SSG) exist — what specific problem does each one solve, what are the performance and SEO tradeoffs of each, and when does each approach become the wrong choice?</summary>

Each rendering strategy optimizes for a different constraint. There is no universal best choice.

**Client-Side Rendering (CSR):**
- The server sends a minimal HTML shell with a JS bundle. The browser downloads JS, executes it, fetches data, and renders the UI.
- **Solves:** Maximum interactivity for app-like experiences. Simple infrastructure (static hosting). Clean separation of frontend/backend.
- **Performance:** Slow LCP (blank screen until JS executes and data loads). Fast subsequent navigations (SPA).
- **SEO:** Poor by default -- crawlers see empty HTML. Googlebot can execute JS but with delays and quotas.
- **Wrong choice when:** SEO matters, initial load speed matters, users are on slow devices/networks.

**Server-Side Rendering (SSR):**
- The server renders full HTML for each request. Browser receives complete content, then hydrates it with JS for interactivity.
- **Solves:** Fast FCP/LCP (content visible before JS loads). SEO (crawlers see full HTML). Personalized/dynamic content.
- **Performance:** Good LCP, but TTFB is slower (server must render). Hydration cost delays interactivity. Server compute scales with traffic.
- **SEO:** Excellent -- full HTML on first response.
- **Wrong choice when:** Content is identical for all users (SSG is better), server infra budget is limited, or pages are highly interactive dashboards where SSR adds complexity without LCP benefit.

**Static Site Generation (SSG):**
- Pages are pre-rendered at build time. Served as static HTML from CDN edge.
- **Solves:** Fastest possible TTFB (CDN edge, no server compute). Perfect for content that changes infrequently. Scales infinitely.
- **Performance:** Best LCP of all strategies (pre-rendered + CDN cached). Zero server compute per request.
- **SEO:** Excellent -- same as SSR but even faster TTFB.
- **Wrong choice when:** Content changes frequently (rebuilds are slow), content is personalized per user, or the site has millions of pages (build times become prohibitive).

**The key tradeoff triangle:** Speed of first paint vs infrastructure complexity vs content freshness. CSR sacrifices first paint. SSR adds infrastructure. SSG sacrifices freshness.

</details>

<details>
<summary>6. Why is per-route rendering strategy mixing (using SSR for some pages, SSG for others, CSR for others) often the right answer instead of picking one approach globally — what factors determine which strategy a given route should use, and what complexity does mixing introduce in your build and deployment pipeline?</summary>

A typical web application has routes with fundamentally different requirements. A marketing homepage needs SEO and fast LCP (SSG). A product page needs fresh inventory data (SSR or ISR). A user dashboard is behind auth and does not need SEO (CSR). Forcing all three into a single strategy means at least two are suboptimal.

**Decision factors per route:**

| Factor | SSG | SSR | CSR |
|--------|-----|-----|-----|
| Needs SEO? | Yes | Yes | No |
| Content changes per request? | No | Yes | Yes |
| Personalized per user? | No | Yes | Yes |
| Number of possible pages? | Manageable (< 50k) | Unlimited | Unlimited |
| Performance priority | Maximum LCP | Good LCP with fresh data | Interactivity over first paint |

**Practical example in Next.js (App Router):**

```typescript
// SSG: marketing pages -- static by default in App Router
// app/about/page.tsx
export default async function AboutPage() {
  const content = await getCMSContent("about"); // fetched at build time by default
  return <AboutContent content={content} />;
}

// SSR: product pages -- opt into dynamic rendering
// app/products/[id]/page.tsx
export const dynamic = "force-dynamic"; // fetch fresh data per request
export default async function ProductPage({ params }: { params: { id: string } }) {
  const product = await getProduct(params.id);
  return <ProductDetails product={product} />;
}

// ISR: revalidate periodically without full rebuild
// app/blog/[slug]/page.tsx
export const revalidate = 3600; // regenerate at most every hour

// CSR: dashboard -- no SEO, behind auth
// app/dashboard/page.tsx
"use client";
export default function Dashboard() {
  const { data } = useSWR("/api/dashboard", fetcher);
  return <DashboardUI data={data} />;
}
```

*Note: The Pages Router (`getStaticProps`, `getServerSideProps`) still works but the App Router with Server Components is the current recommended approach. `generateStaticParams` replaces `getStaticPaths` for dynamic SSG routes.*

**Complexity mixing introduces:**

1. **Build pipeline** -- SSG pages require a build step that fetches data and generates HTML. SSR pages need a running server. CSR pages are just static bundles. Your deployment must handle all three.
2. **Caching strategy diverges** -- SSG pages get `Cache-Control: public, max-age=31536000, immutable`. SSR pages need `s-maxage` with `stale-while-revalidate` at the CDN. CSR pages serve a static shell with `no-cache` on the HTML.
3. **Mental model for developers** -- Each page has different data-fetching patterns and different behavior on navigation. New team members need to understand which strategy each route uses and why.
4. **ISR (Incremental Static Regeneration)** adds another axis -- SSG pages that revalidate in the background. This blurs the line between SSG and SSR and introduces stale content windows.

The complexity is worth it because the alternative -- one strategy globally -- means either over-engineering simple pages or under-serving complex ones.

</details>

<details>
<summary>7. Why is hydration expensive and error-prone — what actually happens during hydration, what causes hydration mismatches and why are they dangerous, how do progressive hydration, partial hydration (islands architecture), and streaming SSR each address the hydration cost problem, and what are the tradeoffs of each approach?</summary>

**What happens during hydration:** The server renders HTML and sends it to the client. The browser displays this HTML immediately (fast FCP/LCP). Then React (or your framework) downloads the full JS bundle, re-executes the entire component tree in memory, and "attaches" event handlers to the existing DOM nodes by walking the server-rendered HTML and matching it to the virtual DOM. The user sees content before hydration but cannot interact with it -- the "uncanny valley" where buttons are visible but unresponsive.

**Why it is expensive:**
- The entire component tree must re-execute on the client, including all the logic that already ran on the server
- All JS for every component must be downloaded and parsed, even for static content that will never change
- This work runs on the main thread, blocking interactivity and directly hurting INP

**Hydration mismatches:** If the HTML the server rendered differs from what the client renders, React detects a mismatch. Common causes:
- Using `Date.now()` or `Math.random()` during render (different on server vs client)
- Browser-only APIs like `window.innerWidth` used during initial render
- Locale/timezone differences between server and client
- Third-party scripts modifying DOM before hydration completes

**Why mismatches are dangerous:** React must discard the server-rendered DOM and re-render from scratch (client-side), which destroys the SSR performance benefit and can cause a flash of content change visible to users.

**Solutions and their tradeoffs:**

**Progressive hydration** -- Hydrate components lazily based on priority. Above-the-fold interactive components hydrate first; below-the-fold components hydrate on scroll/idle. Reduces initial main-thread work.
- *Tradeoff:* Components are unresponsive until hydrated. Requires careful prioritization. Framework support varies.

**Partial hydration (Islands architecture)** -- Only hydrate interactive "islands" (a search bar, a carousel). Static content (headers, text, images) ships zero JS. Used by Astro.
- *Tradeoff:* Best JS reduction for content-heavy sites. But inter-island communication is complex, and the mental model differs from traditional SPA development.

**Streaming SSR with selective hydration (React 18+)** -- The server streams HTML in chunks as components resolve. `<Suspense>` boundaries define streaming units. On the client, React hydrates eagerly whichever `<Suspense>` boundary the user interacts with first.

```tsx
// Server streams HTML progressively
// Components inside Suspense boundaries stream as they resolve
<Suspense fallback={<Skeleton />}>
  <Comments /> {/* streams when data is ready */}
</Suspense>
```

- *Tradeoff:* Improves TTFB (no waiting for slowest data fetch). Selective hydration prioritizes what the user is interacting with. But you still ship JS for all components -- it does not reduce bundle size like islands architecture does.

**Summary:** Streaming SSR fixes the TTFB problem. Progressive hydration fixes the "hydrate everything at once" problem. Islands architecture fixes the "ship JS for everything" problem. They are complementary and increasingly combined in modern frameworks.

</details>

<details>
<summary>8. How does the HTTP caching model work for frontend assets — what do the key Cache-Control directives (max-age, no-cache, no-store, immutable, stale-while-revalidate) actually mean, how do ETag and conditional requests (If-None-Match) prevent unnecessary transfers, and how does CDN caching layer on top of browser caching to create a multi-tier cache with different invalidation concerns?</summary>

**Cache-Control directives:**

| Directive | What it means |
|-----------|--------------|
| `max-age=N` | Cache is fresh for N seconds. Browser uses cached copy without contacting the server during this window. |
| `no-cache` | The response CAN be cached, but must revalidate with the server before every use. Misleading name -- it does not mean "don't cache." |
| `no-store` | Do not cache at all. Not in memory, not on disk. Used for sensitive data. |
| `immutable` | The resource will never change at this URL. Browser should not even revalidate on user refresh. Used with content-hashed filenames. |
| `stale-while-revalidate=N` | After `max-age` expires, serve the stale version immediately while fetching a fresh copy in the background. User gets instant response; cache updates silently. |
| `s-maxage=N` | Like `max-age` but only for shared caches (CDNs). Overrides `max-age` at the CDN layer. |
| `public` / `private` | `public` allows CDNs to cache. `private` restricts to browser only (for user-specific responses). |

**ETag and conditional requests:**

When `max-age` expires (or with `no-cache`), the browser sends a conditional request:

1. Server originally responds with `ETag: "abc123"` (a hash of the content)
2. Browser's cache entry expires
3. Browser sends `GET /style.css` with `If-None-Match: "abc123"`
4. Server checks: if content has not changed, responds `304 Not Modified` (no body) -- saves bandwidth
5. If content changed, responds `200` with new body and new ETag

`Last-Modified` / `If-Modified-Since` works similarly but uses timestamps. ETags are more precise.

**Multi-tier caching (Browser + CDN):**

```
User -> Browser Cache -> CDN Edge Cache -> Origin Server
```

Each tier has different invalidation concerns:

- **Browser cache:** Controlled by `max-age`. You cannot remotely invalidate it -- once the browser has a cached response, it will use it until `max-age` expires. This is why `index.html` must never have a long `max-age`.
- **CDN cache:** Controlled by `s-maxage`. You CAN invalidate CDN caches (purge API, surrogate keys). CDN operators have control; browser caches are under user control only.
- **Cache busting strategy:** Hash filenames for assets (`main.a1b2c3.js`). The HTML references the hash. New deployment = new HTML with new hash = new URL = cache miss. Old cached bundles naturally expire.

The critical insight: you control CDN cache invalidation but not browser cache invalidation. This asymmetry is why `index.html` uses `no-cache` (always revalidate) while hashed assets use `max-age=31536000, immutable` (cache forever, new content = new URL).

</details>

<details>
<summary>9. Why do bundle size optimizations (tree shaking, code splitting, vendor chunking) matter even with fast networks — how does each technique work under the hood, what are the tradeoffs between aggressive code splitting (more requests, better caching) vs fewer larger bundles (fewer requests, worse caching), and when does over-splitting actually hurt performance?</summary>

Bundle size matters even on fast networks because **JavaScript has a cost beyond download time**. Every byte of JS must be parsed, compiled, and executed on the main thread. On a mid-range phone, 1MB of JS takes 2-4 seconds to parse and compile -- even if it downloaded in 200ms. This directly blocks interactivity and hurts INP.

**Tree shaking** -- The bundler (Vite/Rollup, Webpack) statically analyzes ES module `import`/`export` statements and removes exported code that no module imports. It works because ES modules have static structure (unlike CommonJS `require()` which can be dynamic).

What breaks tree shaking:
- **Side effects** -- If a module executes code at import time (not just exporting), the bundler cannot safely remove it. Mark side-effect-free packages with `"sideEffects": false` in `package.json`.
- **CommonJS** -- `require()` is dynamic, so bundlers cannot statically determine what is used. This is why importing from `lodash` (CJS) bundles everything, while `lodash-es` (ESM) tree-shakes.
- **Re-exports via barrel files** -- `import { Button } from './components'` where `components/index.ts` re-exports 50 components can defeat tree shaking in some bundler configurations.

**Code splitting** -- Instead of one monolithic bundle, split code into chunks loaded on demand. Route-based splitting is the most common: each page loads only the JS it needs.

```tsx
// Route-based code splitting with React.lazy
const ProductPage = React.lazy(() => import("./pages/ProductPage"));
const Dashboard = React.lazy(() => import("./pages/Dashboard"));
```

**Vendor chunking** -- Separate third-party libraries (React, lodash) into a dedicated chunk. Since vendor code changes infrequently, this chunk stays cached across deployments while your app code chunk changes.

**The splitting tradeoff:**

| More chunks (aggressive splitting) | Fewer chunks (conservative) |
|-------------------------------------|----------------------------|
| Better cache hit rates (change one module, only that chunk invalidates) | Fewer HTTP requests (less connection overhead) |
| Smaller initial download (only what's needed) | Simpler debugging and source maps |
| Risk of waterfall loading (chunk A loads, discovers it needs chunk B) | Larger initial download (unused code shipped) |

**When over-splitting hurts:**
- **Tiny chunks (< 5KB)** -- HTTP/2 multiplexing helps, but each chunk still has per-request overhead (headers, TLS record framing) and the browser has limited parallel connections per domain.
- **Deep dependency chains** -- If chunk A dynamically imports B, which imports C, you create a loading waterfall. Prefetching helps but adds complexity.
- **Too many vendor chunks** -- Splitting every npm package into its own chunk creates dozens of requests on first load. Group related packages instead.

The sweet spot: route-based splitting + one stable vendor chunk + a shared commons chunk for code used by 2+ routes. Analyze with a bundle visualizer, not guesswork.

</details>

<details>
<summary>10. Why is image optimization the highest-impact performance win on most websites — what are the tradeoffs between WebP and AVIF formats, how do responsive images with srcset/sizes prevent unnecessary bandwidth usage, why does lazy loading below-the-fold images matter for LCP, and when can aggressive image optimization backfire?</summary>

Images account for 40-60% of total page weight on most websites. Optimizing them reduces more bytes than any other single technique.

**WebP vs AVIF:**

| | WebP | AVIF |
|---|------|------|
| **Compression** | 25-35% smaller than JPEG | 50%+ smaller than JPEG (better than WebP) |
| **Browser support** | 97%+ (all modern browsers) | ~93% (no IE, older Safari) |
| **Encode speed** | Fast | Slow (5-10x slower to encode) |
| **Features** | Lossy, lossless, transparency, animation | Lossy, lossless, transparency, HDR, wider color gamut |
| **Best for** | General use, safe default | Bandwidth-constrained users, maximum compression |

**Practical recommendation:** Serve AVIF with WebP fallback using `<picture>`. Use WebP-only if build-time encoding speed matters or AVIF support is insufficient for your audience.

**Responsive images with srcset/sizes:**

```html
<img
  srcset="hero-400.webp 400w, hero-800.webp 800w, hero-1200.webp 1200w"
  sizes="(max-width: 600px) 100vw, (max-width: 1200px) 50vw, 600px"
  src="hero-800.webp"
  alt="Hero image"
/>
```

- `srcset` tells the browser which image files are available and their widths
- `sizes` tells the browser how wide the image will display at each viewport size
- The browser picks the optimal file -- a 400px phone downloads 400w, a desktop downloads 1200w

**Why `sizes` matters:** Without `sizes`, the browser assumes the image is `100vw` wide and always downloads the largest source. If your image actually displays at `50vw` on desktop, the browser downloads 2x the pixels needed.

**Lazy loading and LCP:**

```html
<!-- LCP image: load eagerly, add fetchpriority hint -->
<img src="hero.webp" alt="Hero" fetchpriority="high" />

<!-- Below-the-fold images: lazy load -->
<img src="product.webp" alt="Product" loading="lazy" />
```

Lazy loading below-the-fold images frees bandwidth and main-thread time for the LCP image. But **never lazy-load the LCP image** -- `loading="lazy"` delays it until the browser determines it is in the viewport, which requires layout to complete first, adding hundreds of milliseconds to LCP.

**When aggressive optimization backfires:**
- **Over-compressing:** AVIF at very low quality creates visible artifacts, especially on text-heavy images
- **Too many srcset variants:** Generating 10 sizes per image bloats build time and CDN storage without meaningful user benefit. 3-4 breakpoints cover 95% of cases.
- **Lazy loading above-the-fold images:** As mentioned, this directly hurts LCP
- **Complex `<picture>` markup for tiny images:** The overhead of format negotiation is not worth it for icons or thumbnails under 5KB

</details>

<details>
<summary>11. How do resource hints (preconnect, preload, prefetch, dns-prefetch) and critical CSS inlining improve loading performance — what does each resource hint do, when does each one help vs waste bandwidth, how does critical CSS inlining eliminate render-blocking round trips, and what are the risks of over-using preload or inlining too much CSS?</summary>

**Resource hints:**

| Hint | What it does | When to use | When it wastes bandwidth |
|------|-------------|-------------|--------------------------|
| `dns-prefetch` | Resolves DNS for a domain ahead of time (~20-120ms saving) | Third-party domains you will use soon | Minimal cost, safe to over-use |
| `preconnect` | DNS + TCP + TLS handshake (~100-300ms saving) | Critical third-party origins (fonts, API, CDN) | Each connection consumes resources. Limit to 2-4 origins. |
| `preload` | Fetches a specific resource with high priority, for the current page | Resources the browser discovers late (fonts in CSS, JS-loaded images, async chunks) | If the resource is not used within 3s, Chrome warns. Preloading too many things dilutes priority. |
| `prefetch` | Fetches a resource at low priority for a future navigation | Next-page resources (route chunks the user is likely to navigate to) | User never navigates there -- wasted bandwidth. Avoid on metered connections. |

```html
<head>
  <link rel="dns-prefetch" href="https://analytics.example.com" />
  <link rel="preconnect" href="https://fonts.googleapis.com" crossorigin />
  <link rel="preload" href="/fonts/Inter.woff2" as="font" type="font/woff2" crossorigin />
  <link rel="prefetch" href="/chunks/dashboard.js" />
</head>
```

**Critical CSS inlining:**

Normally, CSS is in external stylesheets that must be downloaded before rendering begins. Critical CSS inlining extracts the CSS needed for above-the-fold content and embeds it directly in `<style>` tags in the HTML:

```html
<head>
  <!-- Critical CSS inlined -- no extra round trip -->
  <style>
    .header { display: flex; height: 64px; }
    .hero { font-size: 2rem; margin: 2rem; }
    /* only above-the-fold styles */
  </style>
  <!-- Full stylesheet loaded asynchronously -->
  <link rel="stylesheet" href="/styles.css" media="print" onload="this.media='all'" />
</head>
```

This eliminates the render-blocking round trip for CSS. The browser can render the above-the-fold content immediately from the inlined styles while the full stylesheet loads in the background.

**Risks of over-using preload:**
- Preloading too many resources (> 5-6) contends for bandwidth with actually critical resources, potentially slowing LCP
- Preloaded resources that are not used within ~3 seconds trigger console warnings and waste bandwidth
- Preload does not respect the browser's built-in priority heuristics -- you are overriding its scheduling, which can backfire

**Risks of inlining too much CSS:**
- Inlined CSS cannot be cached separately. If you inline 50KB of CSS, every page load re-downloads it in the HTML. External stylesheets are cached across pages.
- Increases HTML document size, which affects TTFB
- The sweet spot is typically 10-15KB of critical CSS. Beyond that, the caching tradeoff is not worth it.

**Early hints (103):** The server sends a `103 Early Hints` response before the full `200` response, telling the browser to start preconnecting/preloading while the server computes the page. This recovers the server think time as preloading time.

</details>

<details>
<summary>12. Why is JavaScript execution cost one of the biggest performance bottlenecks in modern web apps — what happens during parsing and compilation, why does main thread blocking create long tasks, how do long tasks directly degrade INP, and what strategies (code splitting, web workers, yielding to the main thread) reduce execution cost without sacrificing functionality?</summary>

**Why JS execution cost dominates:** Modern web apps ship 500KB-2MB+ of JavaScript. Unlike images (decoded off-thread), JS must be parsed, compiled, and executed on the main thread. On a median mobile device, every 100KB of JS takes ~100-200ms to parse/compile alone.

**The execution pipeline:**
1. **Download** -- Network transfer (compressible with gzip/brotli)
2. **Parse** -- Convert source text to AST (Abstract Syntax Tree). Cannot be skipped.
3. **Compile** -- V8 uses multiple compilation tiers: Sparkplug (fast baseline compiler) produces unoptimized code quickly, Maglev (mid-tier optimizing compiler) recompiles warm functions, and TurboFan (full optimizing compiler) recompiles hot functions with maximum optimization. Initial compilation is fast but produces slower code; optimization happens progressively over time.
4. **Execute** -- Run the compiled code. This is where your component trees render, event handlers attach, state initializes.

Steps 2-4 all happen on the **main thread**.

**Long tasks and INP:** A "long task" is any main-thread task exceeding 50ms. During a long task, the browser cannot respond to user input. If a user clicks a button while JS is executing a 200ms task, the input event queues until the task finishes. INP measures this exact delay -- the time from input to next paint.

INP breaks down into three parts:
- **Input delay** -- Time the event waits in the queue (caused by long tasks already running)
- **Processing duration** -- Time your event handler takes to execute
- **Presentation delay** -- Time for the browser to layout, paint, and composite after your handler

**Strategies to reduce execution cost:**

**Code splitting** (as covered in question 9) -- Ship less JS per page. If the user never visits the settings page, that 80KB of settings code never downloads or executes.

**Web Workers** -- Move heavy computation off the main thread:

```typescript
// worker.ts
self.onmessage = (e: MessageEvent<number[]>) => {
  const sorted = heavySort(e.data); // runs off main thread
  self.postMessage(sorted);
};

// main.ts
const worker = new Worker(new URL("./worker.ts", import.meta.url));
worker.postMessage(largeDataset);
worker.onmessage = (e) => renderResults(e.data);
```

Limitation: Workers cannot access the DOM. Good for data processing, parsing, image manipulation -- not for UI logic.

**Yielding to the main thread** -- Break long tasks into smaller chunks so the browser can process input between them:

```typescript
// Using scheduler.yield() (modern browsers)
async function processItems(items: Item[]) {
  for (const item of items) {
    processItem(item);

    // Yield control so the browser can handle pending interactions
    if (navigator.scheduling?.isInputPending?.() || items.indexOf(item) % 10 === 0) {
      await scheduler.yield();
    }
  }
}

// Fallback: setTimeout breaks the task
function yieldToMain(): Promise<void> {
  return new Promise((resolve) => setTimeout(resolve, 0));
}
```

**`requestIdleCallback`** -- Schedule non-urgent work during browser idle periods:

```typescript
requestIdleCallback((deadline) => {
  while (deadline.timeRemaining() > 0 && workQueue.length > 0) {
    processNextItem(workQueue.shift()!);
  }
});
```

**Practical priority:** Code splitting gives the biggest bang (less code = less everything). Yielding fixes specific long tasks. Workers are for heavy computation that is clearly separable from UI.

</details>

## Practical — Measurement & Auditing

<details>
<summary>13. Set up real user monitoring for Core Web Vitals using the web-vitals library and PerformanceObserver — show the code for collecting LCP, INP, and CLS in production, explain how you send these metrics to your analytics backend, what attribution data you capture to make the metrics actionable, and how you handle the differences between page loads and SPA navigations.</summary>

**Basic collection with the web-vitals library:**

```typescript
import { onCLS, onINP, onLCP } from "web-vitals/attribution";
import type { MetricWithAttribution } from "web-vitals";

interface VitalPayload {
  name: string;
  value: number;
  delta: number;
  id: string;
  rating: "good" | "needs-improvement" | "poor";
  navigationType: string;
  url: string;
  // Attribution fields
  debugTarget?: string;
  attribution: Record<string, unknown>;
}

function buildPayload(metric: MetricWithAttribution): VitalPayload {
  const base = {
    name: metric.name,
    value: metric.value,
    delta: metric.delta,
    id: metric.id,
    rating: metric.rating,
    navigationType: metric.navigationType,
    url: location.href,
  };

  switch (metric.name) {
    case "LCP":
      return {
        ...base,
        debugTarget: metric.attribution.target,
        attribution: {
          element: metric.attribution.target,
          url: metric.attribution.url,
          ttfb: metric.attribution.timeToFirstByte,
          resourceLoadDelay: metric.attribution.resourceLoadDelay,
          resourceLoadDuration: metric.attribution.resourceLoadDuration,
          renderDelay: metric.attribution.elementRenderDelay,
        },
      };
    case "INP":
      return {
        ...base,
        debugTarget: metric.attribution.interactionTarget,
        attribution: {
          target: metric.attribution.interactionTarget,
          type: metric.attribution.interactionType,
          inputDelay: metric.attribution.inputDelay,
          processingDuration: metric.attribution.processingDuration,
          presentationDelay: metric.attribution.presentationDelay,
          loadState: metric.attribution.loadState,
        },
      };
    case "CLS":
      return {
        ...base,
        debugTarget: metric.attribution.largestShiftTarget,
        attribution: {
          largestShiftTarget: metric.attribution.largestShiftTarget,
          largestShiftTime: metric.attribution.largestShiftTime,
          largestShiftValue: metric.attribution.largestShiftValue,
        },
      };
    default:
      return { ...base, attribution: {} };
  }
}

function sendToAnalytics(metric: MetricWithAttribution): void {
  const payload = buildPayload(metric);

  // Use sendBeacon for reliability -- fires even during page unload
  const blob = new Blob([JSON.stringify(payload)], { type: "application/json" });
  navigator.sendBeacon("/api/vitals", blob);
}

// Register all three
onLCP(sendToAnalytics);
onINP(sendToAnalytics);
onCLS(sendToAnalytics);
```

**Why attribution matters:** Without it, you know "LCP is 4.5s" but not why. With attribution, you know "LCP is 4.5s because the hero image (`img.hero-banner`) had 2.1s of resource load delay" -- now you know exactly what to fix.

**Key attribution fields per metric:**
- **LCP:** `timeToFirstByte`, `resourceLoadDelay`, `resourceLoadDuration`, `elementRenderDelay` -- tells you which phase of LCP loading is the bottleneck
- **INP:** `inputDelay`, `processingDuration`, `presentationDelay` -- tells you if the problem is a blocked main thread, slow handler, or expensive re-render
- **CLS:** `largestShiftTarget`, `largestShiftValue` -- tells you exactly which element shifted and by how much

**SPA navigation handling:**

Soft navigation measurement is still evolving (Chrome's Soft Navigation Heuristics API is experimental). Currently, the web-vitals library reports metrics for the full page session:

- **LCP** finalizes on first user interaction and is only reported for the initial page load
- **INP** accumulates across the entire page session (including SPA navigations) -- one value per page
- **CLS** accumulates across the entire session

For SPA-specific tracking, you can use `PerformanceObserver` directly:

```typescript
// Track CLS per SPA route change
let currentCLS = 0;

interface LayoutShiftEntry extends PerformanceEntry {
  hadRecentInput: boolean;
  value: number;
}

const observer = new PerformanceObserver((list) => {
  for (const entry of list.getEntries() as LayoutShiftEntry[]) {
    if (!entry.hadRecentInput) {
      currentCLS += entry.value;
    }
  }
});

observer.observe({ type: "layout-shift", buffered: true });

// On SPA navigation, report and reset
function onRouteChange(): void {
  navigator.sendBeacon("/api/vitals", JSON.stringify({
    name: "CLS",
    value: currentCLS,
    url: location.href,
  }));
  currentCLS = 0;
}
```

**Sending strategy:** Use `navigator.sendBeacon()` over `fetch()` because sendBeacon survives page unloads -- critical since CLS and INP report their final values as the user leaves.

</details>

<details>
<summary>14. A page's LCP is 4.5 seconds in the field but 2.1 seconds in Lighthouse — walk through the systematic diagnosis process: how you identify the LCP element, determine whether the bottleneck is server response time, render-blocking resources, resource load time, or client-side rendering delay, and what specific fixes you apply for each root cause. Show the DevTools workflow and the code/config changes.</summary>

**Step 1: Understand the lab vs field gap**

The 2.4s gap suggests real-user conditions differ significantly from Lighthouse. Common causes: slower devices, slower networks, CDN cache misses, third-party script impact, or a different LCP element in the field.

**Step 2: Identify the LCP element**

Use the web-vitals attribution data you collect in production (as covered in question 13):

```typescript
// Your RUM data tells you:
// attribution.target = "img.hero-product-image"
// attribution.url = "https://cdn.example.com/products/hero.jpg"
```

In DevTools: Performance panel > record a page load > look for the "LCP" marker in the timings lane. Hover it to see which element was identified.

**Step 3: Break down the LCP time**

LCP = TTFB + Resource Load Delay + Resource Load Duration + Element Render Delay

Your RUM attribution data gives these breakdowns directly. Diagnose the largest contributor:

**If TTFB is the bottleneck (> 800ms):**

- Check if slow TTFB only affects certain geographies (CDN coverage issue)
- Check server response time: is the origin slow, or is it a CDN cache miss?
- Fixes:
  - Ensure CDN caching for the HTML document (`s-maxage` with `stale-while-revalidate`)
  - Use `103 Early Hints` to start preloading while the server computes the response
  - Optimize server-side rendering time (database queries, API calls)

**If Resource Load Delay is the bottleneck:**

This is the time between TTFB and when the browser starts loading the LCP resource. It means the browser discovered the image late.

- Common cause: the image URL is in CSS (`background-image`) or set via JavaScript -- the preload scanner cannot find it
- Fixes:
  ```html
  <!-- Preload the LCP image so it starts loading immediately -->
  <link rel="preload" as="image" href="/hero.jpg" fetchpriority="high" />

  <!-- Or use an <img> tag instead of CSS background -->
  <img src="/hero.jpg" alt="Hero" fetchpriority="high" />
  ```

**If Resource Load Duration is the bottleneck:**

The image itself is slow to download.

- Check image file size: is it unoptimized? (common in field where users are on 3G)
- Fixes:
  ```html
  <!-- Serve optimized formats with responsive sizes -->
  <picture>
    <source srcset="/hero.avif" type="image/avif" />
    <source srcset="/hero.webp" type="image/webp" />
    <img src="/hero.jpg" alt="Hero" fetchpriority="high" width="1200" height="600" />
  </picture>
  ```
  - Ensure the image is on the same CDN origin (avoid cross-origin connection overhead)
  - Use `preconnect` if the image is on a different domain

**If Element Render Delay is the bottleneck:**

The image has loaded but rendering is delayed. This usually means render-blocking JS or CSS is preventing the browser from painting.

- Check for render-blocking scripts in `<head>`
- Fixes:
  ```html
  <!-- Move non-critical JS to defer -->
  <script src="/analytics.js" defer></script>

  <!-- Inline critical CSS, async-load the rest -->
  <style>/* critical above-the-fold styles */</style>
  <link rel="stylesheet" href="/full.css" media="print" onload="this.media='all'" />
  ```

**Step 4: Explain the lab vs field gap specifically**

After fixing, verify the gap closes. Common reasons for the 2.4s difference:
- Lighthouse tests on simulated 4G with a mid-tier CPU; real users on 3G with low-end phones are 2-3x slower
- Lighthouse does not execute third-party scripts that block the main thread in production
- CDN cache is warm for Lighthouse (testing from one location) but cold for users in far regions

</details>

<details>
<summary>15. A page has a CLS score of 0.35 in the field — walk through how you diagnose which elements are shifting and why: how you use the Layout Shift attribution in DevTools and the web-vitals library to find the culprits, and show the specific fixes for the most common causes (images without dimensions, late-loading fonts, dynamically injected content, ads).</summary>

A CLS of 0.35 is well above the 0.1 "good" threshold. Diagnosis starts with finding which elements are shifting.

**Step 1: Identify shifting elements from RUM data**

Your web-vitals attribution data (question 13) reports `largestShiftTarget` -- the CSS selector of the element that contributed most to CLS. Aggregate this across sessions to find the top culprits.

**Step 2: Reproduce in DevTools**

1. Open Performance panel > check "Screenshots" and "Web Vitals" lanes
2. Record a page load (or SPA navigation if CLS happens after load)
3. Look for red "Layout Shift" markers in the Experience lane
4. Click each shift to see which elements moved, the shift score, and whether it had recent input (user-triggered shifts are excluded from CLS)

Alternatively, use the Layout Shift Regions overlay:
```javascript
// Paste in console to visualize all layout shifts
new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    for (const source of (entry as any).sources || []) {
      const el = source.node;
      if (el) {
        el.style.outline = "3px solid red";
        console.log("Shifted:", el, "by", (entry as any).value);
      }
    }
  }
}).observe({ type: "layout-shift", buffered: true });
```

**Fix 1: Images without dimensions**

When an `<img>` has no `width`/`height`, the browser allocates 0px until the image loads, then shifts content down.

```html
<!-- BAD: no dimensions, causes layout shift -->
<img src="/product.jpg" alt="Product" />

<!-- GOOD: explicit dimensions reserve space -->
<img src="/product.jpg" alt="Product" width="400" height="300" />

<!-- GOOD: CSS aspect-ratio (modern approach) -->
<style>
  .product-img {
    width: 100%;
    aspect-ratio: 4 / 3;
    object-fit: cover;
  }
</style>
<img src="/product.jpg" alt="Product" class="product-img" />
```

**Fix 2: Late-loading fonts (FOUT - Flash of Unstyled Text)**

Custom fonts load after CSS. The fallback font renders first with different metrics, then the page shifts when the custom font swaps in.

```css
/* Use font-display: optional to eliminate shift entirely */
@font-face {
  font-family: "CustomFont";
  src: url("/fonts/custom.woff2") format("woff2");
  font-display: optional; /* uses fallback if font isn't cached; no swap, no shift */
}

/* Or use size-adjust to match fallback metrics */
@font-face {
  font-family: "CustomFont Fallback";
  src: local("Arial");
  size-adjust: 105%;
  ascent-override: 90%;
  descent-override: 20%;
  line-gap-override: 0%;
}

body {
  font-family: "CustomFont", "CustomFont Fallback", Arial, sans-serif;
}
```

Also preload the font to reduce the swap window:
```html
<link rel="preload" href="/fonts/custom.woff2" as="font" type="font/woff2" crossorigin />
```

**Fix 3: Dynamically injected content**

Banners, cookie consent, or notifications that push content down after load.

```css
/* Reserve space for known dynamic content */
.cookie-banner-slot {
  min-height: 80px; /* matches banner height */
}

/* Or use transform instead of layout-affecting properties */
.notification {
  /* BAD: inserts into flow, shifts everything below */
  /* position: static; */

  /* GOOD: overlays without shifting */
  position: fixed;
  bottom: 0;
  transform: translateY(100%);
  transition: transform 0.3s;
}
.notification.visible {
  transform: translateY(0);
}
```

**Fix 4: Ads and iframes**

Ads are the most common CLS cause in the field. The ad loads late, expands to its actual size, and shifts everything.

```html
<!-- Reserve space with a container -->
<div style="min-height: 250px; min-width: 300px;">
  <!-- Ad loads inside reserved space -->
  <div id="ad-slot-1"></div>
</div>
```

```css
/* For responsive ads, use aspect-ratio */
.ad-container {
  width: 100%;
  aspect-ratio: 300 / 250;
  contain: layout; /* CSS containment prevents child changes from affecting siblings */
}
```

**Fix 5: Use `content-visibility` carefully**

`content-visibility: auto` can cause shifts if the browser's estimated size does not match the actual rendered size. Always pair it with `contain-intrinsic-size`:

```css
.long-list-item {
  content-visibility: auto;
  contain-intrinsic-size: 0 200px; /* estimated height */
}
```

</details>

<details>
<summary>16. A page has poor INP (400ms) in the field — walk through how you identify which interactions are slow using the web-vitals library with attribution, how you find the responsible long tasks in a DevTools performance trace, and what specific fixes you apply for the most common causes (heavy event handlers, layout thrashing during interaction, third-party scripts blocking the main thread).</summary>

**Step 1: Identify the slow interaction from RUM**

Your web-vitals INP attribution data tells you exactly what is slow:

```
interactionTarget: "button.add-to-cart"
interactionType: "pointer"
inputDelay: 250ms        // event waited in queue
processingDuration: 100ms // handler execution
presentationDelay: 50ms   // layout + paint after handler
```

The 400ms breaks down as: 250ms input delay + 100ms processing + 50ms presentation. Input delay dominates -- something is blocking the main thread when the user clicks.

**Step 2: Reproduce in DevTools**

1. Performance panel > Start recording > Perform the interaction > Stop
2. Find the click event in the "Interactions" lane
3. The gap between the interaction marker and the event handler start is the input delay
4. Look at what is running on the main thread during that gap -- this is the blocking work
5. Long tasks are highlighted with red corners in the main thread flame chart

**Step 3: Fix based on root cause**

**Root cause A: Third-party scripts blocking the main thread (most common for high input delay)**

A third-party analytics or chat widget is executing a long task when the user clicks.

```html
<!-- BAD: synchronous third-party script blocks main thread -->
<script src="https://analytics.example.com/tracker.js"></script>

<!-- GOOD: defer or async so it doesn't block -->
<script src="https://analytics.example.com/tracker.js" async></script>

<!-- BETTER: load after page is interactive -->
<script>
  // Load third-party scripts only after the page is idle
  if ("requestIdleCallback" in window) {
    requestIdleCallback(() => {
      const script = document.createElement("script");
      script.src = "https://analytics.example.com/tracker.js";
      document.body.appendChild(script);
    });
  }
</script>
```

For third-party scripts you cannot control, consider loading them in a web worker using Partytown:
```html
<script type="text/partytown" src="https://analytics.example.com/tracker.js"></script>
```

**Root cause B: Heavy event handler (high processing duration)**

The click handler does too much synchronous work.

```typescript
// BAD: heavy synchronous work in handler
function handleAddToCart(product: Product): void {
  const recommendations = computeRecommendations(catalog, product); // 150ms
  updateCartState(product);
  renderRecommendations(recommendations);
}

// GOOD: defer non-critical work
async function handleAddToCart(product: Product): void {
  // Critical: update cart immediately (fast, < 50ms)
  updateCartState(product);

  // Yield to let the browser paint the cart update
  await scheduler.yield();

  // Non-critical: compute recommendations after paint
  const recommendations = computeRecommendations(catalog, product);
  renderRecommendations(recommendations);
}
```

In React, use `startTransition` to mark non-urgent updates:

```tsx
import { startTransition } from "react";

function handleAddToCart(product: Product): void {
  // Urgent: cart count updates immediately
  setCartItems((prev) => [...prev, product]);

  // Non-urgent: recommendations can be deferred
  startTransition(() => {
    setRecommendations(computeRecommendations(product));
  });
}
```

**Root cause C: Layout thrashing during interaction (high presentation delay)**

The handler reads and writes layout properties alternately, forcing synchronous layouts. Fix using the batch-reads-then-writes pattern from Q2 -- separate all DOM reads from all DOM writes to avoid forcing the browser to recalculate layout on every iteration.

**Step 4: Verify the fix**

After deploying, monitor the INP attribution breakdown in your RUM dashboard. The metric should show improvement within a few days as field data accumulates. Check that the specific interaction (`button.add-to-cart`) shows reduced input delay and processing time.

</details>

## Practical — Rendering & Caching Configuration

<details>
<summary>17. Configure the loading behavior of scripts on a page that has a critical inline script, a large framework bundle, a non-critical analytics script, and a module-based component — show how you use async, defer, type="module", and the preload scanner to control execution order, explain what render-blocking means for each configuration, and what breaks if you get the loading order wrong.</summary>

```html
<!DOCTYPE html>
<html>
<head>
  <!-- 1. Critical inline script: executes immediately, blocks parsing -->
  <!-- Use for things that MUST run before any content renders (theme, feature flags) -->
  <script>
    // Runs synchronously during HTML parsing -- blocks DOM construction
    document.documentElement.dataset.theme =
      localStorage.getItem("theme") || "light";
  </script>

  <!-- 2. Preload the framework bundle so the preload scanner discovers it early -->
  <link rel="preload" href="/js/framework.js" as="script" />

  <!-- 3. Framework bundle with defer: downloads in parallel, executes after DOM is ready -->
  <script src="/js/framework.js" defer></script>

  <!-- 4. Module-based component: type="module" is deferred by default -->
  <script type="module" src="/js/interactive-widget.js"></script>

  <!-- 5. Analytics: async -- downloads in parallel, executes as soon as available -->
  <script src="/js/analytics.js" async></script>
</head>
<body>
  <!-- content -->
</body>
</html>
```

**How each loading strategy works:**

| Script | Download | Execution | Render-blocking? |
|--------|----------|-----------|-----------------|
| **Inline `<script>`** | N/A (already in HTML) | Immediately, synchronously | Yes -- blocks parsing and rendering |
| **`defer`** | Parallel with parsing | After DOM is complete, before `DOMContentLoaded`, in document order | No |
| **`async`** | Parallel with parsing | As soon as downloaded (interrupts parsing) | No for download; Yes briefly during execution |
| **`type="module"`** | Parallel with parsing | After DOM is complete (like defer), in dependency order | No (modules are deferred by default) |
| **Plain `<script src>`** | Blocks parsing until downloaded | Immediately after download | Yes -- fully render-blocking |

**Execution order guarantee:**

```
1. Inline script (synchronous, first in document)
2. defer scripts in document order (framework.js first, guaranteed)
3. type="module" scripts after defer scripts resolve their imports
4. async scripts whenever they finish downloading (NO order guarantee)
```

**The preload scanner's role:** While the inline script blocks the parser, the preload scanner looks ahead and starts downloading `framework.js`, `interactive-widget.js`, and `analytics.js` in parallel. Without the preload scanner (or the explicit `<link rel="preload">`), these downloads would not start until the parser reaches them.

**What breaks with wrong loading order:**

- **Framework as `async` instead of `defer`:** If the framework loads before the app code, it works. If the app code loads first and the framework is not ready, you get `React is not defined` errors. `defer` guarantees document order; `async` does not.
- **Analytics as `defer` instead of `async`:** Analytics would delay `DOMContentLoaded` event and any scripts waiting for it. Since analytics does not depend on DOM or other scripts, `async` is correct -- let it run whenever.
- **Module without `type="module"`:** It would execute as a classic script. `import`/`export` syntax would throw a syntax error.
- **Critical inline script at end of `<body>`:** The theme would apply after content renders, causing a flash of wrong-themed content.

</details>

<details>
<summary>18. Design the HTTP caching strategy for a production frontend app that serves hashed JS/CSS bundles, an index.html entry point, API responses, and static images through a CDN — show the Cache-Control headers for each resource type, explain why index.html must never be cached aggressively, how you handle cache busting on deployment, and what happens when the CDN serves a stale index.html pointing to deleted hashed bundles.</summary>

**Caching strategy per resource type:**

```
# Hashed JS/CSS bundles (e.g., main.a1b2c3.js, styles.d4e5f6.css)
Cache-Control: public, max-age=31536000, immutable

# index.html (entry point)
Cache-Control: no-cache
# (or: Cache-Control: public, max-age=0, must-revalidate)
# CDN layer: surrogate-control: max-age=60, stale-while-revalidate=30

# API responses (dynamic data)
Cache-Control: private, no-cache
# Or for semi-stable data: private, max-age=60, stale-while-revalidate=300

# Static images (non-hashed, e.g., /images/logo.png)
Cache-Control: public, max-age=86400
ETag: "hash-of-content"

# Fonts
Cache-Control: public, max-age=31536000, immutable
```

**Why index.html must never be aggressively cached:**

`index.html` is the only resource whose URL never changes between deployments. It contains `<script src="/js/main.a1b2c3.js">` -- the hash references. If a user's browser caches `index.html` with `max-age=86400`, they will keep requesting the old hashed bundles for up to 24 hours after you deploy. During that window:

- If you keep old bundles on the server (most CDN setups do temporarily), users get stale app versions
- If you delete old bundles immediately, users get 404 errors on the JS/CSS files -- a blank page

`no-cache` means the browser revalidates with the server on every request. With ETags, this is just a `304 Not Modified` (no body transferred) when nothing changed -- negligible overhead.

**Cache busting on deployment:**

```
Before deploy:
  index.html -> references main.abc123.js
  CDN has: main.abc123.js (cached 1 year)

After deploy:
  index.html -> references main.def456.js (new hash)
  CDN has: main.abc123.js (still cached) + main.def456.js (new)
```

The flow:
1. Deploy new hashed bundles to CDN/origin
2. Update `index.html` to reference new hashes
3. Purge CDN cache for `index.html` (or rely on short TTL / `no-cache`)
4. Users fetch fresh `index.html` -> see new bundle URLs -> CDN cache miss for new bundles -> fetched from origin -> cached at CDN for 1 year

**The stale index.html problem:**

If the CDN serves a stale `index.html` that references `main.abc123.js`, but that bundle was already deleted from the origin:

1. Browser gets stale `index.html` from CDN
2. Browser requests `main.abc123.js`
3. CDN cache miss -> forwards to origin -> 404
4. User sees a blank page or broken app

**Prevention strategies:**

```nginx
# Nginx: keep old bundles for at least 24 hours after deploy
# Use a deploy script that:
# 1. Uploads new bundles (don't delete old ones yet)
# 2. Updates index.html
# 3. Purges CDN cache for index.html
# 4. Scheduled job deletes bundles older than 24h
```

```typescript
// CDN configuration (e.g., CloudFront, Fastly)
// Invalidate /index.html on every deployment
// Keep origin storage with old + new bundles

// deploy.ts
async function deploy(): Promise<void> {
  await uploadNewBundles();        // new hashed files to S3
  await updateIndexHtml();         // point to new hashes
  await invalidateCDN(["/index.html", "/"]); // purge HTML only
  // DO NOT delete old bundles yet
  await scheduleCleanup({ olderThan: "24h" });
}
```

The key principle: **immutable content-hashed URLs for assets + always-revalidate for the HTML entry point + keep old bundles alive during the transition window.**

</details>

<details>
<summary>19. Set up streaming SSR with selective hydration in a React application — show the server-side rendering code using renderToPipeableStream, how Suspense boundaries control the streaming and hydration order, and explain how this approach improves Time to First Byte and INP compared to traditional SSR where the entire page must render before any HTML is sent.</summary>

**Traditional SSR problem:** `renderToString` must render the entire component tree to a string before sending any bytes. If your page fetches data from 3 APIs (50ms, 200ms, 800ms), the user sees nothing for 800ms+. TTFB equals the slowest data fetch.

**Streaming SSR with `renderToPipeableStream`:**

```tsx
// server.tsx
import { renderToPipeableStream } from "react-dom/server";
import express from "express";
import { App } from "./App";

const app = express();

app.get("*", (req, res) => {
  const { pipe, abort } = renderToPipeableStream(
    <App url={req.url} />,
    {
      bootstrapScripts: ["/js/client.js"],

      onShellReady() {
        // The shell (everything outside Suspense boundaries) is ready
        // Start streaming immediately -- don't wait for suspended content
        res.statusCode = 200;
        res.setHeader("Content-Type", "text/html");
        pipe(res);
      },

      onShellError(error) {
        // Shell itself failed -- fall back to client-side rendering
        res.statusCode = 500;
        res.send("<!DOCTYPE html><html><body><div id='root'></div></body></html>");
      },

      onError(error) {
        // Non-shell errors (inside Suspense boundaries)
        console.error("Streaming error:", error);
      },
    }
  );

  // Abort if rendering takes too long
  setTimeout(() => abort(), 10_000);
});
```

**App structure with Suspense boundaries:**

```tsx
// App.tsx
import { Suspense } from "react";

function App({ url }: { url: string }) {
  return (
    <html>
      <head>
        <title>My App</title>
        <link rel="stylesheet" href="/css/main.css" />
      </head>
      <body>
        {/* Shell: renders immediately, streamed in onShellReady */}
        <Header />
        <HeroSection />

        {/* Suspense boundary 1: streams when product data resolves */}
        <Suspense fallback={<ProductSkeleton />}>
          <ProductDetails id={extractId(url)} />
        </Suspense>

        {/* Suspense boundary 2: streams when comments load (slowest) */}
        <Suspense fallback={<CommentsSkeleton />}>
          <Comments id={extractId(url)} />
        </Suspense>

        <Footer />
      </body>
    </html>
  );
}
```

**Client-side hydration with selective hydration:**

```tsx
// client.tsx
import { hydrateRoot } from "react-dom/client";
import { App } from "./App";

hydrateRoot(document, <App url={window.location.pathname} />);
```

**How streaming works step by step:**

1. Browser receives the shell (Header, HeroSection, skeletons for ProductDetails and Comments, Footer) almost immediately -- TTFB is fast because it does not wait for data
2. The browser renders the shell and starts downloading `client.js`
3. As ProductDetails data resolves (200ms), the server streams an HTML chunk with the product content plus an inline `<script>` that tells React to swap the skeleton
4. As Comments data resolves (800ms), the same happens for that Suspense boundary
5. Hydration begins as chunks arrive -- React hydrates each boundary independently

**Selective hydration:** If the user clicks on the ProductDetails section before Comments has streamed, React prioritizes hydrating ProductDetails first, even if Comments HTML arrived earlier. This directly improves INP because the component the user is interacting with becomes interactive first.

**Performance comparison:**

| Metric | Traditional SSR (`renderToString`) | Streaming SSR (`renderToPipeableStream`) |
|--------|-----------------------------------|----------------------------------------|
| **TTFB** | Blocked by slowest data fetch (800ms+) | Shell streams immediately (~50ms) |
| **FCP/LCP** | All content appears at once after full render | Shell content appears immediately; rest streams in |
| **INP** | All components hydrate at once (one big task) | Selective hydration prioritizes user-interacted components |
| **Server memory** | Entire HTML string buffered in memory | Streamed in chunks, lower peak memory |

**Gotcha:** The shell (everything outside Suspense boundaries) still blocks TTFB. Keep the shell lightweight -- no data fetching, no heavy computation. Move all async work inside Suspense boundaries.

</details>

## Practical — Asset Optimization & Delivery

<details>
<summary>20. Configure code splitting, vendor chunking, and tree shaking in a Vite or Webpack build for a large React application — show the configuration for splitting vendor libraries into a stable long-cached chunk, route-based dynamic imports with React.lazy, and how you use a bundle analyzer to find unexpectedly large modules. Explain what configuration mistakes silently break tree shaking.</summary>

**Vite configuration (Rollup under the hood):**

```typescript
// vite.config.ts
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";
import { visualizer } from "rollup-plugin-visualizer";

export default defineConfig({
  plugins: [
    react(),
    visualizer({
      filename: "bundle-analysis.html",
      open: true, // opens after build
      gzipSize: true,
    }),
  ],
  build: {
    rollupOptions: {
      output: {
        manualChunks: {
          // Vendor chunk: stable across deployments, cached long-term
          vendor: ["react", "react-dom", "react-router-dom"],
          // Separate large libraries that not every page needs
          charts: ["recharts", "d3-scale", "d3-shape"],
        },
      },
    },
    // Target modern browsers for smaller output
    target: "es2020",
    // Enable source maps for debugging (but not in production serving)
    sourcemap: true,
  },
});
```

**Route-based code splitting with React.lazy:**

```tsx
// routes.tsx
import { lazy, Suspense } from "react";
import { Routes, Route } from "react-router-dom";

// Each route is a separate chunk -- only downloaded when navigated to
const Dashboard = lazy(() => import("./pages/Dashboard"));
const ProductList = lazy(() => import("./pages/ProductList"));
const ProductDetail = lazy(() => import("./pages/ProductDetail"));
const Settings = lazy(() => import("./pages/Settings"));

function AppRoutes() {
  return (
    <Suspense fallback={<PageSkeleton />}>
      <Routes>
        <Route path="/" element={<Dashboard />} />
        <Route path="/products" element={<ProductList />} />
        <Route path="/products/:id" element={<ProductDetail />} />
        <Route path="/settings" element={<Settings />} />
      </Routes>
    </Suspense>
  );
}
```

**Prefetching likely next routes:**

```tsx
// Prefetch on hover/focus for instant navigation
function NavLink({ to, children }: { to: string; children: React.ReactNode }) {
  const prefetch = () => {
    // Vite resolves dynamic imports to chunk URLs
    // Hovering triggers the browser to fetch the chunk
    switch (to) {
      case "/products": import("./pages/ProductList"); break;
      case "/settings": import("./pages/Settings"); break;
    }
  };

  return (
    <Link to={to} onMouseEnter={prefetch} onFocus={prefetch}>
      {children}
    </Link>
  );
}
```

**Using the bundle analyzer:**

After running `vite build`, the visualizer plugin generates an interactive treemap (`bundle-analysis.html`) showing:
- Size of each module (raw, gzipped)
- Which chunk each module belongs to
- Dependency relationships

Common findings and fixes:
- **`moment.js` (300KB+ with locales):** Replace with `date-fns` or `dayjs` (tree-shakeable)
- **Full `lodash` import:** Use `lodash-es` for tree shaking, or import specific functions: `import debounce from "lodash-es/debounce"`
- **Duplicate dependencies:** Two versions of the same library bundled (e.g., two React versions). Fix with `resolve.dedupe` in Vite config.

**What silently breaks tree shaking:**

1. **CommonJS imports** -- `require()` is dynamic; bundlers cannot determine what is used. Always use ESM (`import`/`export`). Check that dependencies ship ESM builds.

2. **Missing `sideEffects` in `package.json`** -- Without `"sideEffects": false`, the bundler assumes every module has side effects and cannot remove unused exports:
   ```json
   {
     "name": "my-library",
     "sideEffects": false
   }
   // Or specify which files DO have side effects:
   {
     "sideEffects": ["*.css", "./src/polyfills.ts"]
   }
   ```

3. **Barrel files with side effects** -- A barrel `index.ts` that re-exports 50 modules forces the bundler to evaluate all of them. If any module has side effects at the top level, nothing gets shaken out.

4. **Class-based code** -- Some bundlers struggle to tree-shake classes because method calls might have side effects. Function-based code tree-shakes more reliably.

5. **`eval()` or dynamic property access** -- `obj[dynamicKey]` prevents the bundler from knowing which properties are used, defeating dead code elimination.

</details>

<details>
<summary>21. Set up responsive images with srcset and sizes for a content-heavy page — show the HTML for an image that serves different resolutions at different viewport widths, configure lazy loading for below-the-fold images while ensuring the LCP image loads eagerly, implement WebP/AVIF with fallbacks using the picture element, and explain why getting the sizes attribute wrong defeats the purpose of srcset.</summary>

**Full responsive image setup with format negotiation:**

```html
<!-- LCP hero image: eager load, high priority, AVIF/WebP with fallback -->
<picture>
  <source
    srcset="hero-400.avif 400w, hero-800.avif 800w, hero-1200.avif 1200w, hero-1600.avif 1600w"
    sizes="(max-width: 768px) 100vw, (max-width: 1200px) 75vw, 1200px"
    type="image/avif"
  />
  <source
    srcset="hero-400.webp 400w, hero-800.webp 800w, hero-1200.webp 1200w, hero-1600.webp 1600w"
    sizes="(max-width: 768px) 100vw, (max-width: 1200px) 75vw, 1200px"
    type="image/webp"
  />
  <img
    srcset="hero-400.jpg 400w, hero-800.jpg 800w, hero-1200.jpg 1200w, hero-1600.jpg 1600w"
    sizes="(max-width: 768px) 100vw, (max-width: 1200px) 75vw, 1200px"
    src="hero-800.jpg"
    alt="Hero banner"
    width="1200"
    height="600"
    fetchpriority="high"
  />
</picture>

<!-- Below-the-fold product images: lazy loaded -->
<picture>
  <source
    srcset="product-300.avif 300w, product-600.avif 600w, product-900.avif 900w"
    sizes="(max-width: 768px) 50vw, 300px"
    type="image/avif"
  />
  <source
    srcset="product-300.webp 300w, product-600.webp 600w, product-900.webp 900w"
    sizes="(max-width: 768px) 50vw, 300px"
    type="image/webp"
  />
  <img
    srcset="product-300.jpg 300w, product-600.jpg 600w, product-900.jpg 900w"
    sizes="(max-width: 768px) 50vw, 300px"
    src="product-600.jpg"
    alt="Product thumbnail"
    width="300"
    height="300"
    loading="lazy"
    decoding="async"
  />
</picture>
```

**Key attributes explained:**

- **`fetchpriority="high"`** on the LCP image tells the browser to prioritize this resource over other images. Do not use on below-the-fold images.
- **`loading="lazy"`** defers loading until the image is near the viewport. Never use on the LCP image (as covered in question 10).
- **`decoding="async"`** allows the browser to decode the image off the main thread, avoiding frame drops. Safe for all non-LCP images.
- **`width`/`height`** attributes reserve space to prevent CLS (as covered in question 15).

**How `srcset` and `sizes` work together:**

As explained in Q10, the browser uses `sizes` to determine the display width, multiplies by the device pixel ratio, and picks the smallest source from `srcset` that covers the needed pixels. If `sizes` is missing or wrong, the browser defaults to `100vw` and downloads a much larger image than necessary -- defeating the entire purpose of `srcset`.

Common mistake: copying `sizes="100vw"` everywhere. If your product grid shows images at `25vw` on desktop, the `sizes` attribute must reflect that:

```html
<!-- BAD: sizes says 100vw but image displays at 25% width on desktop -->
<img srcset="..." sizes="100vw" />

<!-- GOOD: matches actual layout -->
<img srcset="..." sizes="(max-width: 768px) 50vw, 25vw" />
```

**Practical recommendation:** Generate 3-4 `srcset` breakpoints per image (400w, 800w, 1200w, 1600w covers most cases). More granularity has diminishing returns. Use a build tool or image CDN (Cloudinary, imgix) to automate variant generation rather than manually creating each size.

</details>

<details>
<summary>22. A page loads slowly on 3G connections despite good Lighthouse scores on fast networks — walk through the diagnosis: how you use resource hints (preconnect to critical origins, dns-prefetch for third parties), extract and inline critical CSS to eliminate render-blocking stylesheets, set up preload for key resources discovered late in the waterfall, and configure the server to support early hints (103). Show the HTML and server configuration changes.</summary>

On fast networks, round-trip latency is low and bandwidth is high, so even unoptimized resource loading chains complete quickly. On 3G (~400ms RTT, ~1.6 Mbps), every unnecessary round trip and render-blocking resource becomes painfully visible.

**Step 1: Profile on a realistic slow connection**

In DevTools, use the Network panel with "Slow 3G" throttling and CPU 4x slowdown. Record a waterfall. Look for:
- **Long connection chains** -- DNS lookup + TCP + TLS per unique origin (each ~400ms on 3G)
- **Render-blocking resources** -- stylesheets and synchronous scripts that delay first paint
- **Late-discovered resources** -- fonts, images, or scripts that only start downloading after CSS or JS executes
- **Large resources on the critical path** -- uncompressed CSS/JS, unoptimized images

**Step 2: Eliminate connection overhead with resource hints**

Each new origin requires DNS + TCP + TLS (~600-1200ms on 3G). Preconnect to critical origins so this happens in parallel with HTML parsing:

```html
<head>
  <!-- Critical: font origin (used on every page) -->
  <link rel="preconnect" href="https://fonts.googleapis.com" />
  <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin />

  <!-- Critical: API origin (data fetch starts immediately) -->
  <link rel="preconnect" href="https://api.example.com" />

  <!-- Non-critical third parties: dns-prefetch only (cheaper) -->
  <link rel="dns-prefetch" href="https://analytics.example.com" />
  <link rel="dns-prefetch" href="https://cdn.chat-widget.com" />
</head>
```

Limit `preconnect` to 2-4 origins. Each open connection consumes memory and competes for bandwidth. Use `dns-prefetch` for the rest -- it is cheap (just a DNS query).

**Step 3: Inline critical CSS and async-load the rest**

On 3G, a render-blocking 50KB stylesheet takes ~250ms to download after the connection is established. Inlining the critical above-the-fold CSS eliminates that round trip entirely:

```html
<head>
  <!-- Critical CSS inlined: no extra round trip, renders immediately -->
  <style>
    /* Only above-the-fold styles (~10-15KB max) */
    :root { --primary: #1a73e8; }
    body { margin: 0; font-family: system-ui, sans-serif; }
    .header { display: flex; align-items: center; height: 64px; padding: 0 1rem; }
    .hero { padding: 2rem; font-size: 1.5rem; }
    .skeleton { background: #e0e0e0; border-radius: 4px; animation: pulse 1.5s infinite; }
    @keyframes pulse { 50% { opacity: 0.5; } }
  </style>

  <!-- Full stylesheet loaded asynchronously -->
  <link rel="stylesheet" href="/css/main.css" media="print" onload="this.media='all'" />
  <noscript><link rel="stylesheet" href="/css/main.css" /></noscript>
</head>
```

To extract critical CSS automatically at build time, use tools like `critters` (Vite/Webpack plugin) that analyze your HTML and inline only the CSS rules needed for above-the-fold content.

**Step 4: Preload late-discovered resources**

Resources the browser discovers late in the waterfall (fonts referenced in CSS, images loaded via JS, async chunk dependencies) need explicit preload hints:

```html
<head>
  <!-- Font: normally discovered only after CSS parses (~1200ms into load on 3G) -->
  <link rel="preload" href="/fonts/inter-var.woff2" as="font" type="font/woff2" crossorigin />

  <!-- LCP image: if set via CSS background-image, preload scanner can't find it -->
  <link rel="preload" href="/images/hero.avif" as="image" type="image/avif" fetchpriority="high" />

  <!-- Critical async chunk: React.lazy component needed on this route -->
  <link rel="preload" href="/js/product-page.abc123.js" as="script" />
</head>
```

**Step 5: Configure Early Hints (103)**

`103 Early Hints` lets the server send preload/preconnect hints while it is still computing the response. On 3G, if the server takes 200ms to render SSR, that is 200ms of wasted time the browser could use to start fetching resources.

**Nginx configuration:**

```nginx
server {
    listen 443 ssl http2;

    location / {
        # Send 103 Early Hints before the proxy response
        add_header Link "</css/main.css>; rel=preload; as=style" early;
        add_header Link "</js/framework.js>; rel=preload; as=script" early;
        add_header Link "<https://fonts.gstatic.com>; rel=preconnect; crossorigin" early;

        proxy_pass http://app_server;
    }
}
```

**Node.js (Express) with HTTP/2:**

```typescript
import express from "express";
import http2 from "http2";

app.get("*", (req, res) => {
  // Send 103 Early Hints immediately
  if (res.writeEarlyHints) {
    res.writeEarlyHints({
      link: [
        '</css/main.css>; rel=preload; as=style',
        '</fonts/inter-var.woff2>; rel=preload; as=font; crossorigin',
        '<https://api.example.com>; rel=preconnect',
      ],
    });
  }

  // Then compute the full response (SSR, data fetching, etc.)
  const html = await renderPage(req.url);
  res.status(200).send(html);
});
```

**CDN support:** Cloudflare supports Early Hints by caching `Link` headers from your origin and replaying them as `103` responses from the edge -- the browser starts preloading before the request even reaches your origin.

**Combined impact on 3G:**

| Optimization | Time saved on 3G |
|-------------|-----------------|
| `preconnect` to font origin | ~600-800ms (skip DNS+TCP+TLS) |
| Critical CSS inlining | ~400-600ms (eliminate render-blocking round trip) |
| `preload` LCP image | ~400-800ms (start download earlier) |
| `preload` font | ~600-1000ms (discovered in `<head>` instead of after CSS) |
| Early hints (103) | ~100-300ms (use server think time for preloading) |

These optimizations compound. On a fast network, each saves 20-50ms and the total improvement is barely noticeable. On 3G, the same changes can cut 2-3 seconds off the load time.

</details>

---

## Experience-Based Questions

These questions test real-world experience. Prepare by mapping them to your own projects and situations.

<details>
<summary>23. Tell me about a time you diagnosed and fixed a Core Web Vitals regression in production — what metric regressed, how did you discover it, what was the root cause, and what was the fix? How did you prevent similar regressions going forward?</summary>

**What the interviewer is looking for:**
- You have production performance monitoring in place (not just lab testing)
- You can systematically diagnose a regression to its root cause using data
- You understand the connection between code changes and metric impact
- You think about prevention, not just fix-and-move-on

**Suggested structure (STAR):**

1. **Situation:** What was the app, what monitoring was in place, what was the baseline
2. **Task:** Which metric regressed, how you discovered it (alert, dashboard, CrUX report, PageSpeed Insights)
3. **Action:** Step-by-step diagnosis -- correlating the regression with a deploy, identifying the specific change, measuring the fix
4. **Result:** Metric improvement (with numbers), prevention measures added

**Example outline to personalize:**

- "Our e-commerce product pages had stable LCP around 2.2s. After a sprint focused on redesign, our RUM dashboard (web-vitals + custom analytics) showed LCP spiking to 3.8s at the p75."
- "I correlated the timing with our deployment log -- the regression started with a deploy that changed the hero image from a static `<img>` tag to a dynamically loaded carousel component. The LCP element changed from a server-rendered image (discovered by the preload scanner) to a JS-rendered image (invisible until the carousel JS executed and fetched the first slide)."
- "The fix was two-fold: (1) server-render the first carousel slide so the LCP image appears in the initial HTML, and (2) add a `<link rel="preload">` for the first slide image. LCP dropped from 3.8s back to 2.0s."
- "For prevention, we added a Lighthouse CI check in our CI pipeline with a performance budget: LCP must stay below 2.5s on the product page. We also added a CrUX-based alert that fires when p75 LCP exceeds our threshold for 48 hours."

**Key points to hit:**
- Mention specific numbers (before/after metrics)
- Show you used field data (RUM), not just lab data, to discover the issue
- Demonstrate you traced the regression to a specific code change or deployment
- Prevention is critical -- CI performance budgets, RUM alerts, or performance review checklists show senior-level thinking

</details>

<details>
<summary>24. Describe a time you migrated a client-side rendered application to SSR or SSG — what drove the decision, what rendering strategy did you choose and why, what were the biggest technical challenges (hydration issues, caching, infrastructure), and what was the measurable performance impact?</summary>

**What the interviewer is looking for:**
- You understand the tradeoffs between CSR, SSR, and SSG deeply enough to justify a migration (not just "SSR is better")
- You can articulate the business case (SEO, conversion rates, LCP) that drove the decision
- You dealt with real implementation challenges, not a textbook migration
- You measured impact with real data

**Suggested structure:**

1. **Driver:** What business or technical problem made CSR insufficient (SEO traffic dropping, poor LCP on mobile, conversion rate tied to load speed)
2. **Decision:** Which rendering strategy you chose and why -- reference the decision factors from question 6 (SEO needs, content freshness, personalization, page count)
3. **Challenges:** Pick 2-3 real challenges and explain how you solved them
4. **Impact:** Quantified results

**Example outline to personalize:**

- "Our marketing pages were a React SPA. Google was indexing them, but with delays -- new pages took weeks to appear in search results, and our SEO team showed us competitor pages with instant indexing. LCP was 4.2s on mobile because the browser had to download React, execute it, then fetch CMS content before anything rendered."
- "We chose a per-route strategy using Next.js: SSG for marketing pages (content changes weekly, excellent cache performance), SSR for the pricing page (personalized by region), and kept CSR for the authenticated dashboard (no SEO need)."
- "Biggest challenges: (1) **Hydration mismatches** -- we had components using `window.innerWidth` during render for responsive logic. We refactored to use CSS media queries or `useEffect` for client-only values. (2) **Caching complexity** -- SSG pages needed CDN invalidation on CMS publish. We set up a webhook from the CMS that triggered Next.js ISR revalidation. (3) **Build times** -- 200+ marketing pages took 8 minutes to build. We moved to ISR with `revalidate: 3600` so pages regenerate on demand instead of all at build time."
- "Results: LCP dropped from 4.2s to 1.4s on marketing pages. Organic search traffic increased 35% over 3 months (faster indexing + better Core Web Vitals ranking signal). The pricing page TTFB increased slightly (SSR compute) but LCP still improved by 1.5s."

**Key points to hit:**
- Show the migration was driven by measurable business need, not trend-following
- Demonstrate you chose different strategies for different routes (shows nuance, references question 6)
- Hydration issues are expected -- mentioning them and how you solved them shows real experience (references question 7)
- Quantified before/after metrics for both performance and business outcomes

</details>

<details>
<summary>25. Tell me about a time you significantly reduced bundle size or improved loading performance for a large frontend application — what analysis did you do to find the biggest wins, what specific changes did you make, and how did you measure the impact in the field (not just in lab tools)?</summary>

**What the interviewer is looking for:**
- You use data-driven analysis (bundle analyzer, RUM) to find the highest-impact wins, not guesswork
- You understand the relationship between bundle size and real user metrics (not just "smaller is better")
- You can make pragmatic tradeoffs (effort vs impact)
- You validate with field data, not just lab benchmarks

**Suggested structure:**

1. **Discovery:** How you identified bundle size as the bottleneck (RUM data showing slow LCP/INP, bundle analysis revealing bloat)
2. **Analysis:** What tools you used and what you found
3. **Changes:** The specific, ordered list of optimizations (ordered by impact)
4. **Measurement:** Field data before/after, not just bundle size diff

**Example outline to personalize:**

- "Our SPA had grown to 1.8MB of JS (420KB gzipped). RUM showed INP of 350ms on mobile -- the main thread was blocked for 2+ seconds just parsing and executing the initial bundle. Lighthouse scored well on fast connections but field data on mid-range Android devices was terrible."
- "Analysis steps: (1) Ran `rollup-plugin-visualizer` and found `moment.js` at 230KB (with all locales), `lodash` at 70KB (imported as full CJS bundle), and an unused PDF library at 150KB still imported from a removed feature. (2) Checked our route structure -- every route was in the main bundle, no code splitting."
- "Changes in order of impact: (1) **Removed dead code** -- the PDF library import. Saved 150KB. (2) **Replaced moment with date-fns** -- tree-shakeable, only imported the 4 functions we used. Saved 210KB. (3) **Switched to lodash-es** with direct imports. Saved 55KB. (4) **Added route-based code splitting** with React.lazy for 8 routes. Initial bundle dropped from 420KB to 180KB gzipped. (5) **Split the charting library** into its own async chunk loaded only on the analytics page."
- "Results: Total JS dropped from 1.8MB to 680KB (initial route: 320KB). In the field, p75 INP improved from 350ms to 140ms on mobile. LCP improved by 800ms because the main thread unblocked faster, allowing the LCP image to render sooner. We tracked this through our web-vitals RUM pipeline over 2 weeks, filtering by device category to isolate the mobile improvement."

**Key points to hit:**
- Start with the bundle analyzer -- show you know how to find the actual culprits (references question 20)
- Order changes by impact (dead code removal is often the biggest free win)
- Distinguish between total bundle size and initial load size (code splitting does not reduce total JS, it defers it)
- Measure with field data segmented by device type -- bundle size improvements disproportionately help low-end devices
- Mention the tools by name: `rollup-plugin-visualizer`, `webpack-bundle-analyzer`, `source-map-explorer`, `web-vitals` library for RUM

</details>
