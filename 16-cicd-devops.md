# CI/CD & Frontend DevOps

> What a lead actually owns: pipeline design, safe deploys, release strategies, monorepo build graphs, observability, and rollback — for frontend systems specifically.

## The Lead's Mental Model

A frontend lead owns the path **from merge to user**, and the ability to **reverse it fast**. Two questions frame every decision: "how do we ship confidently?" and "how do we un-ship in under a minute?" Everything below serves those two.

Frontend has a specific advantage rollback-wise: builds are **immutable static artifacts**. That makes atomic deploys and instant rollback far easier than for stateful backends — a lead should exploit this, not treat frontend deploys like server deploys.

---

## CI Pipeline Stages

Order stages **fast-and-cheap first** so feedback fails early and you do not pay for a 10-minute test run to discover a lint error.

```yaml
# Conceptual stage order (fail fast → fail slow)
stages:
  - install        # restore cache, then install
  - lint           # eslint, stylelint, prettier --check
  - typecheck      # tsc --noEmit
  - unit-test      # vitest/jest, sharded
  - build          # production bundle
  - budget         # bundle-size gate (fails PR if over)
  - a11y           # axe-core against built pages / storybook
  - e2e            # playwright against a preview deploy
```

### Gates that matter for frontend
- **Bundle-size budgets** — fail the PR when a chunk grows past a threshold. Tools: `size-limit`, `bundlesize`, or Lighthouse CI assertions. This is the single most effective guard against slow performance rot, because bundle bloat arrives one innocent import at a time.
- **Type checking** as a separate gate — never rely on the bundler to catch types; Babel/esbuild/SWC strip types without checking them. Run `tsc --noEmit`.
- **a11y checks** — `axe-core` via `@axe-core/playwright` or `jest-axe`, or Lighthouse CI's accessibility category. Catches the regressions humans miss.
- **Visual regression** (optional gate) — Chromatic/Percy/Playwright snapshots for design-system repos.

```jsonc
// .size-limit.json — gate fails the build if exceeded
[
  { "name": "app entry", "path": "dist/assets/index-*.js", "limit": "180 kB" },
  { "name": "vendor",    "path": "dist/assets/vendor-*.js", "limit": "120 kB" }
]
```

### Caching in CI
Cache to skip redundant work, keyed on lockfile hash so a dependency change busts it:

```yaml
- uses: actions/cache@v4
  with:
    path: |
      ~/.pnpm-store
      node_modules/.cache      # eslint/babel/vite caches
    key: ${{ runner.os }}-pnpm-${{ hashFiles('**/pnpm-lock.yaml') }}
    restore-keys: ${{ runner.os }}-pnpm-
```

Cache layers to consider: package store, transpile/lint caches, TypeScript incremental (`tsBuildInfoFile`), bundler cache (Vite/webpack persistent cache), and **remote build cache** for monorepos (below).

### Sharding & parallelism
Split the slowest stages across machines:
- **Test sharding:** `vitest --shard=1/4`, `jest --shard`, `playwright --shard=1/4`. Cuts wall-clock roughly linearly.
- **Matrix builds** across shards; merge coverage after.
- Balance shards by timing data (Playwright/Jest can partition by past duration) so no shard is the long pole.

| Lever | Wins | Cost / caveat |
|---|---|---|
| Cache | Skips reinstall/rebuild | Stale-cache bugs; key discipline needed |
| Shard tests | Linear wall-clock cut | More runners = more $$; merge step |
| Fail-fast order | Fast feedback | None — always do this |
| Affected-only (monorepo) | Only test what changed | Needs correct dependency graph |

---

## Build & Deploy

### Static hosting + CDN
Frontend builds are static files served from a CDN (CloudFront, Cloudflare, Netlify, Vercel, S3+CDN). Optimize for edge caching and cache correctness.

### Cache-busting with content hashes + immutable headers
The canonical strategy: **hashed asset filenames + long immutable cache; short/no cache on the HTML entry point.**

```
index.html                     Cache-Control: no-cache          (revalidate every load)
/assets/index-a1b2c3.js        Cache-Control: public, max-age=31536000, immutable
/assets/vendor-9f8e7d.css      Cache-Control: public, max-age=31536000, immutable
```

