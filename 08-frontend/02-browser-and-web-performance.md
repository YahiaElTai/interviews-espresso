# Browser & Web Performance

> **36 questions** — 16 theory, 20 practical

- Critical rendering path: DOM, CSSOM, render-blocking resources (async, defer, module scripts), preload scanner
- Browser compositing architecture: layout, paint, composite phases — CSS property triggers (will-change, transform, opacity), GPU-accelerated animations, avoiding forced reflows
- JavaScript execution cost: parsing, compilation, main thread blocking, long tasks, impact on responsiveness (INP)
- Core Web Vitals: LCP, INP, CLS — diagnosis and optimization
- Lab data vs field data (RUM) for performance measurement
- SSR vs CSR vs SSG vs ISR — tradeoffs and per-route mixing
- Hydration: cost, mismatches, progressive/partial hydration, streaming SSR
- HTTP caching: Cache-Control directives, ETag, conditional requests, CDN caching
- Service workers: caching strategies, lifecycle, update pitfalls
- Bundle optimization: tree shaking, code splitting, vendor chunking, bundle analysis
- Image optimization: responsive images, srcset/sizes, lazy loading, WebP/AVIF
- Font loading: FOIT vs FOUT, font-display, layout shift prevention
- Third-party script impact: async loading, facades, performance isolation strategies
- Chrome DevTools performance auditing: flame charts, long tasks, layout thrashing
- Real user monitoring with web-vitals library and PerformanceObserver
- Performance budgets: bundle size limits, metric thresholds, CI enforcement, preventing regressions over time
- Resource loading optimization: resource hints (preconnect, preload, prefetch, dns-prefetch), critical CSS inlining, early hints (103)
---

## Foundational

<details>
<summary>1. How does the critical rendering path work end-to-end — what are DOM construction, CSSOM construction, and render tree formation, why are CSS and synchronous JS render-blocking by default, and why does understanding this pipeline matter for every performance optimization decision you make as a senior frontend engineer?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>2. How does the browser's compositing architecture split rendering into layout, paint, and composite phases — why does the browser separate these into distinct stages, what triggers each phase to re-run, and why does understanding which CSS properties trigger layout vs paint vs composite-only changes fundamentally change how you write performant animations and UI updates?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>3. What are Core Web Vitals (LCP, INP, CLS), why did Google choose these three specific metrics over alternatives like Time to Interactive or First Contentful Paint, what does each one actually measure from the user's perspective, and how do they connect to real-world user experience and business outcomes?</summary>

<!-- Answer will be added later -->

</details>

## Conceptual Depth

