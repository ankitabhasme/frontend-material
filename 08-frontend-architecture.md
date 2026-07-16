# Frontend Architecture — HLD & LLD

> System-level and component-level design: topologies, state, patterns, data layer, and the cross-cutting concerns a lead must raise proactively.

## HLD vs LLD

- **HLD (High-Level Design)** — system-level: rendering strategy, app topology (SPA/MPA/micro-frontends), state management approach, API/data layer, module boundaries, deployment/CDN, cross-cutting concerns (auth, i18n, observability, security, a11y). Answers *"how do the big pieces fit and communicate?"*
- **LLD (Low-Level Design)** — component/module-level: component APIs and props contracts, folder structure, state shape, hooks/utilities, data models/types, error/loading states, specific patterns. Answers *"how is this feature implemented cleanly?"*

## A Framework for Answering "Design X"

1. **Clarify requirements** — functional + non-functional (scale, users, devices, SEO, offline, latency, compliance).
2. **Constraints & assumptions** — team size, existing stack, browser support, timeline.
3. **HLD** — topology, rendering strategy, data flow, state, API contract, caching, deployment.
4. **Cross-cutting** — **security, accessibility**, performance, i18n/l10n, observability, error handling, testing.
5. **LLD** — component tree, state shape, key interfaces/types, folder structure.
6. **Trade-offs** — justify choices; call out what you'd revisit at scale.

## Application Topologies
- **SPA** — one app, client routing. Great app-like UX; SEO/first-load caveats (mitigate with SSR).
- **MPA** — server-routed pages; simpler, SEO-friendly; heavier navigations.
- **Micro-frontends (MFE)** — independent teams ship independent deployables composed into one app.
  - **Composition:** build-time (packages), server-side (edge/SSI), or **runtime** (Module Federation, import maps, iframes).
  - **Webpack Module Federation** — share code/deps at runtime across independently deployed apps.
  - **Trade-offs:** team autonomy + independent deploys vs. duplicated deps, version skew, consistency/UX drift, complex shared state and routing, larger total payload. Adopt only when org scale justifies it.
- **Monorepo** (Nx, Turborepo, pnpm workspaces) — shared tooling, atomic cross-package changes, cached builds; needs good CI and ownership boundaries.

## State Management (LLD-heavy)
- **Server state vs client state** — the key distinction. **Server state** (remote, async, cached, can go stale) → **TanStack Query, SWR, RTK Query, Apollo**. Handles caching, revalidation, dedup, retries. **Client/UI state** → local `useState`, Context, or a store.
- **Client state libraries:** Redux Toolkit (predictable, devtools, middleware, large apps), Zustand (minimal, hooks, selectors), Jotai/Recoil (atomic), XState (state machines), MobX (reactive).
- **Choosing:** don't reach for Redux by default. Prefer local → lifted → context → external store, and separate server-state caching from UI state. Over-globalizing state is a common anti-pattern.

## Component Architecture & Patterns (LLD)
- **Composition over inheritance.**
- **Presentational vs container** — "how it looks" vs "how it works."
- **Compound components** — `<Tabs><Tab/></Tabs>` sharing implicit state via context; flexible APIs.
- **Custom hooks** — extract and reuse stateful logic.
- **Render props / headless components** — logic without UI (Radix, TanStack Table) → reusable + accessible + style-agnostic.
- **Design system / component library** — tokens, primitives, documented (Storybook), themable, accessible by default. Lead concerns: consistency, governance, versioning, adoption.
- **Feature-based folder structure** over type-based for scale:
  ```
  src/
    features/
      cart/ (components, hooks, api, state, types, tests)
      checkout/
    shared/ (ui, hooks, utils, lib)
    app/ (routing, providers, layout)
  ```

## Data & API Layer
- REST vs GraphQL vs tRPC; **BFF (Backend-for-Frontend)** to shape data per client and reduce over/under-fetching.
- Contract stability, typed clients (OpenAPI/GraphQL codegen), optimistic updates, error/retry policy, pagination/infinite scroll, cancellation (AbortController).

## Cross-cutting Concerns (a lead must proactively raise these)
- **Security & Accessibility** — dedicated files.
- **Testing strategy** — unit, component (RTL — test behavior), integration, E2E (Playwright/Cypress), visual regression, a11y (axe). Weight toward integration ("testing trophy").
- **Observability** — error tracking (Sentry), RUM/Core Web Vitals, logging, feature flags, source maps.
- **CI/CD** — lint/type-check/test/build gates, preview deployments, progressive rollout, bundle-size budgets in CI.
- **Internationalization** — locale, RTL, formatting, message extraction, avoid concatenation.
- **Resilience** — error boundaries, graceful degradation, offline (service workers/PWA), retry/backoff.
- **DX & governance** — TypeScript, ESLint/Prettier, conventions, ADRs, code ownership.

## Example: Design a large e-commerce listing + PDP (talking points)
- **HLD:** SSR/streaming for SEO + fast LCP on PDP; CDN + ISR for catalog; BFF to aggregate pricing/inventory/reviews; TanStack Query for server-state caching; minimal client state (cart via store, persisted).
- **Perf:** image CDN + AVIF + `srcset`, virtualized/paginated listing, route-split, preload PDP hero.
- **Security:** sanitize reviews (XSS), CSP, auth via httpOnly cookies, rate-limit search.
- **A11y:** semantic product cards, accessible filters (announce results), focus management on route change, contrast on price/badges.
- **LLD:** `ProductCard` (presentational), `useProductFilters` hook, normalized cart state, typed API client, error/empty/loading states, compound `Filters`.

---

### Interview Questions — Architecture

**Q1. Difference between HLD and LLD, and what belongs in each?**
> HLD is system-level (topology, rendering strategy, data flow, state approach, deployment, cross-cutting concerns). LLD is implementation-level (component APIs, state shape, folder structure, patterns, types). HLD answers how big pieces fit; LLD answers how a feature is built cleanly.

**Q2. When would you choose micro-frontends, and what are the costs?**
> When multiple independent teams need autonomous deploys and ownership at org scale. Costs: dependency duplication/version skew, UX/consistency drift, harder shared state/routing, larger payloads, and operational complexity. For a single team, a modular monolith/monorepo is usually better.

**Q3. How do you manage state in a large app?**
> Separate server state (TanStack Query/RTK Query — caching, revalidation) from client/UI state. For client state, prefer local → lifted → context → a store with selectors (Zustand/Redux Toolkit). Avoid over-globalizing; colocate state; use state machines for complex flows.

**Q4. Context re-render problem and solutions?**
> All consumers re-render on any value change. Solutions: split contexts by concern, memoize the value, use selector-based subscriptions (`use-context-selector`) or an external store with selectors.

**Q5. How do you enforce quality across a growing frontend team?**
> TypeScript + strict lint/format, a documented design system, feature-based structure, CI gates (types/tests/bundle budgets/a11y), code review standards, ADRs, Storybook, and a testing strategy weighted toward integration tests, plus observability to catch field regressions.

**Q6. CSR vs SSR vs SSG vs ISR — how do you choose?**
> By content freshness, SEO, and TTFB/LCP needs: SSG/ISR for mostly-static, cacheable pages (marketing, catalog); SSR/streaming for dynamic, SEO-critical, per-request pages (PDP, dashboards); CSR for authenticated app shells where SEO doesn't matter. Increasingly RSC + streaming to cut client JS.
