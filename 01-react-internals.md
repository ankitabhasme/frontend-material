# React Under the Hood

> Reconciliation, Fiber, render vs commit, concurrent features, and hooks internals — the "how React actually works" round.

## The Big Picture

React is a **declarative UI library**. You describe *what* the UI should look like for a given state; React figures out *how* to update the DOM to match. Three concerns:

- **Reconciliation** — the *algorithm* that diffs the new element tree against the previous one to decide what changed.
- **Rendering** — calling your components to produce React elements (a description of UI).
- **Committing** — applying the computed changes to the actual host (DOM, native, etc.).

Key mental model: **Rendering is not the same as updating the DOM.** "Render" = React calls your function. "Commit" = React mutates the DOM. A component can render many times without a single DOM mutation.

## Virtual DOM

- The **Virtual DOM (VDOM)** is a lightweight in-memory JS object tree representing the UI. A React element is just `{ type, props, key, ref }`.
- On state change, React builds a *new* VDOM tree and diffs it against the previous one (reconciliation), producing a minimal set of real DOM mutations.
- Why: direct DOM manipulation is expensive (layout/reflow/repaint). Batching diffs into minimal mutations is cheaper than naive re-rendering.
- **Nuance:** the VDOM is not magically "fast" — it's an abstraction that lets you write declaratively while React optimizes writes. A hand-tuned imperative update can be faster; the VDOM optimizes for developer productivity + "good enough" performance.

## The Reconciliation Algorithm

A general tree-diff is O(n³). React uses heuristics to get O(n):

1. **Different element types → full subtree rebuild.** `<div>` → `<span>` throws away the old subtree (unmount + mount), losing its state.
2. **Same type → diff props, keep the DOM node**, recurse on children.
3. **Lists use `key`** to match children across renders. Without stable keys, React matches by index — causing bugs (wrong state kept, unnecessary re-renders, input focus loss).

### Why keys matter (classic interview trap)
```jsx
// Anti-pattern: index as key on a reorderable list
{items.map((item, i) => <Row key={i} data={item} />)}
```
If you prepend an item, every index shifts. React thinks item 0's *content* changed rather than an item was inserted → re-renders everything, and any local state (e.g., an uncontrolled input) sticks to the wrong row. Use a **stable, unique id**: `key={item.id}`.

## React Fiber

**Fiber** (React 16+) is a complete rewrite of the reconciler. Before Fiber, reconciliation was a synchronous recursive call ("stack reconciler") — once it started, it couldn't be interrupted, blocking the main thread on large trees.

### What a Fiber is
A **fiber node** is a plain JS object representing a unit of work — one per component instance / DOM node. It holds:
- `type`, `key`, `stateNode` (the actual DOM node or class instance)
- `child`, `sibling`, `return` (parent) → a **linked list**, not a recursive tree. This lets React walk the tree with a loop and *pause/resume* at any node.
- `pendingProps` / `memoizedProps`, `memoizedState` (the hooks linked list lives here)
- `alternate` — pointer to the fiber in the other tree (see double buffering)
- `flags` (effect tags: Placement, Update, Deletion), `lanes` (priority)

### Two phases
1. **Render / Reconcile phase (interruptible)** — React builds the *work-in-progress* fiber tree, calling components and computing diffs. This is **async and can be paused, aborted, or restarted**. Because it can be thrown away, **render must be pure and side-effect-free**.
2. **Commit phase (synchronous, not interruptible)** — React applies all mutations to the DOM in one go, then runs lifecycle/effects. Sub-phases: before-mutation, mutation, layout.

### Double buffering
React keeps two trees: **current** (on screen) and **work-in-progress** (being built). When WIP is complete, React swaps the pointer (`current = workInProgress`) in one atomic step. `alternate` links the two so React can reuse fiber objects instead of allocating fresh ones.

### Scheduling & Lanes
- Fiber enables **time-slicing**: break work into chunks, yield to the browser (via a scheduler using `MessageChannel`) so high-priority work (user input) isn't blocked.
- **Lanes** (React 17+) are a bitmask priority model. Different updates get different lanes (discrete input = high, transitions = low). React batches and processes lanes by priority. Replaced the older `expirationTime` model.

## Concurrent React (React 18)

- **Concurrent rendering** is opt-in via concurrent features, not a global mode.
- **Automatic batching** — React 18 batches state updates even across `await`/promises/timeouts/native events (previously only within React event handlers).
- `startTransition` / `useTransition` — mark updates as **non-urgent (transitions)** so urgent updates (typing) interrupt them. UI stays responsive during heavy re-renders.
- `useDeferredValue` — defer re-rendering a value; shows stale content while an expensive render happens in the background.
- **Suspense** — declaratively wait for async (data/code) and show a fallback. React can pause a component's render, show fallback, and resume. Combined with `React.lazy` for code splitting and with data frameworks for streaming SSR.
- **`useId`** — stable, SSR-safe unique IDs (avoids hydration mismatches).

