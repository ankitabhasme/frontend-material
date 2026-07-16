# Accessibility (a11y)

> WCAG, semantic HTML, ARIA, keyboard & focus management, contrast, and screen readers — building inclusive UIs by default.

## WCAG & POUR
- **WCAG 2.1/2.2**, conformance levels **A / AA / AAA** (aim for **AA**).
- **POUR principles:** **Perceivable, Operable, Understandable, Robust.**

## Semantic HTML first
- Use the right element: `<button>`, `<a>`, `<nav>`, `<main>`, `<header>`, `<ul>`, `<label>`, headings in order. Semantics give free keyboard + screen-reader behavior. **A `<div onClick>` is not a button** (no focus, no Enter/Space, no role).

## ARIA (only when needed)
- **First rule of ARIA: don't use ARIA if a native element works.**
- `role`, `aria-label`/`aria-labelledby`, `aria-describedby`, `aria-expanded`, `aria-hidden`, `aria-live` (announce dynamic updates: `polite`/`assertive`), `aria-current`.
- Follow **WAI-ARIA Authoring Practices (APG)** patterns for widgets (menu, combobox, tabs, dialog).

## Keyboard accessibility
- Everything interactive must be **operable by keyboard** (Tab, Shift+Tab, Enter/Space, Arrow keys for composite widgets, Esc to close).
- **Visible focus indicators** (don't remove outlines without a replacement; prefer `:focus-visible`).
- **Focus management:** move focus into a modal on open, **trap focus** within it, restore focus to the trigger on close. On SPA route change, move focus to the new page heading and announce it.
- **Logical tab order**; avoid positive `tabindex`. Use `tabindex="-1"` for programmatic focus, `tabindex="0"` to add custom elements to tab order.
- **Skip link** ("skip to main content").

## Visual & content
- **Color contrast** — text ≥ **4.5:1** (normal), **3:1** (large text / UI components).
- **Don't rely on color alone** to convey meaning (add text/icons).
- **Text resize / zoom to 200%**, responsive reflow.
- **Alt text** for images; empty `alt=""` for decorative. Captions/transcripts for media.
- **Labels** for every form control; associate errors (`aria-describedby`), announce validation.
- Respect **`prefers-reduced-motion`**.

## Screen readers & testing
- Screen readers: NVDA, JAWS (Windows), VoiceOver (Mac/iOS), TalkBack (Android).
- **Automated:** axe-core / `jest-axe`, Lighthouse, `eslint-plugin-jsx-a11y` (catches ~30–50% of issues).
- **Manual:** keyboard-only pass, actual screen-reader testing, zoom, contrast checks. Automation is necessary but not sufficient.

## React specifics
- `htmlFor` (not `for`), `className`, managing focus with refs, `React.forwardRef` for reusable interactive components, portals for modals (with correct focus/aria), announce async updates via live regions, ensure lazy-loaded/route-changed content is announced.

---

### Interview Questions — Accessibility

**Q1. How do you build an accessible modal in React?**
> Render in a portal; on open, move focus into it and trap focus; set `role="dialog"` + `aria-modal="true"` + `aria-labelledby`; close on Esc and overlay click; on close, restore focus to the trigger; make background inert/`aria-hidden`. Follow the APG dialog pattern.

**Q2. When should you use ARIA?**
> Only when native HTML can't express the semantics/state — e.g., custom widgets (tabs, combobox) needing roles/states like `aria-expanded`. Prefer native elements first; incorrect ARIA is worse than none.

**Q3. How do you handle focus on SPA route changes?**
> After navigation, programmatically move focus to the main heading (or a focus target) with `tabindex="-1"`, and announce the page via a live region, since there's no full page reload for screen readers to detect.

**Q4. Key WCAG AA requirements you always check?**
> Sufficient color contrast (4.5:1 text), full keyboard operability with visible focus, semantic structure/labels, alt text, don't-rely-on-color, resiliency to 200% zoom, and accessible names/roles/states for widgets.

**Q5. How do you make accessibility part of the process, not an afterthought?**
> Semantic-HTML-first component library with a11y baked into primitives, `eslint-plugin-jsx-a11y`, automated axe tests in CI, a11y acceptance criteria in tickets, keyboard/screen-reader checks in review, and periodic manual audits.
