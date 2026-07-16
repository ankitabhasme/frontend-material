# Frontend System Design

> A dedicated discipline: designing the client architecture, data flow, rendering, and UX-quality attributes of a product — not a scaled-down backend HLD.

Frontend system design is judged on how you reason about **the browser as a constrained, adversarial runtime**: unreliable network, limited main thread, heterogeneous devices, hostile input, and a user who notices every dropped frame. Backend HLD optimizes throughput and consistency of servers you control; frontend design optimizes *perceived* performance, resilience, and accessibility on a runtime you don't control. Raise **accessibility, security, and performance proactively** — a lead who waits to be asked about them reads as a senior IC, not a lead.

## Answering Framework

Drive the conversation through these stages out loud. Spend the first ~20% on requirements; interviewers fail candidates who architect before scoping.

### 1. Clarify requirements

**Functional:** core user journeys, must-have vs nice-to-have, read/write mix, real-time vs request/response.

**Non-functional (the differentiators):**
- **Scale** — DAU, peak concurrency, list sizes (100 rows vs 1M), payload sizes.
- **Devices** — mobile-first? low-end Android? desktop dashboards? bandwidth/CPU budget.
- **Offline** — must it work offline / flaky network? optimistic writes?
- **SEO / shareability** — public and crawlable, or authed app behind login?
- **Latency budget** — target p75 LCP/INP; is this a latency-critical surface (search, checkout)?
- **i18n / l10n** — RTL, pluralization, locale formatting, bundle-per-locale.
- **Compliance / a11y bar** — WCAG 2.2 AA? regulated (finance/health)?

State assumptions explicitly and get a nod before moving on.

### 2. High-level architecture & component boundaries

- App shell vs feature modules; route-level code splitting boundaries.
- Container/presentational or feature-sliced structure; where domain logic lives (not in components).
- Shared design-system layer, data layer, and cross-cutting concerns (auth, telemetry, feature flags).
- Draw the box diagram: client app ↔ BFF/API ↔ services; where caching and real-time transports sit.

### 3. Client data model & entities

- Define the normalized entity shapes the client holds (User, Message, Post…), their IDs and relationships.
- Decide **normalized store** (entities keyed by id + lists of ids) vs nested trees. Normalization prevents duplication and makes cache updates O(1).
- Identify derived/computed state (never store what you can derive).

### 4. API design & data-fetching strategy

- **REST vs GraphQL:** GraphQL when clients need flexible, aggregated, over/under-fetch-sensitive queries and you control the graph; REST for simple, cacheable, CDN-friendly resources. Consider a **BFF** to shape payloads per client and hide fan-out.
- **Pagination:** *cursor* (opaque, stable under inserts, required for infinite feeds/real-time) vs *offset* (jumpable page numbers, cheap, but drifts as data mutates). Prefer cursor for feeds; offset for admin tables with page jumps.
- **Caching / normalization:** HTTP caching (ETag, `Cache-Control`, SWR) at the edge; client cache (React Query / Apollo / RTK Query) with staleness, dedup of in-flight requests, background refetch, and normalized cache updates so one mutation reflects everywhere.
- Batching (`DataLoader`-style), request coalescing, and prefetch on intent (hover/viewport).

### 5. State management

Classify state — most bugs come from conflating these:

| Kind | Example | Tool |
|---|---|---|
| Server cache | fetched lists/entities | React Query / SWR / Apollo |
| Global client | theme, auth session, feature flags | Context / Zustand / Redux |
| Local UI | modal open, form input | `useState`/`useReducer` |
| URL | filters, tab, page | router / query params |
| Ephemeral/derived | computed totals | selectors, `useMemo` |

Keep server state out of Redux. Push shareable state to the URL. Colocate local state.

### 6. Rendering strategy

| Strategy | Use when | Cost |
|---|---|---|
| CSR | authed app, no SEO, rich interactivity | slow first paint, JS-heavy |
| SSR | dynamic + SEO + fast FCP | server cost, TTFB, hydration |
| SSG | static/marketing, infrequently changing | rebuild to update |
| ISR | mostly static, periodic freshness | staleness window |
| Streaming SSR + RSC | large pages, ship less JS, progressive | complexity, infra |