## Hooks Internals

- Hooks are stored as a **linked list on the fiber's `memoizedState`**, in call order. This is why **hooks must be called unconditionally in the same order** — React matches them by position, not name.
- `useState` → creates a hook object with a state value + update queue. Dispatch enqueues an update; React re-renders and replays the queue.
- `useEffect` vs `useLayoutEffect`:
  - `useLayoutEffect` runs **synchronously after DOM mutation, before paint** — use for DOM measurement to avoid flicker. Blocks paint.
  - `useEffect` runs **after paint, asynchronously** — for most side effects (fetch, subscriptions).
- **Dependency arrays** — React does a shallow `Object.is` comparison of each dep. Missing deps → stale closures; unstable deps (new object/function each render) → effect fires every render.
- `useMemo` / `useCallback` cache values/functions across renders keyed on deps — an optimization, not a semantic guarantee (React may discard the cache).
- `useRef` — a mutable box that persists across renders without triggering re-render.
- `useSyncExternalStore` — the correct way to subscribe to external stores (tearing-safe under concurrency).

## Rendering Flow (End to End)

```
setState / dispatch
   → schedule update (assign lane/priority)
   → RENDER PHASE (interruptible):
       walk WIP fiber tree, call components,
       run reconciliation, mark effect flags
   → COMMIT PHASE (sync):
       before-mutation → mutation (DOM writes)
       → swap current tree → layout effects
   → browser paints
   → passive effects (useEffect) flush
```

## React 19 highlights (know these for a lead role)
- **React Compiler** (formerly "React Forget") — auto-memoizes to reduce the need for manual `useMemo`/`useCallback`.
- **Actions** & `useActionState`, `useFormStatus`, `useOptimistic` — first-class async form/mutation handling.
- **`use()`** — read promises/context conditionally during render (works with Suspense).
- **Server Components (RSC)** — components that render on the server, ship zero JS for themselves, and stream to the client. Reduces bundle size; changes the data-fetching model.

## StrictMode
- Dev-only. Double-invokes render and effect setup/cleanup to **surface impure renders and missing effect cleanup**. Not in production. A common "why does my effect run twice?" question.

---

### Interview Questions — React Internals

**Q1. What is reconciliation and how does React keep it O(n)?**
> The diffing process that compares the new element tree to the previous one to compute minimal DOM updates. React avoids O(n³) tree-diff using heuristics: different element types replace the whole subtree; same types are diffed in place; lists are matched via stable `key`s. This makes it effectively O(n).

**Q2. Explain React Fiber and why it was introduced.**
> Fiber is the reconciler rewrite that models work as a linked list of fiber nodes instead of a recursive call stack, making reconciliation interruptible. This enables time-slicing, prioritized updates (lanes), and concurrent features. The old stack reconciler was synchronous and blocked the main thread on large trees.

**Q3. Render phase vs commit phase — what's safe in each?**
> Render phase is interruptible and may run multiple times or be discarded, so it must be pure (no side effects, no DOM mutations). Commit phase is synchronous and runs once — DOM is mutated and effects fire there. `useLayoutEffect` runs in the commit's layout sub-phase (before paint); `useEffect` runs after paint.

**Q4. Why must hooks be called in the same order every render?**
> Hooks are stored as a positional linked list on the fiber. React matches a hook to its stored state by call order, not name. Conditional hooks break the alignment, corrupting state.

**Q5. `useMemo`, `useCallback`, `React.memo` — when do they actually help?**
> They help when (a) the referenced computation is genuinely expensive, or (b) a referentially-stable value/function is needed to prevent a memoized child from re-rendering or an effect from re-firing. They add comparison + memory cost, so blanket use can hurt. With React Compiler (React 19), much of this becomes automatic.

**Q6. What does `startTransition` solve?**
> It marks updates as non-urgent so React can interrupt them for urgent updates (like typing). Example: filtering a huge list while keeping the input responsive. `useDeferredValue` achieves a similar effect at the value level.

**Q7. Why does my effect run twice in development?**
> React StrictMode intentionally double-invokes effects (and renders) in dev to expose missing cleanup and impure logic. Write effects that are safe to run/cleanup repeatedly; it doesn't happen in production.

**Q8. Controlled vs uncontrolled components?**
> Controlled: React state is the single source of truth (`value` + `onChange`). Uncontrolled: the DOM holds the value, read via a ref/`defaultValue`. Controlled is predictable and validatable; uncontrolled is simpler/faster for large forms or file inputs.
