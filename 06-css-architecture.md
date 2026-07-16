# CSS & Styling Architecture

> The parts of CSS that decide correctness and performance at scale — cascade, stacking, layout engines, containment — and choosing a styling strategy as a deliberate architecture decision.

## Box Model

Every element is content + padding + border + margin. The `box-sizing` property decides what `width` means.

- **`content-box`** (default): `width` = content only; padding and border add on top → actual rendered width is larger than declared.
- **`border-box`**: `width` includes content + padding + border → the number you write is the box's outer size. This is what almost every design system wants.

```css
*, *::before, *::after { box-sizing: border-box; } /* the standard reset */
```

Margins **collapse** vertically between adjacent/nested block elements (the larger wins, not the sum) — a frequent source of "phantom" spacing. Flex/grid items don't collapse margins.

## Cascade, Specificity, Inheritance

When multiple rules match, the winner is decided in order:

1. **Origin & importance**: (from weakest) user-agent → user → author → author `!important` → user `!important` → transitions/animations at the top for active states.
2. **Cascade layers** (`@layer`) — see below.
3. **Specificity**: `(inline=1000, IDs=100, classes/attrs/pseudo-classes=10, elements/pseudo-elements=1)`. `!important` and inline styles sit above this ladder.
4. **Source order**: last one wins on a tie.

- **Inheritance**: text-ish properties (`color`, `font`, `line-height`, `visibility`) inherit; box properties (`margin`, `border`, `background`) don't. Force it with `inherit`, `initial`, `unset`, `revert`.
- `:where()` has **zero specificity** — the lead's tool for shippable, easily-overridable base styles. `:is()` takes the specificity of its most specific argument.

| Selector | Specificity |
|---|---|
| `#nav a.active` | 1,1,1 |
| `.btn.primary` | 0,2,0 |
| `:where(.btn) a` | 0,0,1 |
| `li::before` | 0,0,2 |

## Cascade Layers (`@layer`)

`@layer` lets you order groups of styles **independent of specificity and source order**. Later-declared layers win over earlier ones, regardless of selector strength — so a low-specificity override in a later layer beats a high-specificity rule in an earlier layer. This is how you tame specificity wars between reset / framework / components / utilities.

```css
@layer reset, framework, components, utilities;

@layer framework {
  .card #title { color: gray; } /* high specificity... */
}
@layer utilities {
  .text-black { color: black; }  /* ...but this wins: later layer */
}
```

Unlayered styles win over all layered styles — a subtle gotcha when migrating.

## Stacking Context & z-index

A stacking context is a self-contained z-axis world; `z-index` only competes **within the same context**. A child in a lower stacking context can never paint above a sibling of its parent, no matter how large its `z-index`.

**What creates one**: root `<html>`; `position` + `z-index` (not `auto`); `opacity < 1`; `transform`, `filter`, `perspective`, `clip-path`, `mask`; `will-change` of a stacking property; `isolation: isolate`; flex/grid children with `z-index`; `position: fixed/sticky`.

- **Classic pitfall**: a modal with `z-index: 9999` still hides behind a header because an ancestor with `opacity: 0.99` or `transform` created a trapping context. Fix by restructuring the DOM/portals, not by escalating z-index.
- **`isolation: isolate`** creates a context without side effects — the clean way to sandbox a component's stacking.

## Flexbox vs Grid

| | Flexbox | Grid |
|---|---|---|
| Dimensionality | 1D (row *or* column) | 2D (rows *and* columns) |
| Content vs layout driven | Content-driven (items size themselves) | Layout-driven (you define the grid) |
| Best for | Toolbars, nav, chip rows, distributing along one axis | Page/section layout, card galleries, overlapping items |
| Gaps | `gap` supported | `gap`, plus `grid-template-areas` |

Rule of thumb: **Grid for the two-dimensional skeleton, Flex for one-dimensional runs of content inside it.** They compose — a grid cell can be a flex container.

```css
.layout { display: grid; grid-template-columns: 240px 1fr; gap: 1rem; }
.toolbar { display: flex; gap: .5rem; align-items: center; }
/* auto-responsive card grid, no media queries */
.cards { display: grid; grid-template-columns: repeat(auto-fill, minmax(220px, 1fr)); gap: 1rem; }
```

## Positioning

- `static` (default), `relative` (offset from its normal spot; establishes containing block for absolute children), `absolute` (out of flow, positioned to nearest positioned ancestor), `fixed` (to viewport, or to a transformed ancestor — a common surprise), `sticky` (relative until a scroll threshold, then fixed within its scroll container).
- `sticky` silently fails if an ancestor has `overflow: hidden/auto` or the element has no defined `top`/`bottom`.

## Container Queries vs Media Queries

