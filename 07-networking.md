# Networking for Frontend

> How data actually gets to the browser — protocol versions, caching semantics, API paradigms, and the cancellation/retry hygiene a lead is expected to own.

## HTTP/1.1 vs HTTP/2 vs HTTP/3

| | HTTP/1.1 | HTTP/2 | HTTP/3 |
|---|---|---|---|
| Transport | TCP | TCP | **QUIC over UDP** |
| Multiplexing | No (1 request/connection at a time) | Yes (many streams / connection) | Yes |
| Head-of-line blocking | At HTTP layer | Removed at HTTP layer, **still at TCP layer** | **Removed** (per-stream, in QUIC) |
| Header compression | None (plain text) | HPACK | QPACK |
| Connection setup | TCP handshake | TCP + TLS | **1-RTT / 0-RTT** (QUIC folds in TLS 1.3) |
| Server push | No | Yes (**deprecated** in practice) | Effectively gone |

- **HTTP/1.1**: one in-flight request per connection → browsers open ~6 parallel connections per origin, motivating hacks like domain sharding and sprite sheets.
- **HTTP/2**: **multiplexes** many streams over one connection, compresses headers (HPACK), and prioritizes streams. This killed sharding/concatenation as best practices. But because it rides TCP, a single lost packet stalls *all* streams — **TCP-level head-of-line blocking**. Server push is deprecated (browsers removed support) — use `103 Early Hints` + `preload` instead.
- **HTTP/3 / QUIC**: runs over UDP, integrates TLS 1.3, and gives each stream independent delivery so packet loss on one stream doesn't block others. Faster connection setup (0-RTT resumption) and seamless connection migration across networks (Wi-Fi↔cellular).

Lead takeaway: on H2/H3, **bundle less aggressively** (many small cacheable chunks are fine and improve cache granularity); the old "concatenate everything" advice is a net negative because one changed byte busts one big file.

## TLS / HTTPS Handshake & TTFB

- **TLS 1.3 handshake**: 1-RTT (0-RTT on resumption). Client hello → server hello + cert → keys derived → encrypted. On QUIC it's folded into the transport handshake.
- **TTFB (Time To First Byte)** = DNS + TCP + TLS + server think time + first byte travel. High TTFB usually means backend latency or missing CDN/edge caching, not payload size. Break it down in DevTools' timing tab (Queueing, DNS, Connecting, SSL, Waiting/TTFB, Content Download).
- Cut connection setup with `preconnect` (does DNS + TCP + TLS ahead of time) for known critical third-party origins.

## HTTP Caching (in depth)

Two goals: avoid re-fetching (freshness) and avoid re-downloading unchanged bytes (validation).

**Freshness — `Cache-Control`:**
- `max-age=<s>` — fresh for N seconds (client cache).
- `s-maxage=<s>` — overrides `max-age` for shared caches/CDNs.
- `no-cache` — *may* cache but **must revalidate** before use (not "don't cache").
- `no-store` — never cache (sensitive data).
- `private` vs `public` — `private` = browser only, not CDN.
- `immutable` — never revalidate during freshness; for fingerprinted assets (`app.9f3c.js`).
- `stale-while-revalidate=<s>` — serve stale instantly, revalidate in background (perceived-instant + fresh soon).
- `stale-if-error=<s>` — serve stale if origin errors.

**Validation (conditional requests) when stale:**
- **`ETag`** (content hash) → client sends `If-None-Match` → server replies `304 Not Modified` (no body) if unchanged.
- **`Last-Modified`** → client sends `If-Modified-Since`. Weaker (1s granularity); ETag is preferred.

```
# Fingerprinted static asset — cache forever, never revalidate
Cache-Control: public, max-age=31536000, immutable

# HTML / API you want fresh but resilient
Cache-Control: no-cache            # or:
Cache-Control: max-age=0, stale-while-revalidate=60
ETag: "a1b2c3"
```