<details>
<summary>4. Why do you need both lab data (Lighthouse, WebPageTest) and field data (RUM via CrUX, web-vitals library) to understand performance — what does each one capture that the other misses, when can lab data actively mislead you about real user experience, and how do you reconcile conflicting results between the two?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>5. Why do different rendering strategies (SSR, CSR, SSG) exist — what specific problem does each one solve, what are the performance and SEO tradeoffs of each, and when does each approach become the wrong choice?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>6. Why is per-route rendering strategy mixing (using SSR for some pages, SSG for others, CSR for others) often the right answer instead of picking one approach globally — what factors determine which strategy a given route should use, and what complexity does mixing introduce in your build and deployment pipeline?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>7. Why is hydration expensive and error-prone — what actually happens during hydration, what causes hydration mismatches and why are they dangerous, how do progressive hydration, partial hydration (islands architecture), and streaming SSR each address the hydration cost problem, and what are the tradeoffs of each approach?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>8. How does the HTTP caching model work for frontend assets — what do the key Cache-Control directives (max-age, no-cache, no-store, immutable, stale-while-revalidate) actually mean, how do ETag and conditional requests (If-None-Match) prevent unnecessary transfers, and how does CDN caching layer on top of browser caching to create a multi-tier cache with different invalidation concerns?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>9. Why do service workers exist as a separate caching layer from HTTP caching — how does the service worker lifecycle (install, activate, fetch) work, what are the main caching strategies (cache-first, network-first, stale-while-revalidate), and what are the dangerous update pitfalls that can leave users stuck on stale versions of your app?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>10. Why do bundle size optimizations (tree shaking, code splitting, vendor chunking) matter even with fast networks — how does each technique work under the hood, what are the tradeoffs between aggressive code splitting (more requests, better caching) vs fewer larger bundles (fewer requests, worse caching), and when does over-splitting actually hurt performance?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>11. Why is image optimization the highest-impact performance win on most websites — what are the tradeoffs between WebP and AVIF formats, how do responsive images with srcset/sizes prevent unnecessary bandwidth usage, why does lazy loading below-the-fold images matter for LCP, and when can aggressive image optimization backfire?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>12. Why do web fonts cause both rendering delays and layout instability — what is the difference between FOIT (Flash of Invisible Text) and FOUT (Flash of Unstyled Text), how do the font-display descriptor values (swap, fallback, optional) change the tradeoff between rendering speed and visual stability, and how do fonts contribute to CLS?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>13. Why are third-party scripts one of the most common causes of performance degradation — how do they block the main thread, what strategies exist for isolating their impact (async/defer loading, facades, web workers, iframes), and how do you make the business case for removing or deferring a third-party script when the marketing team insists it stays?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>14. How do resource hints (preconnect, preload, prefetch, dns-prefetch) and critical CSS inlining improve loading performance — what does each resource hint do, when does each one help vs waste bandwidth, how does critical CSS inlining eliminate render-blocking round trips, and what are the risks of over-using preload or inlining too much CSS?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>15. What problem does Incremental Static Regeneration (ISR) solve that plain SSG and SSR don't — how does ISR's stale-while-revalidate model work in Next.js, what are the tradeoffs around stale content windows, when does ISR break down (personalized content, high-frequency updates), and when should you choose on-demand revalidation over time-based revalidation?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>16. Why is JavaScript execution cost one of the biggest performance bottlenecks in modern web apps — what happens during parsing and compilation, why does main thread blocking create long tasks, how do long tasks directly degrade INP, and what strategies (code splitting, web workers, yielding to the main thread) reduce execution cost without sacrificing functionality?</summary>

<!-- Answer will be added later -->

</details>

## Practical — Measurement & Auditing

<details>
<summary>17. Walk through how you use Chrome DevTools Performance panel to audit a slow page — how do you record and read a flame chart, identify long tasks blocking the main thread, detect layout thrashing (forced synchronous layouts), and determine whether a performance problem is CPU-bound (scripting/rendering) vs network-bound? Show what specific patterns you look for in the trace.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>18. Set up real user monitoring for Core Web Vitals using the web-vitals library and PerformanceObserver — show the code for collecting LCP, INP, and CLS in production, explain how you send these metrics to your analytics backend, what attribution data you capture to make the metrics actionable, and how you handle the differences between page loads and SPA navigations.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>19. A page's LCP is 4.5 seconds in the field but 2.1 seconds in Lighthouse — walk through the systematic diagnosis process: how you identify the LCP element, determine whether the bottleneck is server response time, render-blocking resources, resource load time, or client-side rendering delay, and what specific fixes you apply for each root cause. Show the DevTools workflow and the code/config changes.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>20. A page has a CLS score of 0.35 in the field — walk through how you diagnose which elements are shifting and why: how you use the Layout Shift attribution in DevTools and the web-vitals library to find the culprits, and show the specific fixes for the most common causes (images without dimensions, late-loading fonts, dynamically injected content, ads).</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>21. A page has poor INP (400ms) in the field — walk through how you identify which interactions are slow using the web-vitals library with attribution, how you find the responsible long tasks in a DevTools performance trace, and what specific fixes you apply for the most common causes (heavy event handlers, layout thrashing during interaction, third-party scripts blocking the main thread).</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>22. Set up performance budgets for a frontend project — show how you define bundle size limits and Core Web Vitals thresholds, integrate them into CI so builds fail when budgets are exceeded (using tools like bundlesize or Lighthouse CI), and explain how you prevent performance regressions from creeping in over time as the team ships new features.</summary>

<!-- Answer will be added later -->

</details>

## Practical — Rendering & Caching Configuration