Discuss **hydration cost**, islands/partial hydration, and streaming with Suspense to improve TTFB/LCP without blocking on slow data.

### 7. Performance optimizations

- **Load:** code-split by route/interaction, tree-shake, lazy-load below-fold, preconnect/preload critical assets, compress (Brotli), HTTP/2-3, CDN.
- **Assets:** responsive images (`srcset`, AVIF/WebP), `loading=lazy`, font-display swap, subset fonts.
- **Runtime:** virtualize long lists, memoize, avoid layout thrash, offload heavy work to Web Workers, debounce/throttle, `requestIdleCallback`.
- **Metrics:** Core Web Vitals — **LCP** (loading), **INP** (interactivity, replaced FID), **CLS** (stability); plus TTFB, TBT, bundle size budgets.

### 8. Accessibility

Semantic HTML first; ARIA only to fill gaps. Keyboard operability for every interaction, visible focus, focus management on route/modal changes, `prefers-reduced-motion`, color contrast, live regions for async updates, labels/roles. Follow **WAI-ARIA Authoring Practices (APG)** for widgets.

### 9. Security

- **XSS:** escape by default, avoid `dangerouslySetInnerHTML`, sanitize (DOMPurify), strict **CSP**.
- **CSRF:** SameSite cookies + tokens; or bearer tokens in memory.
- **Auth token storage:** access token in memory, refresh in httpOnly cookie — avoid `localStorage` for tokens.
- **Supply chain:** SRI, lockfiles, dependency audit. **Clickjacking:** frame-ancestors. **PII:** don't log it, redact in telemetry. Validate/authorize on the server — the client is never a trust boundary.

### 10. Edge cases: error / empty / loading / offline

For every data surface, design the **four states**: loading (skeletons, not spinners where possible), empty (with a next action), error (retry + fallback, distinguish transient vs fatal), and success. Add offline/degraded behavior, partial failure, slow-network, race conditions, and stale-data reconciliation. Optimistic UI needs a rollback path.

### 11. Trade-offs & what you'd revisit at scale

Close by naming what you deliberately deferred and the signal that would make you revisit it (see follow-ups below).

---

## Worked Examples

### 1) Typeahead / Autocomplete

- **Requirements:** sub-100ms perceived response, keyboard-navigable, handles slow/failed queries, ranks results, mobile + desktop.
- **Architecture:** controlled input → query controller (debounce + cancel) → cache → results listbox. Optional client-side trie for small static datasets; server search for large/dynamic.
- **Data/API:** `GET /search?q=&limit=` returning ranked items with ids; support prefix + fuzzy on server. Cache by normalized query key.
- **Key decisions:** debounce ~150–250ms; **cancel in-flight** requests with `AbortController` so a slow old query can't overwrite a fresh one; ignore out-of-order responses (compare request seq/query). Minimum query length. Cache recent queries (LRU).
- **Optimizations:** prefetch on focus (recent/popular), memoize rendering, virtualize if many results, highlight matches, request coalescing.
- **A11y / edge cases:** `role=combobox` + `aria-expanded`, `aria-activedescendant` for the highlighted option, `role=listbox`/`option`, Up/Down/Enter/Esc handling, announce result count via live region. Empty state, error state, no-results, network offline. Ranking: exact prefix > token match > fuzzy; tie-break by popularity/recency.

```ts
function useTypeahead(q: string) {
  const [state, setState] = useState<{items: Item[]; status: 'idle'|'loading'|'error'}>({items: [], status: 'idle'});
  useEffect(() => {
    if (q.length < 2) return;
    const ctrl = new AbortController();
    const t = setTimeout(async () => {
      setState(s => ({ ...s, status: 'loading' }));
      try {
        const res = await fetch(`/search?q=${encodeURIComponent(q)}`, { signal: ctrl.signal });
        setState({ items: await res.json(), status: 'idle' });
      } catch (e) {
        if ((e as Error).name !== 'AbortError') setState(s => ({ ...s, status: 'error' }));
      }
    }, 200);
    return () => { clearTimeout(t); ctrl.abort(); }; // cancel stale query + timer
  }, [q]);
  return state;
}
```