- **Media queries** respond to the **viewport** — good for global layout shifts.
- **Container queries** respond to a **parent element's** size — the component adapts to wherever it's placed, which is what you actually want for reusable components in a design system.

```css
.card-wrap { container-type: inline-size; container-name: card; }
@container card (min-width: 400px) {
  .card { grid-template-columns: 120px 1fr; }
}
```

Also: `@container style()` queries and container query units (`cqi`, `cqw`) for truly component-relative sizing.

## Logical Properties

Physical (`margin-left`) breaks under RTL/vertical writing modes; logical properties follow the writing direction:

- `margin-inline-start` / `-end`, `padding-block`, `inset-inline`, `border-inline`, `text-align: start`.

Use logical properties by default in any internationalized product — one stylesheet works for LTR and RTL without mirroring.

## Custom Properties (Theming & Design Tokens)

Runtime, cascading, inheritable variables — unlike Sass vars they're live in the DOM and readable/writable from JS.

```css
:root {
  --space: 8px;
  --color-bg: white; --color-fg: #111;
}
[data-theme="dark"] { --color-bg: #111; --color-fg: #eee; }
.panel { background: var(--color-bg); color: var(--color-fg); padding: calc(var(--space) * 2); }
```

- **Theming**: flip a `data-theme` attribute and the whole tree re-themes with no re-render and no extra CSS — cheaper than swapping stylesheets or class trees.
- Scope tokens to components for local overrides; expose a semantic layer (`--color-surface`) over a primitive layer (`--blue-500`).
- `@property` registers a custom property with a type, enabling **animatable** variables and typed fallbacks.

## Responsive & Fluid Design

- **`clamp(min, preferred, max)`** for fluid type/spacing without breakpoints.
- `min()` / `max()` for adaptive constraints.

```css
h1 { font-size: clamp(1.5rem, 1rem + 3vw, 3rem); }
.container { width: min(100% - 2rem, 72ch); margin-inline: auto; }
```

Prefer intrinsic sizing (`min-content`, `max-content`, `fit-content`, `auto-fill`) and fluid values over cascades of media-query breakpoints — fewer magic numbers, fewer breakpoints to maintain.

## Performance: Critical CSS, Containment, will-change

- **Critical CSS**: inline the above-the-fold styles in `<head>`, load the rest asynchronously, so first paint isn't blocked on the full stylesheet. Extract with tooling (Critters/Beasties, `critical`) and re-measure FCP/LCP.
- **`contain`**: promises the browser that an element's layout/paint/style is self-contained, letting it skip work outside the box. `contain: layout paint` isolates reflow scope; `contain: content` is a common shorthand. Huge win for lists/cards where one item's change shouldn't reflow the page.
- **`content-visibility: auto`** (+ `contain-intrinsic-size`): skip rendering of off-screen subtrees entirely until near the viewport — dramatic wins on long pages. Verify with rendering time in the Performance panel.
- **`will-change`**: pre-promotes an element to its own layer before an animation. Use sparingly and remove after — persistent `will-change` wastes GPU memory and can *hurt* (layer explosion).

## Styling Strategy as an Architecture Decision

The choice is a trade-off across runtime cost, DX, theming, SSR, and bundle size — not taste.

| Approach | Runtime cost | Scoping | Theming | SSR | Notes |
|---|---|---|---|---|---|
| **Plain CSS / BEM** | None | Manual (naming discipline) | Custom props | Trivial | Scales only with rigor; specificity drift |
| **CSS Modules** | None (build-time hashing) | Automatic, local by default | Custom props | Trivial | Great default; static, no dynamic props at runtime |
| **CSS-in-JS runtime** (styled-components, Emotion) | **Yes** — serialize/inject on render | Automatic | Full JS access, dynamic | Needs SSR extraction; hydration cost | Ergonomic, colocated; runtime tax + can break React Server Components |
| **Zero-runtime CSS-in-JS** (Vanilla Extract, Linaria) | None (extracted at build) | Automatic | Typed tokens / contracts | Trivial | Best of both: colocation + no runtime; less dynamic |
| **Utility / Tailwind** | None (build-time, purged) | N/A (atomic) | Config tokens | Trivial | Tiny shared CSS, fast; verbose markup, design-system config is the contract |

Decision guidance for a lead:
- **Default to CSS Modules or a zero-runtime solution** for most product apps — you get scoping and theming with zero runtime and clean SSR.
- **Tailwind** when you want a token-constrained system, minimal shipped CSS, and fast iteration; accept markup verbosity and enforce shared components to avoid class soup.
- **Runtime CSS-in-JS** only when highly dynamic, prop-driven styling justifies the cost — and know it fights React Server Components and adds hydration/serialize overhead. Measure the runtime cost (styles injected per render) before adopting at scale.
- **Theming** cross-cuts all of them: build on CSS custom properties so themes are runtime and don't require rebuilding component trees.
- **SSR**: anything build-time-extracted is trivial; runtime CSS-in-JS needs a critical-CSS extraction step and risks flash-of-unstyled-content and hydration mismatches.

