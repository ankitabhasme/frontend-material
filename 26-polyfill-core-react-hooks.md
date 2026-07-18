# Polyfilling React's Own Hooks From Scratch

> Different ask than [25](25-react-custom-hooks-polyfills.md): that file polyfills things React *doesn't* ship (browser-API bindings). This file polyfills the hooks React itself ships — `useState`, `useReducer`, `useRef`, `useEffect`, `useMemo`, `useCallback` — plus `useDebounce`/`useThrottle` built on top, to show they fall out for free once you have `useState`+`useEffect`. This is the "build your own React hooks" machine-coding/whiteboard question, and it's really a test of whether you understand [01 — Hooks Internals](01-react-internals.md#hooks-internals): hooks are state slots matched **by call order**, not identity.

## The core idea: a slots array + a cursor

Real React stores each fiber's hooks as a linked list on `memoizedState`. A toy version needs only an array and an index that resets before every render — this single mechanism is *why* hooks can't be conditional: skip a hook on one render and every hook after it reads the wrong slot.

```js
let hookStates = [];   // one slot per hook call, in call order
let hookIndex = 0;     // cursor, reset before each render
let scheduleRerender;  // wired up by the "renderer" below

function renderComponent(Component, props) {
  hookIndex = 0;               // rewind the cursor — this IS the "same order every render" rule
  const output = Component(props);
  return output;
}
```

A real host would call `renderComponent` again whenever `scheduleRerender()` fires (e.g. `requestAnimationFrame` batching multiple `setState` calls into one render) — omitted here to keep focus on the hooks themselves.

## `useState`

```js
function useState(initialValue) {
  const currentIndex = hookIndex;
  if (hookStates[currentIndex] === undefined) {
    hookStates[currentIndex] = typeof initialValue === 'function' ? initialValue() : initialValue;
  }
  function setState(next) {
    const prev = hookStates[currentIndex];
    const value = typeof next === 'function' ? next(prev) : next;
    if (Object.is(value, prev)) return;      // real React bails out on Object.is equality — no re-render, no wasted work
    hookStates[currentIndex] = value;
    scheduleRerender();
  }
  hookIndex++;
  return [hookStates[currentIndex], setState];
}
```

**Gotcha:** lazy initial state (`useState(() => expensive())`) only calls the initializer once, on the slot's first creation — a naive `hookStates[i] ?? initialValue` re-evaluates the expensive call *every render* even though it discards the result. The `undefined` check plus calling the function form only inside that branch is what makes it lazy.

## `useReducer` (built on `useState`)

```js
function useReducer(reducer, initialArg, init) {
  const [state, setState] = useState(init ? () => init(initialArg) : initialArg);
  function dispatch(action) {
    setState(prev => reducer(prev, action));   // functional update -> always reduces the latest state, never a stale closure
  }
  return [state, dispatch];
}
```

**Why this isn't circular reasoning:** real React actually implements `useReducer` and `useState` as siblings on the same underlying update-queue mechanism (`useState` is `useReducer` with an identity-ish reducer under the hood) — deriving one from the other here is a faithful simplification, not a cheat.

## `useRef`

```js
function useRef(initialValue) {
  const currentIndex = hookIndex;
  if (hookStates[currentIndex] === undefined) {
    hookStates[currentIndex] = { current: initialValue };   // the SAME object every render — never reallocated
  }
  hookIndex++;
  return hookStates[currentIndex];
}
```

**Gotcha:** mutating `ref.current` must never call `scheduleRerender()` — that's the entire point of a ref (persist across renders without triggering one). A common bug is a junior re-implementing this as `useState({current: initialValue})[0]`, which persists the value fine but silently re-renders on every mutation because it's routed through `setState`.

## `useEffect`

```js
function useEffect(callback, deps) {
  const currentIndex = hookIndex;
  const prevHook = hookStates[currentIndex];
  const depsChanged = !prevHook || !deps || deps.some((d, i) => !Object.is(d, prevHook.deps[i]));

  if (depsChanged) {
    if (prevHook?.cleanup) prevHook.cleanup();      // run the PREVIOUS effect's cleanup before the next effect
    queueMicrotask(() => {                          // real React: after paint. Toy version: next microtask, good enough to prove the "after render" ordering
      const cleanup = callback();
      hookStates[currentIndex].cleanup = cleanup;
    });
    hookStates[currentIndex] = { deps, cleanup: prevHook?.cleanup };
  }
  hookIndex++;
}
```

**Gotcha (the one every interviewer probes):** cleanup for render *N* must run **before** the new effect for render *N* fires, using the deps/closure captured at render *N-1* — not render *N*. Get the ordering backwards and you leak subscriptions/timers or read stale closures. This is also exactly why `useEffect` runs *after* paint (`queueMicrotask`/real scheduler) while `useLayoutEffect` would run synchronously in the same slot, before the browser paints — the polyfill for `useLayoutEffect` is identical except the `callback()` call happens synchronously instead of queued.

## `useMemo`

```js
function useMemo(factory, deps) {
  const currentIndex = hookIndex;
  const prevHook = hookStates[currentIndex];
  const depsChanged = !prevHook || !deps || deps.some((d, i) => !Object.is(d, prevHook.deps[i]));
  if (depsChanged) {
    hookStates[currentIndex] = { value: factory(), deps };
  }
  hookIndex++;
  return hookStates[currentIndex].value;
}
```

## `useCallback` (built on `useMemo`)

```js
function useCallback(fn, deps) {
  return useMemo(() => fn, deps);   // memoizing "the function" is just memoizing a value whose value happens to be a function
}
```

**Interview one-liner:** `useCallback(fn, deps)` is defined as `useMemo(() => fn, deps)` in React's actual source too — not just a teaching simplification.

---

## `useDebounce` and `useThrottle`, built from the polyfills above

The payoff of building `useState`/`useEffect` yourself: these utility hooks are trivial once the primitives exist — nothing React-internal left to invent.

```js
function useDebounce(value, delay = 300) {
  const [debounced, setDebounced] = useState(value);
  useEffect(() => {
    const timer = setTimeout(() => setDebounced(value), delay);
    return () => clearTimeout(timer);   // cancels the PREVIOUS pending update whenever value/delay changes before it fires
  }, [value, delay]);
  return debounced;
}

function useThrottle(value, limit = 200) {
  const [throttled, setThrottled] = useState(value);
  const lastRan = useRef(0);
  useEffect(() => {
    const remaining = limit - (Date.now() - lastRan.current);
    if (remaining <= 0) {
      setThrottled(value);
      lastRan.current = Date.now();
      return;
    }
    const timer = setTimeout(() => {
      setThrottled(value);
      lastRan.current = Date.now();
    }, remaining);
    return () => clearTimeout(timer);
  }, [value, limit]);
  return throttled;
}
```

Both are identical in shape to the "real" versions in [25](25-react-custom-hooks-polyfills.md#3-timers) — proof that once `useState`/`useEffect`/`useRef` exist (polyfilled or real), everything above them is ordinary composition, not new React magic.

---

## How real React actually differs from this toy version

- **Per-fiber storage, not one global array.** This polyfill has a single `hookStates`/`hookIndex` pair — fine for one component instance. Real React keys hooks off the **currently-rendering fiber's** `memoizedState` linked list, so thousands of component instances each get independent, isolated hook storage.
- **Effects are prioritized and batched**, not fired on a bare `queueMicrotask` — passive effects (`useEffect`) are scheduled on a lane after commit+paint; layout effects (`useLayoutEffect`) run synchronously in the commit phase before paint.
- **Concurrent rendering** means the render phase can be paused, resumed, or thrown away entirely (`Object.is` bailouts, `startTransition`, time-slicing) — this polyfill's `renderComponent` always runs to completion synchronously.
- **`Object.is`, not `!==`**, for both state and dependency-array comparisons — matters for `NaN` (`NaN !== NaN` is `true`, but `Object.is(NaN, NaN)` is `true`) and `-0`/`+0`.

---

### Interview Questions — Polyfilling Hooks

**Q1. Why does a hooks polyfill need an array + index instead of, say, a `Map` keyed by variable name?**
> There's no reliable way to know a hook's "name" at runtime — `const [x, setX] = useState(0)` gives the engine no string to key off. React (and any polyfill) instead relies on **call order being stable across renders**, storing hooks positionally and walking them with a cursor that resets every render. That's precisely why conditional or looped hook calls corrupt state: the cursor no longer lines up with the same logical hook each time.

**Q2. In your `useEffect` polyfill, why must the cleanup from the *previous* render run before the *new* effect, and using the *old* deps?**
> The cleanup exists to tear down whatever the previous effect set up — an event listener, a subscription, a timer — using the closure it captured at that time. If the new effect ran first, or cleanup used the new deps, you'd either double-subscribe (leak) or attempt to unsubscribe something that was never subscribed with those params (silent no-op leak). Order must be: run old cleanup → then run new effect → store new cleanup.

**Q3. How would you extend this polyfill to support multiple independent component instances?**
> Move `hookStates`/`hookIndex` off the module-level globals and onto a per-instance "fiber" object; the renderer looks up (or creates) that instance's hook array before calling the component function, and points a module-level "currently rendering fiber" pointer at it for the duration of the call — mirroring how real React's dispatcher resolves hooks against `currentlyRenderingFiber.memoizedState`.

**Q4. Why is `useCallback(fn, deps)` just `useMemo(() => fn, deps)` and not a separate primitive?**
> Both hooks solve "recompute only when deps change" — `useMemo` memoizes an arbitrary computed value, and a function is just a value. Wrapping `fn` in a factory that returns it unchanged reuses all of `useMemo`'s deps-comparison and slot-storage logic instead of duplicating it, which is exactly what React's own source does.

**Q5. Your polyfilled `useDebounce` fires a state update after the component that owns it has been torn down and remounted — what breaks, and how does the cleanup function prevent it?**
> Without cleanup, an in-flight `setTimeout` from the old instance would call `setDebounced` against hook slots that either no longer exist or now belong to a *different* logical instance (since a fresh mount rebuilds `hookStates` from scratch) — best case a no-op, worst case corrupting the new instance's state. The `return () => clearTimeout(timer)` inside `useEffect` runs as cleanup on unmount (and on every deps change) specifically to cancel that pending timer before it can fire against dead or reused state.