### 2) Infinite-scroll news feed

- **Requirements:** endless scroll, smooth on mobile, real-time-ish freshness, image-heavy.
- **Architecture:** virtualized list + IntersectionObserver sentinel + cursor pagination + server-cache layer.
- **Data/API:** `GET /feed?cursor=&limit=` returning items + `nextCursor`. Cursor (not offset) so new posts at the head don't shift pages and duplicate/skip items.
- **Key decisions:** **windowing/virtualization** to keep DOM nodes bounded regardless of scroll depth; measure variable heights (react-virtual). Prefetch next page before the sentinel is reached. Optimistic like/save with rollback. Handle stale data: pull-to-refresh / "N new posts" pill rather than auto-reordering under the user.
- **Optimizations:** image lazy-load + `srcset` + LQIP placeholders + fixed aspect-ratio boxes to avoid CLS; keep a bounded cache and recycle offscreen items; store scroll position for back-navigation restoration.
- **A11y / edge cases:** don't trap keyboard users; ensure content reachable without infinite scroll (or provide "load more" button fallback), announce new items politely. Handle end-of-feed, error mid-scroll (retry that page only), empty feed, duplicate ids on refetch.

### 3) Real-time chat / messaging

- **Requirements:** low-latency send/receive, ordering, delivery + read receipts, offline send, history.
- **Architecture:** WebSocket (or SSE for read-only) with a reconnect/backoff manager; local outbox queue; normalized message store keyed by clientMsgId + serverId.
- **Data/API:** WS events for `message`, `ack`, `receipt`, `presence`; REST `GET /conversations/:id/messages?before=<cursor>` for history (reverse cursor pagination).
- **Key decisions:** each outgoing message gets a **client-generated id** → optimistic render as "sending" → server `ack` reconciles to serverId + timestamp. **Ordering** by server sequence/logical clock, not client time; reorder on arrival. **Delivery/read receipts** as separate events updating message status. **Offline queue:** persist unsent messages (IndexedDB), flush on reconnect, dedupe by clientMsgId (idempotency). **Reconnection:** exponential backoff + jitter, resubscribe, fetch missed messages since last seq.
- **Optimizations:** message batching, windowed history list, debounce typing indicators, coalesce presence.
- **A11y / edge cases:** live region for incoming messages, focus stays in composer, keyboard send. Handle failed send (retry/delete), duplicate delivery, clock skew, message edits/deletes, huge conversations (virtualized + paged history).

### 4) Collaborative document editor

- **Requirements:** multiple users edit concurrently, converge without data loss, show presence + remote cursors, work offline.
- **Architecture:** local document model + sync engine over WebSocket; per-doc room; presence channel.
- **Conflict resolution:** **OT** (operational transform — transform ops against concurrent ops; central server ordering; e.g., Google Docs) vs **CRDT** (conflict-free replicated data type — commutative merges, no central authority, great for offline/P2P; e.g., Yjs/Automerge). Trade-off: OT simpler payloads but complex transform functions and server-authoritative; CRDT robust offline/decentralized but larger metadata and memory. For offline-first, lean CRDT.
- **Data/API:** stream ops/updates; periodic snapshots for fast load; version vectors for catch-up.
- **Key decisions:** presence + **remote cursors/selections** as a separate ephemeral channel (not persisted). Awareness state expires on disconnect. Undo/redo must be local-intent-aware.
- **Optimizations:** batch/coalesce keystrokes into ops, debounce awareness, snapshot + incremental updates, lazy-load large docs.
- **A11y / edge cases:** offline edits merge on reconnect; conflicting formatting; late joiners get snapshot then tail; large docs; cursor jitter; ensure screen-reader edit announcements aren't spammed by remote changes.