## Accessibility-Relevant CSS

```css
/* keyboard focus without punishing mouse users */
:focus-visible { outline: 2px solid var(--focus); outline-offset: 2px; }

/* respect motion sensitivity */
@media (prefers-reduced-motion: reduce) {
  *, *::before, *::after { animation-duration: .001ms !important; transition-duration: .001ms !important; }
}

/* honor OS theme */
@media (prefers-color-scheme: dark) { :root { color-scheme: dark; } }
```

- Never remove focus outlines without a visible replacement; prefer `:focus-visible` over `:focus`.
- Don't hide content from AT with `display:none` if it should be read; use a visually-hidden (`.sr-only`) pattern for screen-reader-only text.
- Maintain WCAG contrast on text; test tokens in both themes. Respect `prefers-reduced-motion` for all non-essential animation and `prefers-reduced-data`/`prefers-contrast` where relevant.

### Interview Questions — CSS Architecture

**A modal with `z-index: 99999` still renders behind the header. Diagnose it.**
> `z-index` only competes within the same stacking context, so an ancestor of the modal has created a context that traps it below the header's context. The usual culprits are an ancestor with `opacity < 1`, a `transform`, or a `filter`. Escalating z-index won't help. The fix is to render the modal in a portal at the document root so it's not nested inside the trapping context, or restructure so both share a context. I'd inspect the "Layers" / stacking context tooling to confirm which ancestor created it.

**When do you reach for Grid vs Flexbox, and can you state it as one rule?**
> Grid for two-dimensional layout where I define rows and columns — page skeletons, card galleries, overlap; Flexbox for one-dimensional runs of content that size themselves — toolbars, nav, chip rows. The one rule: Grid for the layout scaffold, Flex for content flow inside it, and they compose since a grid cell can itself be a flex container.

**How do cascade layers change the way you fight specificity?**
> `@layer` orders groups of rules independent of specificity and source order — a later layer beats an earlier one even if the earlier rule has higher specificity. So I declare an explicit order like reset, framework, components, utilities, and utilities reliably win with low-specificity selectors. It replaces `!important` wars and deep selectors. The gotcha is that unlayered styles beat all layered ones, so during migration everything should move into layers.

**Walk me through choosing a styling strategy for a large, themeable product app.**
> I weigh runtime cost, scoping, theming, and SSR. My default is CSS Modules or a zero-runtime solution like Vanilla Extract: automatic scoping, no runtime tax, trivial SSR, and typed tokens. I build theming on CSS custom properties so a `data-theme` flip re-themes the tree with no re-render. I avoid runtime CSS-in-JS unless styling is highly dynamic and prop-driven, because it serializes and injects styles per render, complicates SSR extraction, and fights React Server Components. If the team wants a constrained token system with minimal shipped CSS, Tailwind, enforced through shared components.

**How do container queries change component design vs media queries?**
> Media queries key off the viewport, so a component's responsiveness breaks when you drop it into a narrow sidebar. Container queries key off the component's own container size, so it adapts to wherever it's placed. For a design system that's the correct primitive — components become truly portable. I set `container-type: inline-size` on the wrapper and query it, optionally using container query units for internal sizing.

**What CSS levers do you have for rendering performance on a long, heavy page?**
> `content-visibility: auto` with `contain-intrinsic-size` to skip rendering off-screen subtrees, `contain: layout paint` to bound reflow/paint scope so one card's change doesn't reflow the page, and critical CSS inlined so first paint isn't blocked. For animation I promote layers with `will-change` only during the animation and animate `transform`/`opacity`. I verify each with the Performance panel's rendering/layout timings and FCP/LCP, not by feel.

**Why `:focus-visible` over `:focus`, and what else respects user preferences?**
> `:focus` shows an outline even on mouse click, which designers strip out and thereby break keyboard accessibility; `:focus-visible` only shows the ring when the browser heuristically detects keyboard navigation, so I get accessible focus without punishing mouse users. I also honor `prefers-reduced-motion` to near-disable animation, `prefers-color-scheme` for theming, and set `color-scheme` so form controls match.

**Explain specificity precisely and how `:where()` and `:is()` differ.**
> Specificity is a tuple of inline, IDs, classes/attributes/pseudo-classes, and elements/pseudo-elements, compared left to right, with source order breaking ties and `!important`/inline sitting above the ladder. `:is()` takes the specificity of its most specific argument, which can surprise you; `:where()` always contributes zero specificity, so I use it for base styles that must be trivially overridable. That zero-specificity property is what makes `:where()` safe in a shared framework layer.
