# The Staff-Level Grill

> This file is deliberately harsh. Treat every question as if a skeptical staff engineer is looking for the crack in your reasoning, not the textbook definition. Rules for using it: **answer out loud, from memory, before reading the answer.** For every output-prediction snippet, commit to an answer before scrolling. If you get one wrong, don't just read the fix — say out loud *why* your mental model produced the wrong answer, because that's the gap an interviewer will keep probing. Budget ~90 minutes for a full pass; do it once cold, then again the night before.

---

## Round 1 — JavaScript: Output Prediction & Why

No credit for the right answer without the right reasoning — a real interviewer will ask "why" immediately after your guess.

**Q1.**
```js
console.log('1');
setTimeout(() => console.log('2'), 0);
Promise.resolve().then(() => console.log('3'));
console.log('4');
```
> **`1 4 3 2`.** Synchronous code runs to completion first (`1`, `4`). Then the microtask queue drains completely (`3`) before the event loop pulls the next macrotask off the timer queue (`2`). The trap: people say `1 2 3 4` or `1 4 2 3` — microtasks always fully drain before the next macrotask, no interleaving.

**Q2.**
```js
async function foo() {
  console.log('foo start');
  await null;
  console.log('foo end');
}
console.log('script start');
foo();
console.log('script end');
```
> **`script start`, `foo start`, `script end`, `foo end`.** Everything in `foo` before the first `await` runs synchronously, immediately, when `foo()` is called — that's the trap people miss, they assume `async` defers the whole function. `await` (even `await null`) suspends and schedules the continuation as a microtask, so `script end` runs before `foo end`.

**Q3.**
```js
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 0);
}
```
> **`3 3 3`.** `var` has function/global scope — one shared binding for the whole loop. By the time any callback fires, the loop has finished and `i` is `3`. Swap `var` for `let` and you get `0 1 2`, because `let` creates a fresh binding scoped to each iteration. Know the pre-`let` fix too: `setTimeout((i) => console.log(i), 0, i)` or wrap the body in an IIFE capturing `i` by value.

**Q4.**
```js
const obj = {
  name: 'staff',
  regular: function () { return this.name; },
  arrow: () => this.name,
};
const { regular, arrow } = obj;
console.log(obj.regular(), regular(), obj.arrow(), arrow());
```
> **`'staff'`, `undefined` (or throws in strict mode), `undefined`, `undefined`.** `this` in a regular function is determined by *how it's called*, not where it's defined — `obj.regular()` binds `this` to `obj`, but destructuring `regular` out and calling it bare loses that binding entirely (`this` is `undefined` in strict/module code, the global object otherwise). Arrow functions never have their own `this` — they close over the lexical `this` at *definition* time, which here is the module/top-level scope, not `obj`, so both `obj.arrow()` and `arrow()` give the same (wrong, if you expected `obj`) answer.

**Q5.**
```js
console.log([] + []);
console.log([] + {});
console.log(1 + '1');
console.log('5' - 3);
console.log('5' + 3);
console.log([1, 2] + [3, 4]);
console.log(0.1 + 0.2 === 0.3);
```
> `''`, `'[object Object]'`, `'11'`, `2`, `'53'`, `'1,23,4'`, `false`. The array/object cases both coerce via `.toString()` first (`[].toString()` → `''`, `{}.toString()` → `'[object Object]'`) then concatenate — `+` prefers string concatenation the instant either operand isn't cleanly numeric. `-` has no string-concat meaning, so it always coerces to number. The float comparison is IEEE-754 binary floating point — `0.1 + 0.2` is `0.30000000000000004`; the fix is an epsilon comparison or integer/cents-based math for money, never raw float equality.

**Q6.**
```js
console.log(typeof null);
console.log(typeof undefined);
console.log(typeof NaN);
console.log(NaN === NaN);
console.log(Object.is(NaN, NaN));
console.log([1] == '1');
console.log(null == undefined);
console.log(null === undefined);
```
> `'object'` (a 25-year-old language bug, `null` is a primitive, not an object — never "fixed" because it would break the web), `'undefined'`, `'number'` (NaN is a numeric value, just an invalid one), `false` (NaN is never equal to itself under `==`/`===`), `true` (`Object.is` uses SameValue, which treats NaN as equal to itself — the one place it differs from `===`), `true` (`[1]` coerces to `'1'` then loose-equals the string), `true` (the spec special-cases `null`/`undefined` as loosely equal to each other and nothing else), `false` (different types, `===` never coerces).