Because asset names change with content, a new deploy ships new filenames; the never-cached `index.html` points to them, so users get updates instantly with no stale-asset risk and maximal caching of unchanged chunks.

### Atomic deploys
All files for a version go live together, or none do. The failure you are preventing: `index.html` referencing `index-NEW.js` while the CDN still has only `index-OLD.js`, yielding blank screens (chunk-load errors). Platforms like Vercel/Netlify do this by design; on S3+CloudFront you deploy to a versioned prefix and flip a pointer, and you **keep old assets around** so already-loaded clients that lazy-load a chunk mid-session do not 404.

### Preview / PR deployments
Every PR gets its own immutable URL (Vercel/Netlify/Cloudflare Pages preview, or your own). This is high-leverage for a lead: design review, product sign-off, e2e targets, and stakeholder demos all run against real deployed artifacts before merge. Wire the preview URL back as a PR comment.

---

## Release Strategies

| Strategy | Mechanism | Rollback | Best for |
|---|---|---|---|
| **Blue-green** | Two identical envs; flip traffic all at once | Flip back instantly | Simple, low-risk full cutover |
| **Canary** | Route small % to new version, watch metrics, ramp | Route back to stable | Catching regressions with real traffic |
| **Progressive rollout** | Canary automated in stages (1%→10%→50%→100%) | Auto-halt on SLO breach | High-traffic, metric-gated releases |
| **Feature flags** | Ship dark, toggle per user/segment at runtime | Toggle off — no redeploy | Decoupling deploy from release |

### Feature flags (LaunchDarkly-style)
Flags decouple **deploy** (code is in production) from **release** (users see it). This is the most important lever a frontend lead has for de-risking.

```tsx
const flags = useFlags();
return flags.newCheckout ? <CheckoutV2 /> : <CheckoutV1 />;
```

- **Targeting:** by user, org, %, region — enables canary at the app layer without infra work.
- **Kill switch:** a boolean flag wrapping a risky feature; flip it off in seconds when something breaks, no rebuild, no redeploy.
- **Trade-off / debt:** flags are branches in your code. They must have owners and expiry — stale flags rot into untested dead paths and combinatorial complexity. Track flag age; delete on cleanup tickets.
- **Client caveat:** flag evaluation should not block first paint; bootstrap defaults, then hydrate. Never gate a *security* decision on a client flag — the client can flip it.

---

## Monorepo Tooling

Nx / Turborepo / pnpm workspaces exist to answer: "given a change, what is the minimum set of build/test/lint work, and can we reuse prior results?"

- **pnpm workspaces:** the package layer — symlinked local packages, one lockfile, strict/hoisted node_modules. Fast, disk-efficient (content-addressed store).
- **Task graph / affected:** Nx and Turborepo build a dependency graph of tasks and run only what a change affects — `nx affected -t test` / `turbo run test --filter=...[origin/main]`. This is what keeps CI time flat as the repo grows.
- **Remote caching:** hash task inputs (source + deps + config); if the hash was built before (locally or by CI or a teammate), replay the cached output instead of re-running. Turborepo remote cache / Nx Cloud. A cache hit turns a 5-minute build into a 2-second download.

```jsonc
// turbo.json — outputs are what gets cached & restored
{
  "tasks": {
    "build": { "dependsOn": ["^build"], "outputs": ["dist/**"] },
    "test":  { "dependsOn": ["build"], "outputs": ["coverage/**"] },
    "lint":  {}
  }
}
```

**Trade-off:** correct caching depends on declaring inputs/outputs honestly. Understated inputs → stale cache serves wrong results (the scary failure). Overstated inputs → cache never hits. Getting the graph right is the lead's job.

---

## Containerization (Frontend)

Two distinct uses:
1. **Build container** — reproducible CI builds pinned to a node version.
2. **Serve container** — ship static files behind nginx (or serve SSR). Use multi-stage builds so the runtime image contains only the artifact, not the toolchain.

```dockerfile
# Stage 1: build
FROM node:20-alpine AS build
WORKDIR /app
COPY package.json pnpm-lock.yaml ./
RUN corepack enable && pnpm install --frozen-lockfile
COPY . .
RUN pnpm build

# Stage 2: serve (tiny final image — no node, no node_modules)
FROM nginx:alpine
COPY --from=build /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
```