### 5) Large file uploader

- **Requirements:** GB-scale files, resumable, reliable on flaky networks, progress, integrity.
- **Architecture:** client chunker → parallel chunk uploader → server assembler; resumable protocol (tus / S3 multipart).
- **Data/API:** `POST /uploads` → uploadId; `PUT /uploads/:id/parts/:n`; `POST /uploads/:id/complete`. Server returns which parts already exist for resume.
- **Key decisions:** **chunk** into fixed sizes (e.g., 5–10MB); **parallelism** with a bounded pool (e.g., 3–6 concurrent) to saturate bandwidth without starving the tab; **retry** per-chunk with exponential backoff; **resumable** by querying uploaded parts on reconnect and skipping them; **integrity** via per-chunk checksum (e.g., MD5/CRC) + final ETag verification.
- **Optimizations:** hash off the main thread (Web Worker), throttle progress UI updates, adapt chunk size/concurrency to measured throughput, use `File.slice` to avoid loading whole file in memory.
- **A11y / edge cases:** progress announced via live region, pause/resume/cancel controls keyboard-accessible; handle tab close (persist upload state), duplicate upload dedupe by content hash, quota/permission errors, network loss mid-chunk.

### 6) Analytics dashboard

- **Requirements:** many widgets/charts, large datasets, interactive filters, exports, fast refilter.
- **Architecture:** widget grid → shared filter/query layer → server aggregation (don't ship raw rows) → cache. Charts render from pre-aggregated series.
- **Data/API:** parameterized aggregation endpoints (`GET /metrics?dim=&range=&granularity=`); server-side rollups; cursor/offset for tabular drill-downs.
- **Key decisions:** **aggregate on the server** — the client should never crunch millions of rows. **Virtualize** big tables; **canvas/WebGL** rendering (not SVG DOM) for high-cardinality charts. Cache query results keyed by filter set; dedupe identical widget queries. URL-encode filter state for shareable/bookmarkable views.
- **Optimizations:** debounce filter changes, request coalescing across widgets, incremental/streaming loads, memoized selectors, downsampling/LTTB for dense time series, Web Worker for any client transform.
- **A11y / edge cases:** charts need text/table alternatives and accessible summaries; keyboard-navigable legends/tooltips; exports (CSV/PDF) generated async with progress; empty/partial data per widget (one widget failing shouldn't kill the dashboard); timezone handling.

---

## Trade-off framing & common follow-ups

Frame every decision as **X over Y because [constraint], accepting [cost]; I'd revisit if [signal].** Interviewers push on:

- **Scale to 10x users:** where does it break? Answer with the client bottleneck (DOM node count, bundle size, WS fan-out, cache memory) and the mitigation (virtualization, CDN, pagination, sharded channels, backpressure).
- **Flaky / offline network:** retry with backoff + jitter, request cancellation, optimistic UI with rollback, offline queue + IndexedDB, SWR, idempotent writes.
- **Huge lists:** windowing/virtualization, cursor pagination, bounded cache, avoid re-render storms (memo, stable keys).
- **SEO:** SSR/SSG/streaming, meta/OG tags, structured data, ensure content in initial HTML, avoid client-only rendering for public pages.
- **Consistency vs latency:** optimistic (fast, may roll back) vs pessimistic (correct, slower); when money/correctness matters, confirm server-side.

Always tie back to **how you'd measure**: RUM for Web Vitals, synthetic Lighthouse in CI with budgets, error rate + retry telemetry, WS reconnect metrics, cache hit ratio, and funnel/abandonment for the UX itself.

### Interview Questions — Frontend System Design

**How is frontend system design different from backend HLD, and what do you insist on covering that juniors skip?**
> Backend HLD optimizes server throughput, consistency, and horizontal scaling; frontend design optimizes perceived performance, resilience, and UX quality on an uncontrolled runtime (variable network/CPU/device, hostile input). I insist on covering the four states of every data surface (loading/empty/error/success), accessibility, security (XSS/CSP/token storage), and Core Web Vitals proactively — not because I'm asked, but because at lead level these are requirements, not extras. I also spend real time on requirements/non-functionals before drawing boxes.

**Cursor vs offset pagination — when and why?**
> Offset (`?page=3`) is simple, supports jumping to arbitrary pages, and is cheap for static admin tables, but it drifts when items are inserted/deleted between requests, causing skipped or duplicated rows. Cursor pagination uses an opaque pointer to the last item, so it's stable under mutation and is the correct choice for infinite feeds and real-time data. The cost is you can't jump to page N. Rule of thumb: cursor for feeds/timelines, offset for paginated tables that need page numbers.

**Walk me through making a typeahead feel instant and correct under a slow network.**
> Debounce input (~200ms) to avoid a request per keystroke, use an AbortController to cancel the previous in-flight request so a slow old response can't overwrite a newer one, and guard against out-of-order responses by tagging requests and dropping stale ones. Cache results by normalized query (LRU) and prefetch popular/recent on focus. Perceived speed comes from optimistic display of cached prefixes and a debounce short enough to feel live. I'd measure INP and query-to-render latency, plus a "stale result rendered" error metric to confirm cancellation works.

**OT vs CRDT for a collaborative editor — how do you choose?**
> Both converge concurrent edits. OT transforms operations against concurrent ones and typically relies on a central server to order operations — smaller payloads but complex transform functions and server-authoritative, which is fine for online-first apps like Google Docs. CRDTs make operations commutative so replicas merge without coordination — excellent for offline-first and decentralized scenarios, at the cost of more per-character metadata and memory. If the requirement includes robust offline editing or P2P, I choose a CRDT (e.g., Yjs); if it's server-centric with tight payload budgets, OT. Presence/cursors go on a separate ephemeral channel either way.

**A feed must scroll infinitely on low-end phones without jank. What's your plan and where does it break at 10x?**
> Virtualize the list so DOM node count stays bounded regardless of scroll depth, use cursor pagination with prefetch of the next page before a sentinel is reached, lazy-load images with fixed aspect ratios to avoid CLS, and recycle offscreen nodes. At 10x, the break points are cache memory growth (bound it, evict offscreen), image bandwidth (responsive images + CDN), and re-render cost (stable keys, memoized rows). Freshness is handled with a "N new posts" pill rather than reordering under the user. I'd watch INP, long-task count, and memory in RUM segmented by device class.

**How would you store auth tokens and defend a SPA against XSS and CSRF?**
> Keep the access token in memory (JS variable/closure), and the refresh token in an httpOnly, Secure, SameSite cookie — never tokens in localStorage, which is fully readable by any XSS. Against XSS: escape by default, avoid dangerouslySetInnerHTML (sanitize with DOMPurify if unavoidable), and ship a strict Content-Security-Policy. Against CSRF: SameSite=Lax/Strict cookies plus anti-CSRF tokens or a custom header the server verifies. Crucially, the client is never a trust boundary — all authorization is enforced server-side.

**Design the data-fetching and caching layer for a dashboard with 20 widgets sharing filters.**
> A shared query layer keyed by filter set + widget query, so identical queries dedupe/coalesce into one request and results are cached with staleness (SWR: serve cached, revalidate in background). Server does all aggregation — the client never processes raw rows. Filter state lives in the URL so views are shareable and refilters are debounced. Each widget owns its loading/error/empty state so one failing widget doesn't blank the dashboard. I'd measure cache hit ratio, request count per interaction, and per-widget error rate.

**You've designed a feature; the interviewer says 'the network is flaky and users are offline half the time.' What changes?**
> I move to offline-first: persist critical state and pending writes in IndexedDB, queue mutations with client-generated idempotency keys and flush on reconnect, and reconcile using server sequence/timestamps. Reads use stale-while-revalidate so the UI works from cache. Writes become optimistic with an explicit rollback path on failure. Transport gets exponential backoff with jitter and request cancellation. I'd add telemetry for reconnect frequency, queue depth, and write-conflict rate so I can tune backoff and detect data-loss regressions.