<details>
<summary>23. Configure the loading behavior of scripts on a page that has a critical inline script, a large framework bundle, a non-critical analytics script, and a module-based component — show how you use async, defer, type="module", and the preload scanner to control execution order, explain what render-blocking means for each configuration, and what breaks if you get the loading order wrong.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>24. Design the HTTP caching strategy for a production frontend app that serves hashed JS/CSS bundles, an index.html entry point, API responses, and static images through a CDN — show the Cache-Control headers for each resource type, explain why index.html must never be cached aggressively, how you handle cache busting on deployment, and what happens when the CDN serves a stale index.html pointing to deleted hashed bundles.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>25. Implement a service worker with a stale-while-revalidate strategy for app shell assets and a network-first strategy for API calls — show the code for install, activate, and fetch event handlers, explain how you handle the update flow so users get new versions without being stuck on stale code, and what skipWaiting/clients.claim tradeoffs you need to consider.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>26. Set up streaming SSR with selective hydration in a React application — show the server-side rendering code using renderToPipeableStream, how Suspense boundaries control the streaming and hydration order, and explain how this approach improves Time to First Byte and INP compared to traditional SSR where the entire page must render before any HTML is sent.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>27. Configure ISR in a Next.js application for a product catalog page — show the page component with getStaticProps and revalidate, explain how fallback: 'blocking' vs fallback: true affects user experience for uncached pages, how on-demand revalidation via res.revalidate() works for immediate updates, and what monitoring you set up to detect when users are seeing stale content longer than expected.</summary>

<!-- Answer will be added later -->

</details>

## Practical — Asset Optimization & Delivery

<details>
<summary>28. Configure code splitting, vendor chunking, and tree shaking in a Vite or Webpack build for a large React application — show the configuration for splitting vendor libraries into a stable long-cached chunk, route-based dynamic imports with React.lazy, and how you use a bundle analyzer to find unexpectedly large modules. Explain what configuration mistakes silently break tree shaking.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>29. Set up responsive images with srcset and sizes for a content-heavy page — show the HTML for an image that serves different resolutions at different viewport widths, configure lazy loading for below-the-fold images while ensuring the LCP image loads eagerly, implement WebP/AVIF with fallbacks using the picture element, and explain why getting the sizes attribute wrong defeats the purpose of srcset.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>30. Optimize font loading for a site that uses two custom fonts (a heading font and a body font) — show the @font-face declarations with appropriate font-display values, implement font preloading for the critical body font, set up size-adjusted fallback fonts to minimize CLS during font swap, and explain why you might choose font-display: optional for a non-critical decorative font.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>31. A product page loads a heavy third-party chat widget, an analytics SDK, and a social media embed that together add 400ms to the main thread — show how you implement a facade pattern for the chat widget (render a fake button that loads the real widget on click), defer the analytics script until after the page is interactive, and isolate the social embed in an iframe. Explain the tradeoffs of each isolation strategy.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>32. A page loads slowly on 3G connections despite good Lighthouse scores on fast networks — walk through the diagnosis: how you use resource hints (preconnect to critical origins, dns-prefetch for third parties), extract and inline critical CSS to eliminate render-blocking stylesheets, set up preload for key resources discovered late in the waterfall, and configure the server to support early hints (103). Show the HTML and server configuration changes.</summary>

<!-- Answer will be added later -->

</details>

---

## Experience-Based Questions

These questions test real-world experience. Prepare by mapping them to your own projects and situations.

<details>
<summary>33. Tell me about a time you diagnosed and fixed a Core Web Vitals regression in production — what metric regressed, how did you discover it, what was the root cause, and what was the fix? How did you prevent similar regressions going forward?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>34. Describe a time you migrated a client-side rendered application to SSR or SSG — what drove the decision, what rendering strategy did you choose and why, what were the biggest technical challenges (hydration issues, caching, infrastructure), and what was the measurable performance impact?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>35. Tell me about a time you significantly reduced bundle size or improved loading performance for a large frontend application — what analysis did you do to find the biggest wins, what specific changes did you make, and how did you measure the impact in the field (not just in lab tools)?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>36. Describe a time you had to deal with a third-party script or integration that was degrading your site's performance — how did you measure the impact, what solution did you implement, and how did you balance the performance concerns against the business need for the integration?</summary>

<!-- Answer framework will be added later -->

</details>