For pure static SPAs a CDN usually beats a container. Containers earn their place for SSR/edge runtimes, on-prem, or a uniform platform (K8s) across services.

---

## Environment Config & Secrets

**The cardinal rule: never put a secret in the bundle.** Anything bundled into client JS is public — inspectable in DevTools. `VITE_`/`NEXT_PUBLIC_` prefixes literally mean "this will be exposed to the browser."

- Client env vars hold **only public config** (public API base URLs, public analytics keys, feature-flag SDK *client* keys designed for exposure).
- Real secrets (server API keys, DB creds, private tokens) live server-side — in a BFF, edge function, or secret manager (Vault, AWS Secrets Manager, Doppler, GitHub Actions secrets) — and are injected at runtime on the server, never at build time into client code.
- Config that varies by environment (dev/staging/prod) is injected at deploy, ideally read at runtime (a fetched `/config.json` or server-rendered) so one artifact promotes across environments rather than rebuilding per env.

---

## Observability & Monitoring

### Error tracking (Sentry)
Capture unhandled exceptions, promise rejections, and React error-boundary catches with release + user + breadcrumb context. Tag every event with the **release version** so you can correlate a spike to a specific deploy.

### RUM & Core Web Vitals
Real User Monitoring beats lab metrics because it reflects actual devices/networks. Track the field CWV: **LCP, INP** (replaced FID in 2024), **CLS**. Send via `web-vitals` to your RUM/analytics; alert on p75 regressions, which is the threshold Google uses.

```ts
import { onLCP, onINP, onCLS } from "web-vitals";
[onLCP, onINP, onCLS].forEach((fn) => fn((m) => sendToRUM(m)));
```

### Source maps
Upload source maps to the error tracker at build time so stack traces are readable, but **do not serve them publicly** (they expose your source). Upload to Sentry, then delete from the deployed artifact — or restrict access. Tie the upload to the release/commit so symbolication matches the exact build.

### Alerting & SLOs
- Define **SLOs** in user terms: e.g. "p75 LCP < 2.5s", "JS error rate < 0.1% of sessions", "checkout success > 99.5%".
- Alert on **symptoms users feel** (error-rate spike, CWV regression, checkout drop), not on noise. Route to on-call; tie thresholds to the SLO's error budget.
- Wire alerts to the release so a regression alert links straight to the suspect deploy.

---

## Rollback Strategy

Frontend's superpower: static artifacts make rollback fast. Have **more than one lever**, ordered by speed:

1. **Feature flag / kill switch** — seconds, no deploy. First resort for a bad feature.
2. **Redeploy previous immutable build** — flip the CDN pointer / "promote previous deployment" (Vercel/Netlify one-click). Fast because the old artifact still exists.
3. **Revert the commit + re-release** — slowest; for when the state must actually change.

Preconditions a lead enforces: keep the last N artifacts warm on the CDN; make deploys atomic so rollback is atomic; ensure old chunks stay reachable so mid-session clients do not chunk-error during a rollback; and rehearse rollback so it is muscle memory, not improvisation during an incident.

---

## Versioning: Semantic Release & Changesets

Automate version bumps and changelogs from commit metadata so releases are boring and repeatable.

- **semantic-release:** parses Conventional Commits (`feat:`, `fix:`, `feat!:`) to auto-compute the semver bump, tag, changelog, and publish. Great for single-package repos.
- **Changesets:** contributor writes an intent file per PR declaring the bump per package; on release it aggregates them, bumps, and publishes. Purpose-built for **monorepos with independently versioned packages** (design systems, shared libs).

| | semantic-release | Changesets |
|---|---|---|
| Bump source | commit messages | explicit changeset files |
| Sweet spot | single package | monorepo, many packages |
| Human control | fully automated | contributor states intent |

Either way the win is the same: version, changelog, and tag are derived, not hand-edited — removing the most common source of release mistakes.

---

### Interview Questions — CI/CD & DevOps

**Walk me through how you'd order CI stages for a frontend app, and why.**

> Fastest and cheapest checks first so feedback fails early: install (with cache restore), then lint, then typecheck, then unit tests, then build, then the build-dependent gates — bundle-size budget, a11y, and e2e against a preview deploy. The principle is fail-fast: I never want to burn a ten-minute test run to surface a lint error I could have caught in fifteen seconds. Typecheck is its own gate because bundlers strip types without checking them, so `tsc --noEmit` is the only thing actually verifying types. I also gate on bundle size because performance rots one innocent import at a time, and a budget is the cheapest way to hold the line.

