# Performance Optimization

> Core Web Vitals, React rendering, bundle/network/asset layers, rendering strategies, and how to measure — the "make it fast, prove it" round.

Think in layers: **rendering, JS/bundle, network, assets, runtime, measurement.**

## Core Web Vitals (must know)
- **LCP (Largest Contentful Paint)** — loading; largest element visible. Target **< 2.5s**. Improve: optimize hero image, preload critical assets, faster TTFB, SSR/streaming, remove render-blocking resources.
- **INP (Interaction to Next Paint)** — responsiveness (replaced FID in 2024). Target **< 200ms**. Improve: break up long tasks, reduce main-thread work, `startTransition`, web workers, debounce/throttle.
- **CLS (Cumulative Layout Shift)** — visual stability. Target **< 0.1**. Improve: set width/height/`aspect-ratio` on media, reserve space for ads/embeds, careful `font-display`, avoid inserting content above existing.
- Others: **TTFB, FCP, TBT** (lab proxy for INP).

## React Rendering Optimization
- **Avoid unnecessary re-renders:** `React.memo` (props equality), `useMemo`/`useCallback` for stable references, split state so updates are localized, lift state only as far as needed.
- **Context pitfall:** every consumer re-renders when context value changes. Fix: split contexts, memoize the value, or use a selector-based store (Zustand/Redux with selectors, `use-context-selector`).
- **List virtualization/windowing** — render only visible rows (`react-window`, TanStack Virtual) for large lists/tables.
- **Keys** — stable keys prevent remount churn.
- **State colocation** — keep state close to where it's used to shrink re-render scope.
- **`useTransition` / `useDeferredValue`** — keep interactions responsive during heavy renders.
- **React Compiler** — auto-memoization reduces manual work.

## Bundle / JS Optimization
- **Code splitting** — route-based (`React.lazy` + `Suspense`) and component-based dynamic imports.
- **Tree shaking** + avoiding barrel files that defeat it.
- **Analyze** — `webpack-bundle-analyzer`, `source-map-explorer` to find bloat (moment.js, full lodash → `lodash-es`/per-method or date-fns).
- **Minification** (Terser/esbuild), **compression** (gzip, and **Brotli** for better ratios).
- **Defer/async scripts**, remove unused polyfills via `browserslist`.
- **Prefetch/preload** — `<link rel="preload">` critical assets; `rel="prefetch"` likely-next routes; webpack magic comments (`/* webpackPrefetch: true */`).

## Network / Delivery
- **CDN** — serve static assets from edge close to users.
- **HTTP caching** — `Cache-Control`, `ETag`, immutable hashed assets (`max-age=31536000, immutable`).
- **HTTP/2 / HTTP/3** — multiplexing removes the need for aggressive concatenation; HTTP/3 (QUIC) improves on lossy networks.
- **Resource hints** — `preconnect`, `dns-prefetch` for third-party origins.
- **Reduce waterfalls** — parallelize requests, avoid sequential dependent fetches, use SSR/streaming, colocate data.

## Asset Optimization
- **Images** — modern formats (WebP/AVIF), responsive `srcset`/`sizes`, lazy-load offscreen (`loading="lazy"`), correct dimensions, CDN image transforms.
- **Fonts** — subset, `font-display: swap`, `preload`, self-host or use `size-adjust` to reduce CLS; limit weights.
- **CSS** — critical CSS inlined, remove unused (PurgeCSS), avoid huge blocking stylesheets.

## Rendering Strategies
- **CSR** — ship JS, render in browser. Fast TTFB, slow FCP/LCP, poor SEO by default.
- **SSR** — render HTML on server per request. Better FCP/SEO, higher server cost, hydration cost.
- **SSG** — pre-render at build. Fastest, but stale until rebuild.
- **ISR** — SSG + periodic/background regeneration.
- **Streaming SSR + RSC** — stream HTML and hydrate progressively; ship less JS.
- **Hydration cost** — a real perf tax; mitigations: partial/progressive hydration, islands architecture, RSC.

## Measurement
- **Lab:** Lighthouse, WebPageTest, React DevTools Profiler, Chrome Performance panel.
- **Field (RUM):** `web-vitals` library, real-user monitoring, `PerformanceObserver`.
- **Rule:** measure first, optimize the biggest bottleneck, verify with data. Avoid premature micro-optimizations.

---

### Interview Questions — Performance

**Q1. Walk me through diagnosing a slow-loading page.**
> Start with data: run Lighthouse + field RUM to see which Core Web Vital is failing. If LCP: inspect the LCP element, check TTFB, render-blocking CSS/JS, image size/format, preload. If INP: profile long tasks, break up JS, offload work. If CLS: find shifting elements, reserve space. Then fix the biggest contributor and re-measure.

**Q2. INP is high on a search page while typing. Fix?**
> Debounce the query, wrap the expensive list update in `startTransition` (keep the input urgent), virtualize the results list, memoize rows, and move heavy computation off the main thread (web worker) if needed.

**Q3. How do you stop unnecessary re-renders from Context?**
> Split context by concern, memoize the provider value, and use selector-based subscriptions so consumers only re-render on the slice they read. Or move high-frequency state to an external store with selectors.

**Q4. Bundle is 2MB. Process to cut it?**
> Analyze with a bundle analyzer, then: route/component code-splitting, replace heavy libs (moment→date-fns, full lodash→per-method), ensure tree-shaking (ESM, no bad barrels), enable Brotli, drop unused polyfills via browserslist, lazy-load below-the-fold features.

**Q5. What changed from FID to INP?**
> FID measured only the delay of the *first* interaction. INP measures responsiveness across *all* interactions during the visit, so it captures ongoing jank, not just initial input.

**Q6. Why is hydration expensive and how do you reduce it?**
> Hydration re-runs component logic and attaches listeners over server HTML, costing main-thread time proportional to app size. Reduce it with less client JS via Server Components, islands/partial hydration, code-splitting, and streaming so interactivity arrives progressively.
