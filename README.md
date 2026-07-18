# Lead Frontend Developer — Interview Prep

> A complete, lead-level study guide. Each file holds deep notes plus interview questions with model answers. Emphasis throughout: **trade-offs** and **"how I'd measure/verify it"** — that's what separates a senior from a lead.

## How to use this

- Study **one file per session**; each is self-contained.
- For every technical answer, add the **trade-off** and **how you'd verify/measure it**.
- Build a **story bank** (see [18](18-behavioral-leadership.md)) mapping real experiences to leadership competencies — behavioral is the biggest differentiator for lead.
- Do at least a few **timed machine-coding** ([10](10-machine-coding.md)) and **system-design** ([09](09-frontend-system-design.md)) mock runs out loud.

## Contents

### Core technical depth
1. [React Under the Hood](01-react-internals.md) — reconciliation, Fiber, render/commit, concurrent, hooks internals
2. [JavaScript & TypeScript](02-javascript-typescript.md) — event loop, closures, `this`, prototypes, promises, GC; TS generics & types
3. [Build Tooling](03-build-tooling.md) — npm/SemVer/lockfiles, modules, Webpack, Babel, Vite
4. [Performance Optimization](04-performance.md) — Core Web Vitals, rendering, bundle/network/asset, measurement

### Platform & fundamentals
5. [Browser & Web Platform](05-browser-web-platform.md) — rendering pipeline, storage, workers, service workers, observers
6. [CSS & Styling Architecture](06-css-architecture.md) — cascade, flex/grid, stacking, styling-strategy decision, Figma → production handoff
7. [Networking](07-networking.md) — HTTP/1.1–3, caching, REST/GraphQL/tRPC, WebSockets/SSE

### Design & delivery
8. [Frontend Architecture (HLD & LLD)](08-frontend-architecture.md) — topologies, state, patterns, cross-cutting concerns
9. [Frontend System Design](09-frontend-system-design.md) — answering framework + 6 worked examples
10. [Machine Coding](10-machine-coding.md) — the practical build round, with implementations
11. [DSA (Frontend-flavored)](11-dsa-frontend.md) — Big-O + classic FE utility implementations
12. [Testing](12-testing.md) — trophy vs pyramid, RTL, MSW, E2E (Playwright/Cypress), Cucumber/Gherkin BDD, a11y testing, CI gates

### Cross-cutting & role
13. [Security](13-security.md) — XSS, CSRF, CORS, CSP, auth, supply chain
14. [Accessibility](14-accessibility.md) — WCAG, semantic HTML, ARIA, focus, contrast
15. [Design Patterns](15-design-patterns.md) — GoF + React idioms + SOLID for components
16. [CI/CD & DevOps](16-cicd-devops.md) — pipelines, release strategies, monorepo, observability, PCF/Cloud Foundry deployment
17. [AI/LLM Integration](17-ai-llm-integration.md) — streaming UIs, keys/proxy, prompt injection, chat UX
18. [Behavioral & Leadership](18-behavioral-leadership.md) — STAR, competencies, story bank, question bank

### Dual-framework & final gauntlet
21. [Angular & NgRx](21-angular-ngrx.md) — components/DI/change detection, RxJS flattening operators, NgRx vs Redux, Angular vs React trade-offs. **Read this if the JD lists Angular alongside React** — it's the biggest gap the rest of this guide doesn't cover.
22. [The Staff-Level Grill](22-staff-level-grill.md) — the hardest version of this interview: JS/TS output-prediction traps, React internals & bleeding-edge features, nasty CSS edge cases, performance under interrogation, architecture curveballs, and a "hold your ground under pushback" round. Do this **last**, after 01–18, as a final stress test, not a first pass.
23. [The Mastercard JD Extended Grill](23-mastercard-jd-extended-grill.md) — a condensed, 12-round sequel to 22 covering the JD gaps at rapid-fire pace: Angular internals, design systems/Figma/UI-UX, TDD/BDD/Cypress/Playwright/Cucumber, Jenkins/PCF, accessibility, cross-framework state, build tooling, cross-browser/mobile, Agentic AI, and an ownership/mentorship pressure round. Good for a fast second pass.
24. [The Final Boss — Mastercard Grill](24-mastercard-final-boss-grill.md) — the deepest version: 18 rounds, same JD-gap coverage as 23 but with full code snippets, output-prediction traps, and payments-specific framing on every answer (money-math precision, PCI scope, 3-D Secure iframes). Overlaps with 23 in topic, not in depth — treat 23 as the quick pass and 24 as the exhaustive one. Do this **last**, cold, in one long sitting.

### Reference appendix
25. [Custom Hooks Library — "Polyfills"](25-react-custom-hooks-polyfills.md) — the popular usehooks.com/react-use-style hook set (storage, timers, DOM/browser bindings, state helpers) implemented from scratch, with the gotcha each one exists to fix. Companion to [10](10-machine-coding.md)'s `useDebounce`/`useFetch`/store hooks.
26. [Polyfilling React's Own Hooks](26-polyfill-core-react-hooks.md) — `useState`/`useReducer`/`useRef`/`useEffect`/`useMemo`/`useCallback` implemented from scratch via a slots-array + cursor (the "build your own React hooks" whiteboard question), with `useDebounce`/`useThrottle` composed on top to show they're free once the primitives exist. Pairs with [01](01-react-internals.md#hooks-internals)'s fiber-linked-list explanation.

### Backend & full-stack
For full-stack JDs requiring Java/Spring/Microservices alongside React, see the companion guide at [../../Backend/interview-prep/](../../Backend/interview-prep/README.md) — it has its own 2-day crash plan that pulls in specific files from this guide (React, testing, security, CI/CD) where they overlap.

## Suggested 2-week plan

| Days | Focus |
|---|---|
| 1–2 | React internals (01) + JS/TS (02) — the fundamentals screens |
| 3 | Build tooling (03) + Performance (04) |
| 4 | Browser (05) + CSS (06) + Networking (07) |
| 5–6 | Architecture (08) + System Design (09) — practice out loud |
| 7 | Machine coding (10) — timed builds |
| 8 | DSA (11) + Testing (12) |
| 9 | Security (13) + Accessibility (14) |
| 10 | Design patterns (15) + CI/CD (16) + AI/LLM (17) |
| 11–12 | Behavioral (18) — write and rehearse your story bank |
| 13–14 | Full mock interviews; revisit weakest areas |

If the JD requires Angular alongside React, insert [21](21-angular-ngrx.md) right after day 1–2. Reserve [22](22-staff-level-grill.md), [23](23-mastercard-jd-extended-grill.md), and [24](24-mastercard-final-boss-grill.md) for the final day or two before the interview — they're stress tests, not study material for a first pass.

## The lead mindset (thread this through every answer)
- **Trade-offs, not absolutes** — every choice has costs; name them.
- **Measure before optimizing** — data first, then act, then verify.
- **Raise cross-cutting concerns proactively** — security, accessibility, performance, testing, observability.
- **Think in systems and teams** — maintainability, governance, onboarding, and how the org scales.
- **Own outcomes** — quantify impact with metrics.