**Explain the cache-busting strategy you'd use for static assets on a CDN.**

> Content-hashed filenames plus a split caching policy. Every JS/CSS asset gets a hash in its name and `Cache-Control: max-age=31536000, immutable`, so browsers and the CDN cache it effectively forever. The HTML entry point gets `no-cache` so it revalidates on every load. Because asset names change only when content changes, a deploy ships new filenames, the always-fresh HTML points at them, and users get updates immediately with zero stale-asset risk while every unchanged chunk stays cached. The critical companion is keeping old assets live after deploy so a client that lazy-loads a chunk mid-session during a rollout does not hit a 404.

**What's the difference between a feature flag and a canary deploy, and when would you use each?**

> A canary is infrastructure-level: you route a small percentage of traffic to a new *version* and ramp up while watching metrics. A feature flag is application-level: the new code is deployed to everyone, but a runtime toggle controls who actually *sees* it. Flags decouple deploy from release, give me per-user/segment/percentage targeting without touching infra, and — crucially — give me a kill switch that reverts in seconds with no redeploy. I use canary/progressive rollout for whole-version risk and infra changes, and flags for feature-level risk and fast reversibility. Often both together: deploy dark, then progressively enable via flag.

**How does affected-based execution and remote caching keep monorepo CI fast?**

> Tools like Nx and Turborepo build a task dependency graph, so on a given change they run only the tasks a change actually affects rather than the whole repo — that keeps CI time roughly flat as the repo grows. Remote caching hashes each task's inputs (source, dependencies, config) and stores its outputs; if that hash was already built — locally, by CI, or by a teammate — it replays the cached output instead of re-running, turning a multi-minute build into a couple-second download. The catch, and my job as lead, is declaring inputs and outputs honestly: understate inputs and you serve stale wrong results; overstate them and you never hit the cache.

**Why must secrets never be in the frontend bundle, and how do you handle keys the app needs?**

> Anything bundled into client JavaScript is public — it ships to the browser and is readable in DevTools, so a "secret" in the bundle is just a published secret. The `VITE_`/`NEXT_PUBLIC_` prefixes literally mean "expose to browser." So client env vars hold only genuinely public config. Anything real — server API keys, third-party secrets — lives server-side in a BFF or edge function backed by a secret manager, and the browser calls our own endpoint that attaches the key server-side. I also prefer runtime config (a fetched config or server-rendered values) over build-time injection so one artifact promotes across environments instead of rebuilding per env.

**A deploy causes an error spike. Walk me through your rollback options in order.**

> Fastest first. If the culprit is a specific feature behind a flag, I flip the kill switch — seconds, no deploy. If it's broader, I promote the previous immutable build / flip the CDN pointer back, which is fast precisely because frontend artifacts are immutable and the old one still exists. Only if state actually has to change do I revert the commit and re-release, which is the slowest path. This works because I've set preconditions up front: atomic deploys, the last N artifacts kept warm, old chunks reachable so mid-session clients don't chunk-error during the rollback, and a rehearsed procedure so it's muscle memory during an incident rather than improvisation.

**How do you handle source maps for a production frontend?**

> I generate them and upload them to the error tracker at build time, tied to the release and commit so stack traces symbolicate against the exact build — otherwise every Sentry trace is minified garbage. But I do not serve them publicly, because a public source map hands out your source. So the flow is: build with source maps, upload to Sentry, then strip them from the deployed artifact or lock down access. Tagging every error event with the release version is what lets me correlate a spike to the specific deploy that caused it.

**What metrics would you put SLOs on for a frontend, and how do you alert?**

> User-felt symptoms, in field data. Core Web Vitals at p75 — LCP under 2.5s, INP under 200ms (INP replaced FID in 2024), CLS under 0.1 — plus a JS error rate per session and business-critical funnel success like checkout completion. I measure these with RUM, not lab tools, because real devices and networks are what users experience. Alerts fire on symptoms tied to the SLO's error budget — an error-rate spike, a p75 CWV regression, a funnel drop — not on noisy internal signals, and each alert links back to the suspect release so on-call goes straight from symptom to likely cause.