**CDN & invalidation:**
- Cache immutable, hashed assets at the edge with long TTLs; cache HTML short or `no-cache`.
- **Cache-busting via content hash in the filename** is the invalidation strategy — a new deploy = new URL, so you never need to purge. Explicit CDN purge/`Surrogate-Key` tagging is the escape hatch for un-hashed resources.
- "There are only two hard things..." — cache invalidation. Prefer designs that don't need active purging.

## Status Codes That Matter

| Code | Meaning / frontend action |
|---|---|
| 200 / 201 / 204 | OK / Created / No Content |
| 206 | Partial (range requests, media) |
| **301 / 308** | Permanent redirect (308 keeps method/body) |
| **302 / 307** | Temporary redirect (307 keeps method/body) |
| **304** | Not Modified — use cached body |
| **400 / 422** | Bad request / validation failed |
| **401 / 403** | Unauthenticated / authenticated-but-forbidden (don't retry blindly) |
| **404 / 410** | Not found / gone |
| **409** | Conflict (optimistic concurrency) |
| **429** | Rate limited — honor `Retry-After`, back off |
| **500 / 502 / 503 / 504** | Server / gateway / unavailable / timeout — safe to retry idempotent requests with backoff |

## Request Lifecycle, Waterfalls, Connection Reuse

- **Waterfall**: DNS → connect → TLS → request → TTFB → download, per resource. Serial dependencies (JS that fetches JSON that renders that fetches more) create long critical chains — the thing to flatten.
- **Connection reuse**: keep-alive / H2 multiplexing avoids repeating DNS+TCP+TLS. Warm critical origins early.
- **Resource hints**:
  - `preconnect` — full connection setup to an origin (use for 1–2 critical cross-origin hosts; costs sockets).
  - `dns-prefetch` — just DNS (cheap, broad).
  - `preload` — fetch a specific late-discovered critical asset at high priority.
  - `prefetch` — low-priority fetch for a likely *next* navigation.
  - `modulepreload` — preload+parse ES modules.

```html
<link rel="preconnect" href="https://cdn.example.com" crossorigin>
<link rel="dns-prefetch" href="https://analytics.example.com">
<link rel="preload" href="/hero.avif" as="image" fetchpriority="high">
```

Fix waterfalls by parallelizing independent fetches, hoisting data requirements up (server-side data fetching / streaming SSR), and preloading the resources you know you'll need.

## API Paradigms

| | REST | GraphQL | tRPC | gRPC-web |
|---|---|---|---|---|
| Contract | URLs + verbs | Schema + query language | TS types (end-to-end) | Protobuf (IDL) |
| Fetching shape | Fixed per endpoint | Client selects fields | RPC, typed | RPC, typed, binary |
| Over/under-fetch | Prone to both | Avoids both | Endpoint-shaped | Endpoint-shaped |
| HTTP caching | Native (GET + Cache-Control/ETag) | Hard (POST, one URL) | Hard (RPC) | N/A (binary) |
| Coupling | Loose, language-agnostic | Language-agnostic | TS-only, tight FE↔BE | Any language, needs proxy |
| Best for | Public APIs, cacheable resources | Many clients, varied data needs, aggregation | TS monorepo, internal APIs | Internal service-to-service, perf-critical |

- **Over-fetching**: REST endpoint returns more than the screen needs. **Under-fetching**: screen needs several endpoints → N round trips. GraphQL solves both by letting the client specify exactly the shape.
- **N+1**: a GraphQL resolver (or REST loop) fetching related records one-by-one per parent. Solve on the server with **DataLoader** (batch + per-request cache); on the client, watch for it in nested queries.
- **Caching implications**: REST rides free on HTTP caching (GET + ETag + CDN). GraphQL typically POSTs to one URL, defeating HTTP/CDN caching → you rely on a **normalized client cache** (Apollo/urql/Relay) and persisted queries (GET + hash) to regain edge caching. tRPC/gRPC-web similarly lean on client-side caching (e.g., React Query).
- Lead framing: paradigm choice is mostly about **who owns the contract and how many clients** — GraphQL/BFF pays off with many heterogeneous clients; tRPC shines in a single TS codebase; REST remains the default for cacheable, public, long-lived APIs.

## WebSocket Lifecycle & Scaling; SSE

- **WebSocket**: HTTP `Upgrade` handshake → open → bidirectional frames → close. You own **heartbeats (ping/pong)** to detect dead connections, **reconnect with exponential backoff + jitter**, and message **backpressure**.
- **Scaling**: connections are stateful, so either **sticky sessions** (load balancer pins a client to a node) or a **shared pub/sub backplane** (Redis/NATS/Kafka) to fan messages across nodes; watch per-node connection limits and memory. Offload to a managed service (Ably/Pusher/API Gateway WS) when fan-out is large.
- **SSE**: server→client only, plain HTTP, auto-reconnect with `Last-Event-ID`, multiplexes over H2. Simpler to scale (stateless-ish, standard HTTP infra) but one-directional and text-only.

## CORS Preflight (brief)

For "non-simple" cross-origin requests (custom headers, methods beyond GET/POST/HEAD, non-form content types), the browser sends an `OPTIONS` **preflight** first; the server must answer with `Access-Control-Allow-Origin/-Methods/-Headers`, and `Access-Control-Max-Age` caches the preflight to avoid repeating it. Credentials require `Allow-Credentials: true` and a non-wildcard origin. (Full CORS/security depth lives in the security file — here it's just: preflight adds a round trip, so cache it and avoid needless custom headers.)

## Compression

- **gzip**: universal, fast, good ratio.
- **Brotli (`br`)**: better ratio than gzip (~15–20% smaller for text), now broadly supported; prefer for text assets. Higher compression levels are slower — use max level for **static, build-time** assets and a lower level for dynamic responses.
- Negotiated via `Accept-Encoding` / `Content-Encoding`. Don't compress already-compressed formats (images, video). Verify actual transfer size vs decoded size in DevTools.

## Reliability: Retry/Backoff, Idempotency, Cancellation

- **Retry only idempotent operations** (GET, PUT, DELETE) automatically. For POST, require an **idempotency key** so a retried "create" doesn't duplicate.
- **Exponential backoff + jitter** to avoid thundering-herd retries; cap attempts; honor `Retry-After` on 429/503; don't retry 4xx (except 429).
- **`AbortController`**: cancel in-flight requests — critical for stale searches, route changes, and avoiding race conditions where a slow earlier response overwrites a newer one.

```js
async function fetchJSON(url, { signal, retries = 3 } = {}) {
  for (let attempt = 0; ; attempt++) {
    try {
      const res = await fetch(url, { signal });
      if (res.status === 429 || res.status >= 500) throw new HttpError(res.status);
      return res.json();
    } catch (err) {
      if (signal?.aborted || attempt >= retries || err.name === 'AbortError') throw err;
      const backoff = Math.min(1000 * 2 ** attempt, 8000) + Math.random() * 250; // jitter
      await new Promise(r => setTimeout(r, backoff));
    }
  }
}

// Cancel the previous search on each keystroke
let controller;
function search(q) {
  controller?.abort();
  controller = new AbortController();
  return fetchJSON(`/api/search?q=${q}`, { signal: controller.signal });
}
```

### Interview Questions — Networking

**HTTP/2 removed head-of-line blocking, yet HTTP/3 still advertises it as a headline feature. Reconcile that.**
> H2 removed head-of-line blocking at the HTTP layer by multiplexing streams over one connection, but it still rides TCP, which delivers bytes strictly in order — so a single lost packet stalls every multiplexed stream until retransmission. H3 runs over QUIC on UDP, where each stream is delivered independently, so packet loss on one stream doesn't block the others. That's the real win, alongside folded-in TLS 1.3 for near-zero-RTT setup and connection migration across networks.

**How does moving from HTTP/1.1 to HTTP/2 change your bundling strategy?**
> On H1.1, each connection carries one request at a time, so we concatenated files and sharded domains to beat the ~6-connection limit. H2 multiplexes, so those hacks become counterproductive: many small, individually-hashed chunks give better cache granularity because one changed file only busts one cache entry, and there's no per-request connection penalty. So I bundle less aggressively and lean on long-lived immutable caching per chunk. I'd verify with the request waterfall and cache-hit rates after deploys.

**Design the caching headers for a fingerprinted JS bundle vs the HTML that references it.**
> The bundle is content-hashed, so `Cache-Control: public, max-age=31536000, immutable` — cache forever, never revalidate, and a new deploy produces a new filename so there's nothing to purge. The HTML must always reflect the latest bundle names, so `no-cache` (or `max-age=0, stale-while-revalidate`) with an `ETag`, letting the browser revalidate cheaply and get a 304 when unchanged. This makes content-hash filenames the invalidation mechanism instead of active CDN purges.

**Explain `no-cache` vs `no-store` vs `stale-while-revalidate`.**
> `no-store` means never write it to any cache — for sensitive data. `no-cache` is a misnomer: it may be stored but must be revalidated with the origin before every use, typically via ETag, so you get a cheap 304 if unchanged. `stale-while-revalidate` serves the cached copy instantly, even if slightly stale, and refreshes it in the background, so the user gets an instant response and the next load is fresh — ideal for semi-dynamic content where perceived speed matters.

**REST vs GraphQL — and what does each cost you on caching?**
> REST is resource-oriented, language-agnostic, and rides HTTP caching for free through GET plus ETag and CDN edge caching, but it's prone to over- and under-fetching. GraphQL lets each client request exactly the fields it needs, killing over/under-fetching and consolidating round trips, which is great for many heterogeneous clients — but it typically POSTs to one endpoint, so you lose HTTP and CDN caching and must rely on a normalized client cache plus persisted queries to get edge caching back. I choose based on client diversity and who owns the contract, defaulting to REST for public cacheable APIs and GraphQL for a BFF serving varied clients.

**What is the N+1 problem and where do you fix it?**
> It's when resolving a list triggers one query per item for related data — one query for N parents, then N more for children. It's a server concern: I fix it with DataLoader, which batches the child lookups into one query per tick and caches within the request. On the client I watch for deeply nested queries that would provoke it and flatten or paginate. The measurement is query count per request and DB time, not just response latency.

**How do you make client-side retries safe?**
> Only auto-retry idempotent methods — GET/PUT/DELETE — with exponential backoff plus jitter, a capped attempt count, and honoring `Retry-After` on 429/503; I don't retry 4xx other than 429. For POST/creates I require an idempotency key so a retried request the server already processed returns the original result instead of creating a duplicate. That combination avoids both duplicated side effects and thundering-herd retry storms.

**A search box fires a request per keystroke and results flicker out of order. Fix it.**
> The out-of-order results are a race: a slow earlier request resolves after a newer one and overwrites it. I attach an `AbortController` to each request and abort the previous one on every keystroke, so only the latest is in flight, and I debounce input to cut request volume. Aborting also frees the connection and stops wasted work. As a backstop I can tag responses with the query and ignore any that don't match the current input.

**What's in TTFB and how do you attack a high value?**
> TTFB is DNS plus TCP plus TLS plus server processing plus the first byte's travel time. A high TTFB is usually backend think-time or a cache miss at the edge, not payload size, so I break it down in the DevTools timing panel. Fixes: cache/render at a CDN edge, add server-side caching, cut backend work, and use `preconnect` to pay DNS/TCP/TLS ahead of time for critical origins. Only after TTFB is addressed do payload-size optimizations like Brotli move the needle.