**Q7.**
```js
function foo() {
  return
  {
    staff: true
  };
}
console.log(foo());
```
> **`undefined`.** Automatic Semicolon Insertion inserts a semicolon immediately after `return` because a newline follows it, so the function returns nothing and the object literal becomes unreachable dead code (a block, actually, that's just discarded). This is *the* canonical ASI trap — the fix is `return {` on the same line as `return`, which is why most style guides forbid a line break there.

**Q8.**
```js
console.log(typeof myVar);
var myVar = 5;

try {
  console.log(typeof myLet);
} catch (e) {
  console.log(e.constructor.name);
}
let myLet = 5;
```
> **`'undefined'`, then `'ReferenceError'`.** Both `var` and `let` are hoisted to the top of their scope, but differently: `var` is hoisted *and initialized* to `undefined`, so reading it early is safe (if confusing). `let`/`const` are hoisted but left in the **Temporal Dead Zone** — the binding exists but touching it before the declaration line throws. This is the precise mechanical answer, not just "let is block-scoped."

**Q9.**
```js
class Counter {
  #count = 0;
  increment() { this.#count++; return this.#count; }
}
const c = new Counter();
console.log(c.increment());
console.log(c.count); // ?
console.log(Object.keys(c)); // ?
```
> **`1`**, then `undefined` (private fields aren't accessible from outside, not even readable — this isn't convention like `_count`, it's enforced by the engine), then `[]` (private fields don't show up in `Object.keys`, `for...in`, or `JSON.stringify` — true encapsulation, unlike the old closure-based-private-field workaround which at least had this same property but at the cost of no shared prototype methods).

**Q10.** — memory leak spotting
```js
function attachHandler(el) {
  const bigData = new Array(1_000_000).fill('x');
  el.addEventListener('click', () => {
    console.log(bigData.length);
  });
}
```
> Every call to `attachHandler` creates a new closure that captures `bigData`, and that closure is now referenced by the DOM element's listener list. As long as `el` is reachable (attached to the document, or referenced from JS), `bigData` cannot be garbage collected — even though the click handler only needs `bigData.length`, which could have been captured as a lone number instead of keeping the entire million-element array alive. This is the realistic version of "closures cause memory leaks": it's not the closure itself, it's *what it unnecessarily keeps alive*. The fix: capture only the primitive you need, or explicitly `removeEventListener` when `el` is torn down, or use a `WeakRef`/`WeakMap` if you must associate large data with a DOM node's lifetime.

---

## Round 2 — TypeScript: Type-System Grilling

**Q1. Why does this print `string[] | number[]` instead of `(string | number)[]`, and how would you get the latter?**
```ts
type ToArray<T> = T extends any ? T[] : never;
type Result = ToArray<string | number>; // string[] | number[]
```
> Conditional types **distribute over naked union type parameters** — `T extends any ? T[] : never` applied to `string | number` runs the conditional once per union member and unions the results, giving `string[] | number[]`. To suppress distribution and get `(string | number)[]`, wrap both sides in a tuple so `T` is no longer "naked": `type ToArray<T> = [T] extends [any] ? T[] : never`. This exact distributive-vs-non-distributive distinction is one of the most commonly misunderstood parts of TS's type system, and a very fair staff-level question.

**Q2. What does `infer` do here, and what's the return type of `ReturnTypeOf<() => Promise<string>>`?**
```ts
type ReturnTypeOf<T> = T extends (...args: any[]) => infer R ? R : never;
```
> `infer R` introduces a type variable that TypeScript solves for by pattern-matching the function's return position against the call — it's how `ReturnType<T>` (a built-in utility type) is implemented under the hood. For `() => Promise<string>`, `R` is inferred as `Promise<string>` — **not** `string`. This is the trap: `infer` captures the type as written, it doesn't unwrap a `Promise`. To get `string` you'd need a second conditional: `T extends (...args: any[]) => Promise<infer R> ? R : never`.

**Q3. Why does this compile without error, and is that a bug?**
```ts
interface Point { x: number; y: number; }
function log(p: Point) { console.log(p.x, p.y); }
const obj = { x: 1, y: 2, z: 3 };
log(obj); // ?
```
> It compiles fine — TypeScript is **structurally typed**: `obj` satisfies `Point` because it has *at least* the required shape, extra properties on a variable are fine. This is not a bug, it's the whole point of structural typing (vs. nominal typing in Java/C#). The trap is the companion rule: `log({ x: 1, y: 2, z: 3 })` — passing an **object literal directly** — *does* error, because TypeScript runs **excess property checking** only on literals assigned directly to a typed slot, precisely to catch typos like `{ x: 1, y: 2, zz: 3 }` that structural typing alone would let through silently. Know both halves, because "TypeScript is structural" alone gives an incomplete, wrong-sounding answer to the literal case.

**Q4. What's the difference in strictness between these two, and why does it matter for callback safety?**
```ts
interface A { method(x: string): void; }        // method syntax
interface B { method: (x: string) => void; }    // property syntax
```
> Under `strictFunctionTypes`, method-syntax function types (`A`) are checked **bivariantly** (parameters can vary in either direction, unsound but pragmatic for method overriding), while property-syntax function types (`B`) are checked **contravariantly** (correctly strict — a subtype's method parameter must accept at least as much as the supertype expects). Practically: `B` will reject an unsound parameter-widening assignment that `A` lets through silently. This is *why* well-typed callback props on components/interfaces are often written in property syntax rather than method syntax — it's a real, if obscure, soundness hole that trips people up when overriding methods on class hierarchies.

**Q5. What does this template literal type produce, and what real use case does it solve?**
```ts
type EventName<T extends string> = `on${Capitalize<T>}`;
type ClickEvent = EventName<'click'>; // ?
```
> `'onClick'`. `Capitalize` is a built-in intrinsic string-manipulation type; template literal types let you compute string types the same way you'd compute string values. Real use: deriving a strongly-typed props interface for event handlers from a list of event names, so `{ onClick: () => void; onHover: () => void }` is generated rather than hand-written, and a typo in an event name becomes a compile error instead of a silent no-op handler.

**Q6. Is this `any` or `unknown`, and why is the distinction load-bearing?**
```ts
function parse(json: string) {
  return JSON.parse(json); // return type?
}
```
> `JSON.parse` is typed to return `any` in the standard lib — meaning `parse`'s return type is inferred as `any`, which **disables type checking on every downstream use** of the result. This is a real, common source of runtime bugs disguised as "TypeScript app" — the moment `any` enters, it's contagious and silently propagates through anything it touches. The fix at a staff level: wrap with a runtime validator (Zod/Valibot) that returns a properly narrowed type, or at minimum cast to `unknown` and narrow explicitly, since `unknown` forces you to prove the shape before using it — the type-safe counterpart to `any`.

---

## Round 3 — React: Internals & Bleeding-Edge Features

**Q1. Explain what a "Fiber" actually is, precisely — not "it's how React is fast."**
> A Fiber is a JS object representing a unit of work for one component instance — it holds the component type, pending props, current state, a pointer to its return/child/sibling fibers, and an "alternate" pointer linking it to its previous-render counterpart (the double-buffering that makes an in-progress render abortable without corrupting the visible tree). React walks this fiber tree as a linked list rather than recursing the component tree directly, specifically so the work loop can **pause, checkpoint, resume, or abandon** work between fibers — which a plain recursive call stack cannot do. That's the actual mechanical reason Fiber exists: it converts rendering from an unstoppable synchronous recursion into an interruptible, resumable unit-of-work queue, which is the prerequisite for everything concurrent rendering does afterward.

**Q2. `useTransition` vs `useDeferredValue` — most people can name both but not say precisely when each is correct.**
> `useTransition` marks the **state update itself** as low-priority — you call `startTransition(() => setState(...))` around the thing causing expensive work, and React can interrupt that render for higher-priority updates (a keystroke) and show a pending indicator via `isPending`. `useDeferredValue` instead wraps a **value you're consuming**, letting the UI show a stale/previous version of that value while React computes the fresh one in the background, without you controlling the state update that produced it. Rule of thumb: **you own the setState call** → `useTransition`; **you're a consumer of a value someone else set, and can't wrap the setter** → `useDeferredValue` (e.g., filtering a huge list from a search input controlled by a parent, or a value only observable via a hook/context). Reaching for the wrong one is a very common "I know the APIs but not the model" tell.

**Q3. Why does Suspense-for-data-fetching require throwing a promise, and what's the actual contract?**
> A component reading not-yet-resolved data throws the in-flight promise itself (this is what libraries like React Query/Relay do internally); the nearest `Suspense` boundary above catches it, and React's contract is: **when that specific promise resolves, retry rendering the subtree from scratch** — not resume where it left off. That "retry from scratch" detail is the trap — it means the component function up to the suspend point must be safe to re-run with no side effects that shouldn't happen twice, which is exactly why ad-hoc `throw somePromise` in raw component bodies is fragile and why the ecosystem converged on hooks/libraries that manage the caching so the retry gets a *resolved* value instantly instead of re-fetching.

**Q4. What can and cannot happen inside a React Server Component, and why does that boundary exist?**
> RSCs render on the server (or at build time) and ship **zero JS for themselves** to the client — only their rendered output (a special serialized format, not HTML strings) crosses the wire. Consequence: no `useState`, no `useEffect`, no event handlers, no browser APIs — anything stateful/interactive must live in a `'use client'` component, and the boundary is one-directional: a Server Component can render a Client Component and pass it serializable props, but a Client Component cannot import a Server Component (it can only receive one as `children`/props already rendered). The reason this exists at all: it lets you ship a data-heavy page with near-zero client JS for the parts that are pure presentation, while still hydrating only the genuinely interactive islands — a much finer-grained version of what SSR + full hydration used to force on the whole tree.

**Q5. What problem does `useSyncExternalStore` solve that plain `useState` + a subscription couldn't?**
> Under concurrent rendering, React can render the same component multiple times at different priorities, or render a tree partway and pause. If a component reads from an external store (Redux, a custom store, browser APIs like `window.innerWidth`) via a naive subscription + `useState`, the store can mutate *between* those renders, so different parts of the tree being rendered "concurrently" can observe different values of the same store — a **tearing** bug that's invisible until concurrent features are actually in play. `useSyncExternalStore` is React's officially sanctioned integration point: it guarantees the snapshot returned is consistent across a whole render pass and forces a synchronous re-render if the store changes mid-render, closing that tearing hole. If you've ever wondered why library authors don't just tell you to `useState` + `useEffect(() => store.subscribe(...))`, this is exactly why — that pattern is what tears.

**Q6. Diagnose: two tabs of the same list swap their input values when an item is deleted from the middle.**
```tsx
{items.map((item, i) => <Row key={i} item={item} />)}
```
> Classic key-by-index bug. React uses `key` to match a fiber across renders to the *same conceptual element*, not the same array slot. Using the array index as `key` means that after deleting item 2 of 5, what was rendered as index 3 is now rendered as index 2 — React sees "the element at key=2 still exists," assumes it's the *same* component instance (including any internal state like an uncontrolled input's DOM value or a `useState` inside `Row`), and reuses it rather than unmounting/remounting — so state that belonged to the deleted row's neighbor now appears attached to the wrong data. The fix is a **stable, unique identifier from the data itself** (`item.id`), so a deletion correctly unmounts exactly the row that was deleted and every other row keeps its own state attached to the right data.

**Q7. Why does this `useEffect` log a stale value, and what are the two ways to fix it — and their trade-off?**
```tsx
function Timer() {
  const [count, setCount] = useState(0);
  useEffect(() => {
    const id = setInterval(() => {
      console.log(count); // always logs 0
      setCount(count + 1);
    }, 1000);
    return () => clearInterval(id);
  }, []); // empty deps
  return <div>{count}</div>;
}
```
> The effect runs once (empty deps), so the closure created inside it captures `count` as it was on that first render — `0` — forever; the interval callback never sees a newer `count` because it's not a new closure, it's the same one from mount. **Fix 1**: use the updater-function form, `setCount(c => c + 1)`, which doesn't need to read `count` from the closure at all — sidesteps the staleness entirely and lets you keep empty deps. **Fix 2**: add `count` to the dependency array, which correctly re-creates the interval (clearing and re-setting it) every time `count` changes — correct, but means you're tearing down and restarting a `setInterval` every tick, which is wasteful for something like a simple counter. The updater-function form is the better answer here specifically because the effect doesn't actually need to *read* state, only update it — recognizing that distinction is the actual signal of understanding, not just knowing "add it to deps."

---

## Round 4 — CSS: The Edge Cases That Actually Bite

**Q1. A `float`ed child isn't contained by its parent — the parent's height collapses to zero. What's happening, precisely, and name three ways to fix it that each work for a different reason.**
> Floated elements are removed from **normal flow**, so a parent whose height is determined only by normal-flow content sees no content and collapses. Fixes, each establishing containment differently: (1) `overflow: hidden`/`auto` on the parent — creates a **Block Formatting Context**, and a BFC's box computation includes floated descendants' extents; (2) a `::after { content: ""; display: table; clear: both; }` clearfix — explicitly clears past the floats inside the parent's own flow; (3) `display: flow-root` — the modern, side-effect-free way to create a BFC purely for containment, without `overflow`'s side effect of clipping/scrollbars. Knowing *why* each one works (BFC vs explicit clearing) rather than just "add overflow hidden" is the staff-level bar here.

**Q2. `@layer` reorders the cascade — does it also change paint order (stacking context z-order)? Justify precisely.**
> No — `@layer` is purely a **cascade** concept (which *declaration* wins when multiple rules match the same element), it has nothing to do with the **paint/stacking** algorithm (which *element* paints in front of which). A rule in a later layer can win the cascade for, say, `color`, while stacking context order for `z-index` is governed entirely by the separate stacking-context rules (position, opacity, transform, etc.) regardless of which `@layer` declared the `z-index`. Conflating "layer" (cascade grouping) with "stacking context" (paint ordering) despite the similar name is exactly the trap — they solve unrelated problems that happen to share the word "layer" colloquially.

**Q3. What's the difference between `container-type: inline-size` and `container-type: size`, and why is `size` dangerous?**
> `inline-size` lets you query the container's inline-axis (width, in horizontal writing modes) size, and — critically — it does **not** contain block-axis layout, so the container's height can still depend on its content normally, with no circularity risk. `size` lets you query both width and height, but it also forces both-axis **containment**, meaning the container's own size can no longer depend on its content's size (since the content's layout might depend on the container's size — a circular dependency the spec resolves by forcing the container to have an explicit/definite size from outside, e.g. a fixed height). Using `size` on a container whose height is meant to be intrinsic (content-driven) either silently collapses it or requires you to set an explicit height you didn't want to set — the practical reason `inline-size` is the default choice and `size` is a deliberate, narrow-use escape hatch.

**Q4. `aspect-ratio: 16/9` is set on an `<img>` alongside a `width: 100%`, but the image still looks squished before it loads. Why, and what's the full fix?**
> `aspect-ratio` reserves layout space based on the ratio, which correctly prevents layout shift once the image *has* dimensions to compute against — but if the intrinsic dimensions aren't known yet (or the browser's per-image aspect-ratio inference from `width`/`height` attributes isn't present) and no explicit `height` is resolvable, the box can still end up computed oddly before the real image asserts its own ratio, and separately, `aspect-ratio` alone does **not** control how the image's content fills that box — that's `object-fit`'s job. The full, correct pattern is `width: 100%; aspect-ratio: 16/9; object-fit: cover;` plus setting the `width`/`height` HTML attributes on the `<img>` tag itself (not just CSS) so the browser can reserve the correct space from the very first paint, before any CSS or the image bytes have even arrived — this last part is the detail people most often skip.

**Q5. Explain, mechanically, why `position: sticky` "just stops working" inside a card that has `overflow: hidden` for rounded-corner clipping.**
> `sticky` positions an element relative to its nearest **scrolling ancestor** (or the viewport if none), staying within that ancestor's scroll range. Any ancestor with `overflow` other than `visible` — including `hidden`, not just `scroll`/`auto` — becomes a **scroll container** in the spec's sense (it establishes a scrollable overflow region even if it's not user-scrollable), and `sticky`'s scroll range gets clipped to that ancestor's bounds. If that ancestor's bounds are the same height as the sticky element's own content, there's no scroll range left for "sticky" to visibly do anything within — it looks broken but is behaving exactly per spec. Fix: don't put `overflow: hidden` on the actual scrolling ancestor of a sticky element; clip the rounded corners a different way (a wrapping element that isn't in the sticky element's scroll-ancestor chain, or `clip-path` instead of `overflow: hidden`).

---

## Round 5 — Performance: Under Interrogation

**Q1. Break down INP into its three phases, precisely, and name one lever for each.**
> **Input delay** — time from the user's interaction to the browser starting to handle it, usually because the main thread is busy with something else (a long task already running). Lever: break up existing long tasks so the thread is free to respond quickly (`scheduler.yield()`, chunking work, `isInputPending()` checks). **Processing time** — the actual event handler and any synchronous work it triggers, including the resulting re-render. Lever: move genuinely expensive computation off the main thread (a Web Worker) or defer non-urgent parts of it (`startTransition`). **Presentation delay** — time from processing finishing to the next frame actually painting the result, which includes layout, paint, and compositing. Lever: avoid forcing synchronous layout recalculation (layout thrashing) and keep the DOM changes from the interaction as small/targeted as possible. Naming all three phases unprompted, not just "reduce your JS," is the actual bar.

**Q2. Diagnose this pattern and name the fix with its correct name.**
```js
items.forEach(item => {
  item.el.style.height = computeHeight(item) + 'px'; // write
  console.log(item.el.offsetHeight);                  // read — forces layout
});
```
> **Layout thrashing** (forced synchronous layout). Writing a style invalidates layout; reading a geometry property (`offsetHeight`, `getBoundingClientRect`, etc.) forces the browser to synchronously recompute layout *right then* to answer accurately, rather than batching it into the next natural layout pass — and because this write-then-read pattern repeats in a loop, you pay a full synchronous layout recalculation on every single iteration instead of one at the end. The fix is batching: do all the reads first (into a local array), then do all the writes — `items.map(item => computeHeight(item))` then a second pass applying `.style.height`. Tools like FastDOM formalize this batching; the underlying principle is "never read geometry immediately after a write in the same tick."

**Q3. A component leaks memory every time it mounts/unmounts in a SPA (heap grows monotonically). Walk through your actual diagnostic process, not just the likely cause.**
> I wouldn't guess first — I'd take two heap snapshots in DevTools, one after the component's been mounted/unmounted several times, and diff them (the "Comparison" view), filtering for objects whose count grows proportionally to the mount/unmount cycles. That tells me *what* is retained, and DevTools' retainer tree tells me *why* — usually one of: an event listener added on a global (`window`/`document`) or a long-lived singleton service that was never removed on unmount, a `setInterval`/`setTimeout` never cleared, a subscription (WebSocket, RxJS, an external store) never unsubscribed, or a closure captured by any of the above holding a reference to the whole component's scope (and therefore everything else it closed over). Only after the snapshot diff confirms the actual retained object type would I go fix the specific missing cleanup — guessing "it's probably the event listener" and patching blindly is how you fix a symptom while leaving the actual leak.

**Q4. Your team wants to code-split every route into its own chunk for "better performance." When do you push back?**
> Splitting is a trade-off between two costs: unused-code bytes (fewer, bigger chunks) versus per-request overhead and cache-fragmentation (more, smaller chunks) — HTTP/2/3 multiplexing makes the request-count cost much smaller than it used to be, but it's not zero, and every additional chunk is also an additional cache-invalidation unit (a shared-component change busts every chunk that inlined it, if the boundaries are drawn badly). I'd push back on *route-per-chunk as a blanket rule* when routes share a lot of common component/logic weight — that shared code either duplicates across chunks (bytes go up) or forces a separate shared vendor chunk (which needs its own cache-busting story). The actual lead move is profiling real navigation patterns (which routes are visited together, what's actually shared) and splitting along *that* boundary, with a bundle-size budget in CI as the objective check rather than "more splitting is always better."

---

## Round 6 — Architecture & System Curveballs

**Q1. "2% of users are randomly seeing stale data after a deploy." You have no repro steps. Where do you actually start?**
> I'd narrow "random" first — is it every user occasionally, or a consistent 2% of users every time? If it's a *consistent* subset, that smells like a CDN edge-cache or a specific origin/PoP serving an old cached response (check `Age`/`X-Cache` response headers for those users' region), or a service worker that precached an old asset manifest and hasn't been told to update — both produce "always these people, always stale" rather than true randomness. If it's genuinely random across the whole population intermittently, that smells like a race in the client's own data layer — a stale-while-revalidate cache (React Query/SWR) serving cached data while a background revalidation is still in flight, or (in a multi-instance backend) a load balancer round-robining between an old and new deployed version during a rolling deploy, so *which* origin server you hit determines what you get for the few minutes both versions are live. I'd instrument before guessing further — log the served asset/response version alongside the user session so the pattern (consistent subset vs. true random) tells me which of these branches to chase.

**Q2. Module Federation lets Team A and Team B each ship an independently-deployed MFE, and each depends on React. What actually breaks if you get this wrong, mechanically?**
> If each MFE bundles its own copy of React instead of sharing a single instance, you can end up with **two separate React module instances loaded on the same page** — and React's internal dispatcher (the thing that makes hooks work) is tied to a specific module instance. A component from MFE A rendering a component from MFE B (or sharing any context across the boundary) can trip **"Invalid hook call" or context values silently not matching**, because the two React copies don't share internal state even though they're "the same version." The fix is declaring React (and ReactDOM) as a **singleton shared dependency** in the Module Federation config, so all federated modules resolve to one loaded instance — and pinning/aligning versions across teams, because a singleton share of *mismatched* major versions is its own failure mode (whichever loads first typically wins, silently, for everyone).

**Q3. Angular-flavored curveball: an `OnPush` component doesn't re-render when a child pushes an item into an array `@Input`. Why exactly, and what's the fix?**
> `OnPush` decides whether to re-check a component based on whether an `@Input`'s **reference** changed, not its contents. `array.push(item)` mutates the existing array in place — same reference, new contents — so Angular's `OnPush` check (`===` on the input) sees no change and skips re-rendering that subtree entirely, even though the data genuinely changed. This is the exact same discipline React enforces via "never mutate state directly" — the mechanism differs (dirty-checking a reference vs a render+diff) but the required fix is identical: treat the array as immutable and assign a new reference, `this.items = [...this.items, item]`, so the reference-equality check actually detects the change.

---

## Round 7 — Rapid Fire (10-Second Answers, No Partial Credit)

1. **`Object.freeze` vs `const` — what does each actually prevent?** `const` prevents *reassigning the binding*; the object it points to is still fully mutable. `Object.freeze` prevents mutating the object's own properties (shallow only — nested objects are still mutable).
2. **Debounce vs throttle, one sentence each.** Debounce delays execution until input stops for N ms (search-as-you-type); throttle guarantees execution at most once per N ms regardless of how often the event fires (scroll/resize handlers).
3. **What does `Array.prototype.sort()` do to `[10, 1, 2]` with no comparator, and why?** `[1, 10, 2]` — default sort coerces to strings and compares lexicographically; always pass a numeric comparator for numbers.
4. **Why is `for...in` wrong for iterating an array?** It iterates enumerable *keys*, including inherited/non-index ones, in no guaranteed order, and includes any custom enumerable properties added to the array or its prototype — use `for...of`, `.forEach`, or a classic index loop.
5. **What's a WeakMap for, that a Map can't do?** Keys are weakly held — if the only reference to a key object disappears elsewhere, the entry is garbage-collected automatically, making it safe for associating metadata with objects (like DOM nodes) without creating a memory leak.
6. **CSS: what makes an element a "replaced element," and why does it matter?** Its content is rendered from outside the CSS formatting model (`<img>`, `<video>`, `<iframe>`, form controls) — replaced elements have intrinsic dimensions and don't support all the same CSS (e.g., `::before`/`::after` don't render on most of them, and `object-fit` only applies to them).
7. **What does `will-change` actually cost if left on permanently?** Forces the browser to keep a separate compositor layer for that element indefinitely — memory and (on enough elements) compositing overhead, for zero benefit once the animation is over.
8. **`Promise.all` vs `Promise.allSettled` — when does the choice matter?** `all` short-circuits and rejects on the *first* rejection, losing the other results; `allSettled` always resolves with every outcome (fulfilled or rejected) — use `allSettled` when partial failure among independent requests is acceptable and you need every result regardless.
9. **What's a "dangling" `useEffect` cleanup bug in one sentence?** Forgetting to guard an async operation started in an effect against the component having unmounted before it resolves, causing a `setState` call (or worse, a DOM mutation) on an unmounted component.
10. **Why does `<script defer>` differ from `<script async>`?** Both download in parallel without blocking parsing; `async` executes the instant it's downloaded (out of order relative to other scripts, potentially before parsing finishes), `defer` waits until parsing is complete and executes scripts in document order — `defer` is almost always what you want for app scripts that depend on the DOM.

---

## Round 8 — Hold Your Ground (Pressure Round)

These are written as an adversarial back-and-forth. Read the pushback, then answer it as if the interviewer is actually skeptical of you, not just prompting you.

**"You said you'd use `useTransition` here, but honestly, isn't that just over-engineering when we could just optimize the render?"**
> Both are correct in different orders. If the render is genuinely wasteful — unnecessary re-renders, unmemoized expensive computation — I fix that first, because `useTransition` doesn't make slow work fast, it only deprioritizes it. But once the underlying computation is already necessary and irreducible (filtering 50k rows on every keystroke, say), `useTransition` isn't a substitute for optimization, it's what you reach for *after* you've confirmed the work itself can't be cheaper — it changes *when* React does unavoidable work relative to more urgent input, not *how much* work exists. So I'd profile first; if the flame graph shows the component itself is doing avoidable work, that's the actual fix, and I'd say so rather than reflexively reaching for a concurrent API to paper over an unoptimized render.

**"Our coverage is 55%. Why shouldn't I just mandate 90% across the board next sprint?"**
> Because a mandate on the raw percentage optimizes for the number, not for confidence — the fastest way to hit 90% is padding with assertion-light tests on already-safe code, which actively wastes the sprint without reducing real risk. I'd counter with what I'd actually do: identify the highest-risk, currently-undertested modules (checkout, auth, payment logic) and set a *targeted* floor there, while gating CI on coverage *regression* elsewhere so nothing gets worse. If leadership wants a number for a dashboard, I'd rather report critical-path coverage and mutation-testing scores on the modules that matter than a blended average that hides exactly where the real gaps are.

**"You're telling me not to use Redux, but this app clearly has global state — isn't that just you avoiding boilerplate?"**
> Not avoiding boilerplate, avoiding unnecessary coupling. Having *some* global state doesn't imply it all needs one store with one set of conventions — server data (anything from an API) is a different problem than UI state (a sidebar's open/closed) and conflating them under one Redux store is exactly why so many Redux codebases end up with a `isLoading` boolean duplicated for every single resource. I'd split it: TanStack Query (or equivalent) owns server-state caching/revalidation, and genuinely cross-cutting *client* state — the stuff that isn't naturally owned by one component subtree — goes into a lightweight store (Zustand, or Context for something small and infrequent-changing). If, after that split, what's left over is large and complex enough to need Redux's explicit action/reducer discipline and time-travel debugging, then yes, use it — the pushback isn't "never Redux," it's "prove the complexity is real before adopting the heaviest tool for it."

**"Why should we trust a component test over a real E2E test — doesn't E2E prove it actually works?"**
> E2E proves the *integrated system* works at the moment you ran it, but it proves the least per minute of CI time and per hour of maintenance of any layer — it's slow, it's the most likely to be flaky (network, timing, third-party dependencies), and a failure tells you *something* broke without telling you *what*, so you're debugging from a bigger haystack. A component test that renders the real component, drives it with real user-event interactions, and mocks only the network boundary catches the same class of wiring bug — did the click actually call the right handler, does the error state actually render — in milliseconds, with a failure that points directly at the component. I'm not arguing E2E is worthless — I keep a thin layer of it specifically for the paths where only a real browser, real network stack, and real integrated system can catch the bug (a payment redirect flow, cross-origin behavior). But "more E2E is always more confidence" ignores that confidence-per-minute is the actual metric that determines whether the suite stays fast enough that people don't start skipping it.
