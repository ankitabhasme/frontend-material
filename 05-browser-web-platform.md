# Browser & Web Platform Internals

> How the browser turns bytes into pixels, where the platform gives you leverage (storage, workers, observers), and how a lead reasons about main-thread budget and offline.

## Browser Architecture (Processes & Threads)

Modern browsers (Chromium's multi-process model is the reference) split work across OS processes for security and stability. A crash or exploit in one tab shouldn't take down the browser or read another origin's memory.

- **Browser process**: the "kernel" — UI (address bar, tabs), network, storage, permissions. Privileged.
- **Renderer process**: one per site (Site Isolation), sandboxed. Runs the main thread (JS + DOM + style + layout + paint recording), the compositor thread, raster/worker threads, and any Web Worker threads.
- **GPU process**: shared; turns compositor commands into actual draw calls.
- **Network / Storage / utility processes**: fetch, cache, service-worker hosting.

Key mental model: **one main thread per renderer runs your JS, your layout, and your style recalc, and it's the same thread that must respond to input.** Everything you push off it (workers, compositor-only animations) protects responsiveness.

- **Site Isolation**: each site (eTLD+1) gets its own renderer, hardening against Spectre-style cross-origin reads. Cross-origin iframes render out-of-process (OOPIF).
- **Event loop**: macrotasks (timers, I/O callbacks, message events) vs microtasks (Promises, `queueMicrotask`, MutationObserver). The microtask queue drains fully between each macrotask and before rendering — a runaway microtask loop can starve rendering entirely.

```js
// Ordering: sync -> microtasks -> (render) -> next macrotask
console.log('1');
setTimeout(() => console.log('4 (macrotask)'), 0);
Promise.resolve().then(() => console.log('3 (microtask)'));
console.log('2');
// 1, 2, 3, 4
```

## The Rendering Pipeline (Pixel Pipeline)

```
HTML ──parse──▶ DOM ┐
                    ├─▶ Render Tree ─▶ Layout ─▶ Paint ─▶ Composite ─▶ pixels
CSS  ──parse──▶ CSSOM┘   (style)       (reflow)   (raster)  (GPU)
```

1. **Parse HTML → DOM**: incremental, tokenizer builds the node tree. A `<script>` (without `async`/`defer`) blocks the parser.
2. **Parse CSS → CSSOM**: CSS is render-blocking — the browser won't paint meaningful content until it has the CSSOM, because it can't know final styles.
3. **Style / Render tree**: match selectors to nodes, compute used values. `display:none` nodes are excluded; `visibility:hidden` are included (they occupy space).
4. **Layout (reflow)**: compute geometry — box sizes and positions. Expensive; can be superlinear because it cascades to descendants/ancestors.
5. **Paint**: record draw operations (fills, text, borders, shadows) into paint lists, per layer.
6. **Composite**: the compositor thread assembles painted layers (raster tiles) into the final frame on the GPU, applying transforms/opacity. This can happen **without touching the main thread**.

### Reflow vs Repaint vs Composite-only

| Change | Triggers | Cost |
|---|---|---|
| Read/write geometry (`width`, `top`, `offsetHeight`, `getBoundingClientRect`) | Layout → Paint → Composite | Highest |
| `color`, `background`, `box-shadow`, `visibility` | Paint → Composite | Medium |
| `transform`, `opacity`, `filter` (on a composited layer) | Composite only | Cheapest — runs off main thread |

- **Animate `transform`/`opacity`, never `top`/`left`/`width`.** The former stay on the compositor at 60fps even if the main thread is busy.
- **Layout thrashing**: interleaving DOM reads and writes forces synchronous ("forced") reflows. Batch reads, then writes. Verify in DevTools Performance panel — look for purple "Layout" bars and the "Forced reflow" warning, or use `FastDOM`-style read/write scheduling.

```js
// BAD: read-write-read-write forces reflow each iteration
els.forEach(el => { el.style.width = el.offsetWidth + 10 + 'px'; });

// GOOD: batch reads, then writes
const widths = els.map(el => el.offsetWidth);      // read phase
els.forEach((el, i) => el.style.width = widths[i] + 10 + 'px'); // write phase
```

### GPU Layers

A layer is promoted (its own composited layer / raster tile) by things like `transform`, `will-change`, `<video>`, `<canvas>`, `position:fixed`, and 3D transforms. Layers make animation cheap but each consumes GPU memory and raster time — **too many layers ("layer explosion") regresses performance**. Use `will-change` sparingly and remove it after the animation.

## Critical Rendering Path & Render-Blocking

The CRP is the minimum sequence to first render: HTML → CSSOM + DOM → render tree → layout → paint.

- **Render-blocking**: CSS in `<head>` (all of it, by default) and synchronous `<script>` in `<head>`.
- **Parser-blocking**: `<script>` without `async`/`defer` stops HTML parsing (and must wait for any pending CSS, since the script might read computed styles).

Levers a lead reaches for:
- Inline **critical CSS**, defer the rest (`<link rel="preload" as="style" onload="this.rel='stylesheet'">` or media-toggle trick).
- `defer` scripts (execute in order after parse) or `async` (execute ASAP, unordered) — modules are deferred by default.
- `<link rel="preload">` for late-discovered critical assets (fonts, hero image); `preconnect`/`dns-prefetch` for third-party origins.
- Measure with **LCP**, **FCP**, and the request waterfall. `<script defer>` moving LCP earlier is the proof.

```html
<link rel="preload" href="/fonts/inter.woff2" as="font" type="font/woff2" crossorigin>
<link rel="preconnect" href="https://api.example.com">
<script type="module" src="/app.js"></script> <!-- deferred by default -->
```

## Storage Tiers

| | Capacity | API | Persistence | Sent to server | Access from |
|---|---|---|---|---|---|
| **Cookies** | ~4KB each | sync | Configurable (expiry) | Yes, every request | Main + (HttpOnly hides from JS) |
| **localStorage** | ~5–10MB | sync | Until cleared | No | Main only |
| **sessionStorage** | ~5–10MB | sync | Per tab, until close | No | Main only |
| **IndexedDB** | Large (quota-based, often 100s MB–GB) | async | Until cleared | No | Main + Workers |
| **Cache API** | Quota-based | async (Promise) | Until cleared | No | Main + Service Worker |

- **Cookies**: only for data the server needs on every request (session tokens). Use `HttpOnly` (JS can't read → XSS token theft mitigated), `Secure`, `SameSite`. They add weight to every request — don't use for app state.
- **localStorage / sessionStorage**: synchronous → blocks the main thread; keep tiny. String-only (JSON serialize). Not for tokens (XSS-readable). `sessionStorage` is per-tab; `localStorage` shared across tabs of an origin (with a `storage` event for cross-tab sync).
- **IndexedDB**: the real client database — transactional, indexed, async, works in workers. Use a wrapper (`idb`, Dexie) — the raw API is event-based and verbose. Right choice for offline datasets, large blobs, request queues.
- **Cache API**: keyed by `Request` → `Response`; the backbone of Service Worker offline strategies. Separate from the HTTP cache.
- **Security**: all are origin-scoped. `localStorage`/IDB are readable by any script on the origin → an XSS steals everything. This is the core argument for `HttpOnly` cookies over `localStorage` for auth tokens (trade-off: cookies bring CSRF surface, mitigated by `SameSite`).

## Service Workers & PWA

A Service Worker is a network proxy that runs in its own thread, has no DOM, and persists beyond the page.

**Lifecycle**: `register → install → (waiting) → activate → fetch/message`. A new SW waits until all clients of the old one close, unless `self.skipWaiting()` + `clients.claim()`. Scope is defined by the SW's path.

### Caching Strategies

| Strategy | Behavior | Use for |
|---|---|---|
| **Cache-first** | Serve cache, fall to network | Hashed static assets (immutable) |
| **Network-first** | Try network, fall to cache | API data where freshness matters, HTML |
| **Stale-while-revalidate** | Serve cache immediately, update cache in background | Semi-fresh content (avatars, feeds) |
| **Cache-only / Network-only** | Explicit | App shell / non-cacheable |

```js
// stale-while-revalidate
self.addEventListener('fetch', (e) => {
  e.respondWith(caches.open('v1').then(async (cache) => {
    const cached = await cache.match(e.request);
    const network = fetch(e.request).then((res) => {
      cache.put(e.request, res.clone());
      return res;
    });
    return cached || network; // fast path if cached, else wait
  }));
});
```

- **Offline**: precache an app shell at install; serve it for navigations when offline.
- **Background Sync**: `sync` event retries failed mutations (e.g., queued POSTs) once connectivity returns — pair with an IndexedDB outbox.
- **Push**: `push` event + Push API/VAPID delivers server pushes even when the tab is closed; `notificationclick` focuses/opens a client.
- **Gotchas for a lead**: cache versioning & cleanup on `activate`; never cache-first your HTML (you'll strand users on stale shells); ship a kill-switch (SW that unregisters) in case of a bad deploy. Tools: **Workbox** to avoid hand-rolling.

## Web Workers

Run JS on a separate thread — no DOM access, communicate via `postMessage` (structured clone).

- **Offload**: parsing/CRDT/crypto/image processing/heavy compute so the main thread stays responsive (keep long tasks < 50ms).
- **Message passing**: data is **copied** by structured clone by default (cost proportional to size).
- **Transferable objects**: `ArrayBuffer`, `MessagePort`, `ImageBitmap`, `OffscreenCanvas` can be *transferred* (zero-copy, ownership moves) — the sender loses access.
- **SharedArrayBuffer**: true shared memory across threads (with `Atomics` for coordination). Gated behind cross-origin isolation (`COOP`/`COEP` headers) after Spectre.

```js
const buf = new ArrayBuffer(1024 * 1024);
worker.postMessage({ buf }, [buf]); // transfer, not copy — buf now detached here
```

Types: **Dedicated** (one page), **Shared** (multiple same-origin contexts), **Service** (proxy, above). For UI framework work, note **OffscreenCanvas** lets you render canvas off the main thread.

## Observer APIs (declarative, off the hot path)

| API | Watches | Common use |
|---|---|---|
| **IntersectionObserver** | Element visibility vs viewport/root | Lazy-load images, infinite scroll, impression tracking, autoplay-on-view |
| **ResizeObserver** | Element size changes | Container-driven layout, canvas resize, charting |
| **MutationObserver** | DOM tree changes | React to 3rd-party DOM, detect injected nodes |
| **PerformanceObserver** | Perf entries (LCP, CLS, long tasks, resource timing) | Field RUM, Web Vitals |

Why observers beat polling/`scroll` handlers: they fire asynchronously, batched, off the scroll-critical path — no layout thrashing from reading `getBoundingClientRect` in a scroll listener.

```js
const io = new IntersectionObserver((entries) => {
  for (const e of entries) if (e.isIntersecting) loadImage(e.target);
}, { rootMargin: '200px' }); // prefetch before it enters view
document.querySelectorAll('img[data-src]').forEach((img) => io.observe(img));
```

```js
// Measure real-user LCP
new PerformanceObserver((list) => {
  const last = list.getEntries().at(-1);
  report('LCP', last.startTime);
}).observe({ type: 'largest-contentful-paint', buffered: true });
```

## rAF vs rIC

- **`requestAnimationFrame`**: callback runs right before the next paint (~16.6ms at 60Hz). Use for visual updates and to batch DOM writes into a frame. Reading layout inside rAF gives you post-layout values.
- **`requestIdleCallback`**: runs when the main thread is idle, with a `timeRemaining()` deadline. Use for non-urgent work (analytics beacons, prefetch, cache warming). Always pass a `timeout` so it isn't starved forever. Not for anything visible or time-critical.

```js
requestIdleCallback((deadline) => {
  while (deadline.timeRemaining() > 0 && queue.length) process(queue.shift());
}, { timeout: 2000 });
```

## Real-Time Transport: WebSockets vs SSE vs Polling

| | Direction | Protocol | Reconnect | Best for |
|---|---|---|---|---|
| **Polling / long-polling** | Client-pull | HTTP | Manual | Simple, low-frequency; fallback |
| **SSE (EventSource)** | Server→client | HTTP (text) | Built-in auto-reconnect + `Last-Event-ID` | Live feeds, notifications, one-way streams |
| **WebSocket** | Full duplex | `ws`/`wss` (upgraded HTTP) | Manual | Chat, collaboration, games, bidirectional |

- **SSE**: text-only, auto-reconnects, multiplexes over HTTP/2, simple server model. Limited by browser per-domain connection cap on HTTP/1.1.
- **WebSocket**: binary + text, lowest overhead per message, but you own reconnect/heartbeat/backpressure and scaling (sticky sessions or a pub/sub fan-out like Redis). Doesn't get HTTP caching/compression semantics.
- **Decision**: one-way server→client? SSE. Bidirectional/low-latency? WebSocket. Occasional updates or must traverse hostile proxies? Polling.

## Event Model: Delegation, Capturing, Bubbling

Events flow **capture (root→target) → target → bubble (target→root)**. Listeners default to the bubble phase; pass `{ capture: true }` for the capture phase.

- **Delegation**: attach one listener on a container, use `event.target.closest(...)` to handle events from many children — fewer listeners, works for dynamically added nodes.
- `stopPropagation()` halts the flow; `preventDefault()` cancels default behavior (they're independent). `passive: true` on scroll/touch listeners promises no `preventDefault` so the browser can scroll without waiting on JS.
- `focus`/`blur` don't bubble (use `focusin`/`focusout`); this trips up delegation.

```js
list.addEventListener('click', (e) => {
  const item = e.target.closest('[data-id]');
  if (item) select(item.dataset.id);
});
```

### Interview Questions — Browser & Web Platform

**Walk me through what happens from the moment CSS finishes downloading to pixels on screen, and where a slow animation would show up.**
> The browser builds the CSSOM, combines it with the DOM into the render tree, runs layout to compute geometry, paints draw commands into layers, and the compositor rasterizes and assembles them on the GPU. A janky animation shows up in the Performance panel as long main-thread tasks: if you're animating `width`/`top` you trigger layout+paint every frame; move to `transform`/`opacity` so the work stays on the compositor thread and survives a busy main thread. I'd confirm by checking for dropped frames and "forced reflow" warnings.

**When does a style change cost you a reflow vs a repaint vs nothing but compositing?**
> Anything that changes geometry (size, position, reading `offsetHeight`) triggers layout, then paint, then composite. Paint-only changes like `color` or `box-shadow` skip layout. `transform`, `opacity`, and `filter` on a promoted layer are composite-only and run off the main thread. The lead-level point is to keep animations in that last bucket and to avoid layout thrashing by batching reads before writes.

**Where would you store an auth token, and defend the trade-off.**
> An `HttpOnly`, `Secure`, `SameSite` cookie, not `localStorage`. `localStorage` is synchronous and readable by any script on the origin, so a single XSS exfiltrates the token; `HttpOnly` cookies are invisible to JS. The cost is CSRF exposure, which I mitigate with `SameSite=Lax/Strict` and, where needed, anti-CSRF tokens. If the API is cross-site and I must use bearer tokens, I keep them in memory with a short TTL and refresh via an `HttpOnly` refresh cookie.

**Which caching strategy do you pick for the app shell vs API data in a Service Worker, and what's the classic mistake?**
> Cache-first for hashed immutable assets and the precached shell; network-first (or stale-while-revalidate) for API/HTML where freshness matters. The classic mistake is cache-first on navigation HTML — users get stranded on a stale shell after a deploy. I version caches, clean old ones on `activate`, and keep a kill-switch SW that unregisters, so a bad worker can't brick the site.

**A CPU-heavy operation is freezing the UI. How do you fix it, and how do you move the data efficiently?**
> Move the compute into a Web Worker so the main thread stays free for input and rendering. To avoid the structured-clone copy cost, transfer large buffers (`ArrayBuffer`, `ImageBitmap`) as transferables — zero-copy, ownership moves to the worker. If threads need shared state, `SharedArrayBuffer` with `Atomics`, but that requires cross-origin isolation via COOP/COEP. I'd verify by watching for long tasks disappearing from the main-thread flame chart.

**Why prefer IntersectionObserver over a scroll listener for lazy-loading and impression tracking?**
> A scroll listener fires synchronously and tempts you to call `getBoundingClientRect` on every event, thrashing layout on the scroll-critical path. IntersectionObserver reports visibility asynchronously and batched, off that path, with a `rootMargin` to prefetch before entry. It's cheaper, doesn't jank scrolling, and is declarative about the threshold.

**SSE or WebSocket for a live notifications feed? For a collaborative editor?**
> Notifications are one-way server→client, so SSE: it auto-reconnects with `Last-Event-ID`, rides HTTP/2, and needs no custom protocol. A collaborative editor needs low-latency bidirectional messages, so WebSocket — accepting that I now own heartbeats, reconnection with backoff, backpressure, and horizontal scaling via sticky sessions or a Redis pub/sub fan-out.

**Explain rAF vs requestIdleCallback and give a concrete use for each.**
> `requestAnimationFrame` runs just before paint, so I batch visual DOM writes there to hit the frame budget. `requestIdleCallback` runs in leftover idle time with a deadline, so I use it for non-urgent work like flushing analytics or prefetching, always with a `timeout` so it isn't starved. Never use rIC for anything the user is waiting to see.

**How do async script attributes interact with the critical rendering path?**
> A plain `<script>` in the head is parser- and render-blocking and may also wait on pending CSS. `defer` downloads in parallel and executes in order after the DOM is parsed — ideal for app code. `async` executes as soon as it downloads, order-independent — fine for independent third-party tags. Modules are deferred by default. I'd measure the effect on FCP/LCP in the waterfall to prove the script moved off the critical path.
