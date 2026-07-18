# The "Missing Hooks" Library — Popular Custom Hooks as Polyfills

> React ships primitives (`useState`, `useEffect`, `useRef`, `useSyncExternalStore`...) but not the everyday utilities apps actually need — debounced values, `localStorage` sync, click-outside, media queries. The community "polyfills" these gaps as custom hooks (see usehooks.com, `react-use`, `usehooks-ts`, `ahooks`). Interviewers ask you to implement these from scratch in machine-coding rounds precisely because each one hides a real gotcha (cleanup, SSR, stale closures, cross-tab sync).
>
> `useDebounce`, `useFetch` (with `AbortController`), and a tearing-safe global store via `useSyncExternalStore` are already covered in [10 — Machine Coding §7-8](10-machine-coding.md#7-custom-hooks-usedebounce-and-usefetch-with-cancellation). This file rounds out the rest of the popular set. Hook internals (fiber linked list, why order matters) are in [01 — Hooks Internals](01-react-internals.md#hooks-internals).

## 1) Lifecycle / SSR primitives

```tsx
// Avoids the SSR warning: useLayoutEffect on the server does nothing (no DOM),
// so swap to useEffect when window is undefined.
export const useIsomorphicLayoutEffect =
  typeof window !== 'undefined' ? useLayoutEffect : useEffect;

// True only after first client render — gate anything that reads `window`/`document`
// so server and first client render produce the same markup (no hydration mismatch).
export function useHasMounted(): boolean {
  const [mounted, setMounted] = useState(false);
  useEffect(() => setMounted(true), []);
  return mounted;
}

// Guard against "setState on unmounted component" in async callbacks that outlive the component.
export function useIsMounted(): () => boolean {
  const ref = useRef(false);
  useEffect(() => { ref.current = true; return () => { ref.current = false; }; }, []);
  return useCallback(() => ref.current, []);
}

// Value from the *previous* render — relies on the ref-updates-after-render ordering.
export function usePrevious<T>(value: T): T | undefined {
  const ref = useRef<T>();
  useEffect(() => { ref.current = value; }, [value]);
  return ref.current;
}

// Skip the effect on mount, run only on updates — common for "sync on change, not on initial load."
export function useUpdateEffect(effect: () => void | (() => void), deps: unknown[]) {
  const first = useRef(true);
  useEffect(() => {
    if (first.current) { first.current = false; return; }
    return effect();
  }, deps);
}
```

**Gotcha:** `useHasMounted`/`useIsomorphicLayoutEffect` exist *because* of SSR — without them you get the classic "Text content does not match server-rendered HTML" hydration error the moment a hook reads `window.innerWidth` or `localStorage` during render.

## 2) Storage

```tsx
// Cross-tab sync via the `storage` event — fires in OTHER tabs, not the one that wrote it,
// so same-tab consumers need the state setter (below) to see their own write immediately.
export function useLocalStorage<T>(key: string, initial: T) {
  const [value, setValue] = useState<T>(() => {
    try {
      const item = window.localStorage.getItem(key);
      return item ? (JSON.parse(item) as T) : initial;
    } catch { return initial; }
  });

  const setStoredValue = useCallback((next: T | ((prev: T) => T)) => {
    setValue(prev => {
      const resolved = typeof next === 'function' ? (next as (p: T) => T)(prev) : next;
      window.localStorage.setItem(key, JSON.stringify(resolved));
      return resolved;
    });
  }, [key]);

  useEffect(() => {
    const onStorage = (e: StorageEvent) => {
      if (e.key === key && e.newValue) setValue(JSON.parse(e.newValue) as T);
    };
    window.addEventListener('storage', onStorage);
    return () => window.removeEventListener('storage', onStorage);
  }, [key]);

  return [value, setStoredValue] as const;
}
```

**Trade-off:** `JSON.parse`/`stringify` on every read/write is fine for small values; for large or frequently-updated state, debounce the write or move to IndexedDB. Wrap reads in try/catch — Safari private mode throws on `localStorage` access, and malformed JSON from a previous app version shouldn't crash the app.

## 3) Timers

```tsx
// Declarative setInterval (Dan Abramov's pattern) — a ref holds the latest callback so the
// interval itself never needs to be torn down and restarted when the callback closes over new state.
export function useInterval(callback: () => void, delay: number | null) {
  const savedCallback = useRef(callback);
  useEffect(() => { savedCallback.current = callback; }, [callback]);
  useEffect(() => {
    if (delay === null) return;
    const id = setInterval(() => savedCallback.current(), delay);
    return () => clearInterval(id);
  }, [delay]);
}

export function useTimeout(callback: () => void, delay: number | null) {
  const savedCallback = useRef(callback);
  useEffect(() => { savedCallback.current = callback; }, [callback]);
  useEffect(() => {
    if (delay === null) return;
    const id = setTimeout(() => savedCallback.current(), delay);
    return () => clearTimeout(id);
  }, [delay]);
}

// Throttle: at most one update per `limit` ms — for scroll/resize/mousemove handlers.
// Contrast with useDebounce (10-machine-coding.md#7): debounce waits for quiet, throttle rate-limits during continuous activity.
export function useThrottle<T>(value: T, limit = 200): T {
  const [throttled, setThrottled] = useState(value);
  const lastRan = useRef(Date.now());
  useEffect(() => {
    const id = setTimeout(() => {
      if (Date.now() - lastRan.current >= limit) {
        setThrottled(value);
        lastRan.current = Date.now();
      }
    }, limit - (Date.now() - lastRan.current));
    return () => clearTimeout(id);
  }, [value, limit]);
  return throttled;
}
```

**Gotcha (the classic trap):** `setInterval(callback, delay)` written naively — with `callback` used directly instead of via a ref — captures a **stale closure** of `callback` from whenever the interval was created. If deps are wrong, either the interval never picks up new state, or (worse) it's torn down and recreated every render, drifting the actual firing cadence.

## 4) DOM / browser API bindings

```tsx
// Generic addEventListener hook — supports window, a DOM element, or a ref, always via a
// ref-held handler so the listener itself is added once and never needs re-binding on every render.
export function useEventListener<K extends keyof WindowEventMap>(
  eventName: K,
  handler: (e: WindowEventMap[K]) => void,
  element: Window | HTMLElement | null = typeof window !== 'undefined' ? window : null,
) {
  const savedHandler = useRef(handler);
  useEffect(() => { savedHandler.current = handler; }, [handler]);
  useEffect(() => {
    if (!element?.addEventListener) return;
    const eventListener = (e: Event) => savedHandler.current(e as WindowEventMap[K]);
    element.addEventListener(eventName, eventListener);
    return () => element.removeEventListener(eventName, eventListener);
  }, [eventName, element]);
}

// Fire a callback when a pointerdown/mousedown lands outside the ref'd element.
export function useOnClickOutside<T extends HTMLElement>(
  ref: React.RefObject<T>,
  handler: (e: PointerEvent) => void,
) {
  useEventListener('pointerdown' as any, (e: any) => {
    if (!ref.current || ref.current.contains(e.target as Node)) return;
    handler(e);
  }, typeof document !== 'undefined' ? (document as any) : null);
}

export function useWindowSize() {
  const [size, setSize] = useState({ width: 0, height: 0 });
  useIsomorphicLayoutEffect(() => {
    const update = () => setSize({ width: window.innerWidth, height: window.innerHeight });
    update();
    window.addEventListener('resize', update);
    return () => window.removeEventListener('resize', update);
  }, []);
  return size;
}

// matchMedia binding — `useSyncExternalStore` is arguably the *correct* primitive here
// (external, non-React source of truth), same reasoning as the store in 10-machine-coding.md#8.
export function useMediaQuery(query: string): boolean {
  const subscribe = useCallback((cb: () => void) => {
    const mql = window.matchMedia(query);
    mql.addEventListener('change', cb);
    return () => mql.removeEventListener('change', cb);
  }, [query]);
  const getSnapshot = () => window.matchMedia(query).matches;
  const getServerSnapshot = () => false;
  return useSyncExternalStore(subscribe, getSnapshot, getServerSnapshot);
}

export function useIntersectionObserver<T extends Element>(
  ref: React.RefObject<T>,
  options: IntersectionObserverInit = {},
): boolean {
  const [isIntersecting, setIntersecting] = useState(false);
  useEffect(() => {
    if (!ref.current) return;
    const observer = new IntersectionObserver(([entry]) => setIntersecting(entry.isIntersecting), options);
    observer.observe(ref.current);
    return () => observer.disconnect();
  }, [ref, options.root, options.rootMargin, options.threshold]);
  return isIntersecting;
}

export function useHover<T extends HTMLElement>(ref: React.RefObject<T>): boolean {
  const [hovered, setHovered] = useState(false);
  useEventListener('mouseenter' as any, () => setHovered(true), ref.current as any);
  useEventListener('mouseleave' as any, () => setHovered(false), ref.current as any);
  return hovered;
}

export function useOnlineStatus(): boolean {
  return useSyncExternalStore(
    (cb) => { window.addEventListener('online', cb); window.addEventListener('offline', cb);
              return () => { window.removeEventListener('online', cb); window.removeEventListener('offline', cb); }; },
    () => navigator.onLine,
    () => true,
  );
}

export function useCopyToClipboard(): [string | null, (text: string) => Promise<boolean>] {
  const [copiedText, setCopiedText] = useState<string | null>(null);
  const copy = useCallback(async (text: string) => {
    try { await navigator.clipboard.writeText(text); setCopiedText(text); return true; }
    catch { setCopiedText(null); return false; }
  }, []);
  return [copiedText, copy];
}

export function useLockBodyScroll(locked: boolean) {
  useIsomorphicLayoutEffect(() => {
    if (!locked) return;
    const original = document.body.style.overflow;
    document.body.style.overflow = 'hidden'; // pair with a modal — prevents background scroll on iOS Safari
    return () => { document.body.style.overflow = original; };
  }, [locked]);
}
```

**Options-object dependency trap:** `useIntersectionObserver`'s `options` is a new object every render — the effect deps destructure the primitive fields (`options.root`, `options.rootMargin`, `options.threshold`) instead of depending on `options` itself, or the observer would be torn down and rebuilt on every render.

**Ref-in-deps trap:** `useWindowSize`/event-listener hooks that take a `ref` as a dependency rely on the ref object identity staying stable across renders (it does, by React's contract) — but reading `ref.current` inside a dependency array is a no-op, since mutating a ref never triggers a re-render or dep-array re-evaluation. This is why DOM-bound hooks either take the element as a plain argument (re-runs when the node itself changes) or use a callback ref.

## 5) State-shape helpers

```tsx
export function useToggle(initial = false): [boolean, () => void, (v: boolean) => void] {
  const [value, setValue] = useState(initial);
  const toggle = useCallback(() => setValue(v => !v), []);
  return [value, toggle, setValue];
}

export function useArray<T>(initial: T[] = []) {
  const [array, setArray] = useState(initial);
  return {
    array,
    push: (item: T) => setArray(a => [...a, item]),
    remove: (index: number) => setArray(a => a.filter((_, i) => i !== index)),
    update: (index: number, item: T) => setArray(a => a.map((x, i) => (i === index ? item : x))),
    clear: () => setArray([]),
  };
}

export function useMap<K, V>(initial?: Iterable<[K, V]>) {
  const [map, setMap] = useState(new Map(initial));
  const set = useCallback((k: K, v: V) => setMap(prev => new Map(prev).set(k, v)), []);
  const remove = useCallback((k: K) => setMap(prev => { const next = new Map(prev); next.delete(k); return next; }), []);
  return { map, set, remove };
}

// Undo/redo — three stacks. `set` truncates any redo-able future on a new change (standard editor semantics).
export function useUndo<T>(initial: T) {
  const [state, setState] = useState({ past: [] as T[], present: initial, future: [] as T[] });
  const set = useCallback((next: T) => setState(s => ({ past: [...s.past, s.present], present: next, future: [] })), []);
  const undo = useCallback(() => setState(s => {
    if (!s.past.length) return s;
    const previous = s.past[s.past.length - 1];
    return { past: s.past.slice(0, -1), present: previous, future: [s.present, ...s.future] };
  }), []);
  const redo = useCallback(() => setState(s => {
    if (!s.future.length) return s;
    const [next, ...rest] = s.future;
    return { past: [...s.past, s.present], present: next, future: rest };
  }), []);
  return { state: state.present, set, undo, redo, canUndo: state.past.length > 0, canRedo: state.future.length > 0 };
}
```

**Why `Map`/`Set` state needs a new instance on every update:** React compares state with `Object.is`; mutating a `Map` in place (`map.set(k, v)` then calling `setMap(map)` with the same reference) is a no-op re-render-wise. Always spread/copy (`new Map(prev)`) before mutating, exactly like array/object state.

## 6) Composed hooks (build on the above)

```tsx
// Composes useLocalStorage + useMediaQuery: persisted preference, falling back to OS preference
// when the user hasn't explicitly chosen — the "explicit choice beats system default" pattern.
export function useDarkMode(): [boolean, () => void] {
  const prefersDark = useMediaQuery('(prefers-color-scheme: dark)');
  const [enabled, setEnabled] = useLocalStorage<boolean>('dark-mode', prefersDark);
  useEffect(() => { document.documentElement.classList.toggle('dark', enabled); }, [enabled]);
  return [enabled, () => setEnabled(v => !v)];
}
```

---

## Common pitfalls (across this whole library)

- **Missing cleanup** on every browser-API hook — event listeners, observers, timers all need a teardown or they leak across mounts/unmounts (same theme as [10 — Common pitfalls](10-machine-coding.md#common-pitfalls)).
- **SSR reads of `window`/`document` during render** — always gate behind `useEffect`/`useIsomorphicLayoutEffect` or a `typeof window !== 'undefined'` check, never at module or render top-level.
- **New object/array identity per render** feeding a dependency array (options objects, inline callbacks) — either memoize the input or destructure to primitives inside the effect's deps.
- **Reaching for `useSyncExternalStore` vs plain `useState`+`useEffect`:** use the store API when the source of truth is genuinely external and shared (media query, online status, a module-level store) so subscriptions are torn-safe; plain state+effect is fine for per-component derived values.

---

### Interview Questions — Custom Hooks Library

**Q1. Implement `useDebounce` from scratch — wait, why is this in a different file?**
> Covered in [10 — Machine Coding §7](10-machine-coding.md#7-custom-hooks-usedebounce-and-usefetch-with-cancellation); this file assumes that one and the tearing-safe store in §8 as given, and covers the rest of the "missing hooks" set interviewers pull from.

**Q2. `useDebounce` vs `useThrottle` — when do you reach for each?**
> Debounce delays until input goes quiet (search-as-you-type: fire once after the user stops). Throttle caps the rate during continuous activity (scroll-position tracking: fire at most every N ms while scrolling never stops). Debouncing a scroll handler means it may never fire until scrolling ends; throttling a search box means you'd fire requests mid-word.

**Q3. Why does a naive `useInterval(() => setCount(count + 1), 1000)` break, and how does the ref pattern fix it?**
> The callback passed to `setInterval` closes over `count` from the render it was created in. Since the effect that calls `setInterval` only re-runs when `delay` changes (not `count`), the interval keeps calling the *original* closure forever, incrementing from the same stale value every second. The fix stores the latest callback in a ref, updated every render via a separate effect with no delay dependency, so the interval body always calls through to fresh state without needing to be torn down and recreated.

**Q4. Why does `useLocalStorage`'s `storage` event listener not fire in the tab that made the write?**
> Per spec, the `storage` event is dispatched only to *other* browsing contexts (other tabs/windows/iframes) with the same origin — the writing tab already has the new value in its own state, so firing there too would be redundant. This is why the hook's own setter updates local state directly on write, and only listens for the event to catch writes from *other* tabs.

**Q5. Why implement `useMediaQuery`/`useOnlineStatus` with `useSyncExternalStore` instead of `useState` + `useEffect`?**
> Both track state that lives outside React (the browser's media-query engine, `navigator.onLine`) and can change between render and commit, or be read by multiple components independently. `useSyncExternalStore` guarantees a consistent snapshot across the whole tree even under concurrent rendering (no tearing), and it has a built-in server-snapshot argument for SSR — a `useState`+`useEffect` version can flash the wrong value on mount and has no principled SSR answer.

**Q6. A junior engineer's `useOnClickOutside` fires immediately when the triggering element (e.g., an "open menu" button) is clicked. Why, and how do you fix it?**
> If the hook is wired to `click`/`mouseup` and the same click both opens the menu and is the "outside" click the listener sees on the way up (or the listener is attached before the current click event finishes bubbling), it can close what it just opened. Fixes: bind to `pointerdown`/`mousedown` instead of `click` so it fires before the button's own click handler, use a captured target check that also excludes the trigger button's ref, or attach the outside listener in a microtask/next tick after the opening click completes.
