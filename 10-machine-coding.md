# Machine Coding / Practical Build Round

> Build a working, accessible, well-structured component in 60–90 minutes — graded on judgment and code quality, not just "does it work."

The machine-coding round is where a lead separates from a mid-level engineer by getting the **component API, state model, and accessibility** right, narrating trade-offs, and committing in coherent increments — not by cramming features. A slightly smaller scope done cleanly and accessibly beats a feature-complete blob.

## What interviewers actually grade

| Dimension | What "good" looks like |
|---|---|
| **Component API design** | Minimal, predictable props; controlled/uncontrolled done right; sensible defaults; composition over config explosion |
| **State modeling** | Single source of truth, no redundant/derived state, correct state kind (local/server/URL) |
| **Correctness** | Handles the happy path *and* the stated requirements precisely |
| **Edge cases** | Empty, loading, error, boundary values, rapid input, unmount mid-flight |
| **Accessibility** | Keyboard operable, correct roles/ARIA, focus management, labels — the top differentiator |
| **Clean/readable code** | Clear names, small functions, no dead code, consistent style, typed |
| **Incremental commits** | Small, working, message-per-step; not one giant commit |
| **Testing mindset** | Names the cases they'd test; writes 1–2 if time allows |
| **Communication** | Thinks aloud, states assumptions, asks before gold-plating |

## Repeatable approach / checklist

1. **Clarify (2–5 min):** scope the must-haves, confirm what's out of scope, ask about data source (mock vs real API), and agree the a11y bar. Write the requirements list down.
2. **Design the API & state (5 min):** sketch the component props, the state shape, and the events. Decide controlled vs uncontrolled. Say it out loud.
3. **Scaffold:** static markup with semantic HTML first, then wire state. Commit the skeleton.
4. **Vertical slices:** implement one working slice at a time (render → interaction → async → edge states). Commit each.
5. **Accessibility pass:** keyboard nav, roles, focus, labels — do this *as you go*, not at the end.
6. **Edge states:** loading/empty/error, cleanup on unmount, race conditions.
7. **Polish & tests:** extract hooks, remove dead code, add a couple of tests, note extension points you deliberately skipped.

Keep a mock data layer trivial (`setTimeout`/`Promise`) so you spend time on the component, not the backend.

---

## Implementations

### 1) Debounced Autocomplete / Typeahead

Keyboard-navigable combobox with cancellation and ARIA per the APG combobox pattern.

```tsx
import { useEffect, useRef, useState, useId } from 'react';

interface Item { id: string; label: string; }

function useDebounced<T>(value: T, delay = 250): T {
  const [v, setV] = useState(value);
  useEffect(() => {
    const t = setTimeout(() => setV(value), delay);
    return () => clearTimeout(t); // reset timer on change -> true debounce
  }, [value, delay]);
  return v;
}

export function Autocomplete({ fetchItems }: { fetchItems: (q: string, signal: AbortSignal) => Promise<Item[]> }) {
  const [query, setQuery] = useState('');
  const [items, setItems] = useState<Item[]>([]);
  const [active, setActive] = useState(-1);     // highlighted index for aria-activedescendant
  const [open, setOpen] = useState(false);
  const [status, setStatus] = useState<'idle' | 'loading' | 'error'>('idle');
  const debounced = useDebounced(query);
  const listId = useId();

  useEffect(() => {
    if (debounced.trim().length < 2) { setItems([]); return; }
    const ctrl = new AbortController();
    setStatus('loading');
    fetchItems(debounced, ctrl.signal)
      .then(res => { setItems(res); setActive(-1); setStatus('idle'); setOpen(true); })
      .catch(e => { if (e.name !== 'AbortError') setStatus('error'); });
    return () => ctrl.abort(); // cancel stale request -> prevents out-of-order overwrite
  }, [debounced, fetchItems]);

  function onKeyDown(e: React.KeyboardEvent) {
    if (!open && (e.key === 'ArrowDown' || e.key === 'ArrowUp')) { setOpen(true); return; }
    switch (e.key) {
      case 'ArrowDown': e.preventDefault(); setActive(i => Math.min(i + 1, items.length - 1)); break;
      case 'ArrowUp':   e.preventDefault(); setActive(i => Math.max(i - 1, 0)); break;
      case 'Enter':     if (active >= 0) { setQuery(items[active].label); setOpen(false); } break;
      case 'Escape':    setOpen(false); setActive(-1); break;
    }
  }

  return (
    <div>
      <label htmlFor="ac-input">Search</label>
      <input
        id="ac-input" role="combobox" aria-expanded={open} aria-controls={listId}
        aria-autocomplete="list" autoComplete="off"
        aria-activedescendant={active >= 0 ? `${listId}-${active}` : undefined}
        value={query} onChange={e => setQuery(e.target.value)} onKeyDown={onKeyDown}
      />
      {status === 'error' && <p role="alert">Something went wrong. Retry.</p>}
      {open && (
        <ul id={listId} role="listbox">
          {status === 'idle' && items.length === 0 && <li aria-disabled>No results</li>}
          {items.map((it, i) => (
            <li key={it.id} id={`${listId}-${i}`} role="option" aria-selected={i === active}
                onMouseDown={() => { setQuery(it.label); setOpen(false); }}>
              {it.label}
            </li>
          ))}
        </ul>
      )}
    </div>
  );
}
```

**Extension points:** result highlighting, LRU cache by query, min-length config, async-safe seq guard, virtualization for long lists.

### 2) Accessible Tabs (APG pattern, roving tabindex)

```tsx
import { useRef, useState, useId } from 'react';

interface Tab { id: string; label: string; content: React.ReactNode; }

export function Tabs({ tabs }: { tabs: Tab[] }) {
  const [selected, setSelected] = useState(0);
  const base = useId();
  const refs = useRef<(HTMLButtonElement | null)[]>([]);

  function onKeyDown(e: React.KeyboardEvent, i: number) {
    let next = i;
    if (e.key === 'ArrowRight') next = (i + 1) % tabs.length;
    else if (e.key === 'ArrowLeft') next = (i - 1 + tabs.length) % tabs.length;
    else if (e.key === 'Home') next = 0;
    else if (e.key === 'End') next = tabs.length - 1;
    else return;
    e.preventDefault();
    setSelected(next);
    refs.current[next]?.focus(); // move DOM focus with selection
  }

  return (
    <div>
      <div role="tablist" aria-label="Sections">
        {tabs.map((t, i) => (
          <button
            key={t.id} ref={el => { refs.current[i] = el; }}
            role="tab" id={`${base}-tab-${i}`} aria-controls={`${base}-panel-${i}`}
            aria-selected={i === selected}
            tabIndex={i === selected ? 0 : -1}  /* roving tabindex: only active tab is tabbable */
            onClick={() => setSelected(i)} onKeyDown={e => onKeyDown(e, i)}>
            {t.label}
          </button>
        ))}
      </div>
      {tabs.map((t, i) => (
        <div key={t.id} role="tabpanel" id={`${base}-panel-${i}`}
             aria-labelledby={`${base}-tab-${i}`} hidden={i !== selected} tabIndex={0}>
          {t.content}
        </div>
      ))}
    </div>
  );
}
```

**Note:** this is the *automatic activation* variant (selection follows focus). For expensive panels, switch to *manual activation* (arrow moves focus, Enter/Space selects).

### 3) Accessible Modal / Dialog (focus trap, Esc, restore focus, portal)

```tsx
import { useEffect, useRef } from 'react';
import { createPortal } from 'react-dom';

export function Modal({ open, onClose, title, children }: {
  open: boolean; onClose: () => void; title: string; children: React.ReactNode;
}) {
  const ref = useRef<HTMLDivElement>(null);
  const previouslyFocused = useRef<HTMLElement | null>(null);

  useEffect(() => {
    if (!open) return;
    previouslyFocused.current = document.activeElement as HTMLElement;
    const dialog = ref.current!;
    const focusables = () => dialog.querySelectorAll<HTMLElement>(
      'a[href],button:not([disabled]),textarea,input,select,[tabindex]:not([tabindex="-1"])');
    focusables()[0]?.focus() ?? dialog.focus();

    function onKey(e: KeyboardEvent) {
      if (e.key === 'Escape') { onClose(); return; }
      if (e.key !== 'Tab') return;
      const items = focusables();
      if (!items.length) return;
      const first = items[0], last = items[items.length - 1];
      if (e.shiftKey && document.activeElement === first) { e.preventDefault(); last.focus(); }
      else if (!e.shiftKey && document.activeElement === last) { e.preventDefault(); first.focus(); }
    }
    document.addEventListener('keydown', onKey);
    document.body.style.overflow = 'hidden';
    return () => {
      document.removeEventListener('keydown', onKey);
      document.body.style.overflow = '';
      previouslyFocused.current?.focus(); // restore focus to trigger
    };
  }, [open, onClose]);

  if (!open) return null;
  return createPortal(
    <div className="overlay" onClick={onClose}>
      <div ref={ref} role="dialog" aria-modal="true" aria-labelledby="dlg-title"
           tabIndex={-1} onClick={e => e.stopPropagation()}>
        <h2 id="dlg-title">{title}</h2>
        {children}
        <button onClick={onClose}>Close</button>
      </div>
    </div>,
    document.body,
  );
}
```

**Extension points:** `inert` on background, `prefers-reduced-motion` for transitions, scroll-lock that accounts for scrollbar width.

### 4) Star Rating (keyboard + ARIA)

Use a radiogroup so screen readers and keyboard users get native semantics.

```tsx
import { useState } from 'react';

export function StarRating({ max = 5, value, onChange }: {
  max?: number; value: number; onChange: (v: number) => void;
}) {
  const [hover, setHover] = useState(0);
  const shown = hover || value;
  return (
    <div role="radiogroup" aria-label="Rating">
      {Array.from({ length: max }, (_, i) => i + 1).map(n => (
        <span
          key={n} role="radio" aria-checked={value === n} aria-label={`${n} star${n > 1 ? 's' : ''}`}
          tabIndex={value === n || (value === 0 && n === 1) ? 0 : -1}
          onClick={() => onChange(n)}
          onMouseEnter={() => setHover(n)} onMouseLeave={() => setHover(0)}
          onKeyDown={e => {
            if (e.key === 'ArrowRight' || e.key === 'ArrowUp') onChange(Math.min(value + 1, max));
            else if (e.key === 'ArrowLeft' || e.key === 'ArrowDown') onChange(Math.max(value - 1, 1));
            else if (e.key === ' ' || e.key === 'Enter') onChange(n);
          }}>
          {n <= shown ? '★' : '☆'}
        </span>
      ))}
    </div>
  );
}
```

### 5) Infinite scroll with IntersectionObserver

```tsx
import { useEffect, useRef, useState, useCallback } from 'react';

export function InfiniteList({ loadPage }: {
  loadPage: (cursor: string | null) => Promise<{ items: Item[]; nextCursor: string | null }>;
}) {
  const [items, setItems] = useState<Item[]>([]);
  const [cursor, setCursor] = useState<string | null>(null);
  const [done, setDone] = useState(false);
  const [loading, setLoading] = useState(false);
  const sentinel = useRef<HTMLDivElement>(null);

  const loadMore = useCallback(async () => {
    if (loading || done) return;            // guard against overlapping loads
    setLoading(true);
    try {
      const { items: batch, nextCursor } = await loadPage(cursor);
      setItems(prev => [...prev, ...batch]);
      setCursor(nextCursor);
      if (!nextCursor) setDone(true);
    } finally { setLoading(false); }
  }, [cursor, done, loading, loadPage]);

  useEffect(() => {
    const el = sentinel.current;
    if (!el) return;
    const obs = new IntersectionObserver(entries => {
      if (entries[0].isIntersecting) loadMore();
    }, { rootMargin: '200px' });           // prefetch before fully visible
    obs.observe(el);
    return () => obs.disconnect();          // cleanup observer
  }, [loadMore]);

  return (
    <>
      <ul>{items.map(it => <li key={it.id}>{it.label}</li>)}</ul>
      {loading && <p aria-live="polite">Loading…</p>}
      {!done && <div ref={sentinel} aria-hidden />}
      {done && <p>End of list</p>}
    </>
  );
}
```

**Extension points:** virtualization for huge lists, per-page error/retry, scroll-restoration, "Load more" fallback button for keyboard users.

### 6) Nested comments / threaded tree (recursion)

```tsx
interface Comment { id: string; text: string; author: string; children: Comment[]; }

function CommentNode({ node, depth = 0 }: { node: Comment; depth?: number }) {
  return (
    <li style={{ marginLeft: depth * 16 }}>
      <div><strong>{node.author}</strong>: {node.text}</div>
      {node.children.length > 0 && (
        <ul>
          {node.children.map(child => (
            <CommentNode key={child.id} node={child} depth={depth + 1} />
          ))}
        </ul>
      )}
    </li>
  );
}

export function CommentThread({ comments }: { comments: Comment[] }) {
  return <ul>{comments.map(c => <CommentNode key={c.id} node={c} />)}</ul>;
}
```

**Extension points:** collapse/expand per node, lazy-load deep replies, flatten to a normalized `{id -> comment, childIds}` map to make add/edit O(1) instead of deep-tree mutation, cap render depth with "continue thread" link.

### 7) Custom hooks: `useDebounce` and `useFetch` with cancellation

```tsx
export function useDebounce<T>(value: T, delay = 300): T {
  const [debounced, setDebounced] = useState(value);
  useEffect(() => {
    const t = setTimeout(() => setDebounced(value), delay);
    return () => clearTimeout(t);
  }, [value, delay]);
  return debounced;
}

interface FetchState<T> { data?: T; error?: Error; loading: boolean; }

export function useFetch<T>(url: string): FetchState<T> {
  const [state, setState] = useState<FetchState<T>>({ loading: true });
  useEffect(() => {
    const ctrl = new AbortController();
    setState({ loading: true });
    fetch(url, { signal: ctrl.signal })
      .then(r => { if (!r.ok) throw new Error(`HTTP ${r.status}`); return r.json(); })
      .then((data: T) => setState({ data, loading: false }))
      .catch((error: Error) => { if (error.name !== 'AbortError') setState({ error, loading: false }); });
    return () => ctrl.abort(); // cancel on url change / unmount -> no state update after unmount
  }, [url]);
  return state;
}
```

### 8) Tiny global store (`useSyncExternalStore`)

The correct, tearing-safe way to build a store without a library.

```tsx
import { useSyncExternalStore } from 'react';

function createStore<T>(initial: T) {
  let state = initial;
  const listeners = new Set<() => void>();
  return {
    getState: () => state,
    setState: (patch: Partial<T> | ((s: T) => Partial<T>)) => {
      const next = typeof patch === 'function' ? patch(state) : patch;
      state = { ...state, ...next };
      listeners.forEach(l => l());
    },
    subscribe: (l: () => void) => { listeners.add(l); return () => listeners.delete(l); },
  };
}

const counterStore = createStore({ count: 0 });

export function useCounter() {
  const count = useSyncExternalStore(counterStore.subscribe, () => counterStore.getState().count);
  return { count, inc: () => counterStore.setState(s => ({ count: s.count + 1 })) };
}
```

**Why `useSyncExternalStore`:** it's concurrent-safe (no tearing under React 18 concurrent rendering) and gives a server snapshot for SSR — a context+reducer store can tear and re-renders the whole subtree. Mention selectors + `useSyncExternalStoreWithSelector` to avoid over-rendering.

---

## Common pitfalls

- **Stale closures:** an event handler or interval capturing an old state/prop value — fix with functional updates (`setX(prev => …)`), refs, or correct deps.
- **Missing cleanup:** not aborting fetches, clearing timers, or disconnecting observers/listeners on unmount → memory leaks and "setState on unmounted component" / out-of-order updates.
- **Index-as-key:** `key={i}` in editable/reorderable lists corrupts state and breaks reconciliation — use stable ids.
- **A11y omissions:** clickable `<div>` instead of `<button>`, no keyboard handlers, missing labels/roles, no focus management on modal/route change, no visible focus ring.
- **Derived state in state:** storing what you can compute → sync bugs. Compute during render or memoize.
- **Uncontrolled/controlled mixups:** switching an input between the two, or forgetting `value`+`onChange`.
- **Debounce without cancel:** debouncing the request but not cancelling in-flight ones → out-of-order results overwrite fresh data.
- **Over-rendering:** unstable inline objects/functions as props/deps; missing memoization on hot paths.

### Interview Questions — Machine Coding

**You have 75 minutes to build an autocomplete. How do you spend the time and what do you cut first?**
> First 5 minutes clarifying scope and a11y bar and sketching the component API and state shape. Then vertical slices with a commit each: static markup, controlled input + mock fetch, debounce + cancellation, keyboard navigation and ARIA, then loading/empty/error states. Accessibility goes in as I build, not at the end. If time is short I cut result caching, virtualization, and highlighting — never keyboard support or request cancellation, because those are the correctness/quality signals interviewers weight most. I'd end by naming the tests I'd write and the extension points I deferred.

**How do you decide a component's API — controlled vs uncontrolled, and how many props?**
> I default to controlled (`value` + `onChange`) when the parent needs the state or must sync it elsewhere, and support uncontrolled with a `defaultValue` for simple drop-in use. I keep the prop surface minimal and predictable, prefer composition (children/slots) over a config prop that balloons with booleans, give sensible defaults so the common case is zero-config, and expose escape hatches (render props or `aria-*`/`className` passthrough) rather than a prop per variation. The test is: can someone use it correctly without reading the source?

**Walk me through preventing race conditions and leaks in a component that fetches on prop change.**
> Every fetch effect creates an AbortController, aborts in the cleanup function, and ignores AbortErrors — so changing the input or unmounting cancels the previous request and no stale response overwrites fresh data or sets state after unmount. For debounced inputs I also clear the timer in cleanup. If I can't abort (non-fetch async), I use an `ignore` flag captured per-effect-run. I'd verify by rapidly changing the input and asserting only the latest result renders, and by unmounting mid-flight with no console warning.

**Your interviewer says accessibility matters. What do you make sure works before you call the widget done?**
> Full keyboard operability (Tab to reach it, arrow/Enter/Esc as the APG pattern for that widget dictates), correct semantic elements or roles (button not div, listbox/option, dialog with aria-modal), proper labels (label/aria-label/aria-labelledby), focus management (trap and restore for modals, roving tabindex for composites, move focus on route change), a visible focus indicator, and live-region announcements for async state. I sanity-check with keyboard-only navigation and, if available, a screen reader — and I can cite the APG pattern I'm implementing.

**When would you reach for `useSyncExternalStore` over Context + reducer for shared state?**
> When state is read by many components across the tree and updates are frequent, or when I'm building a reusable external store. Context re-renders every consumer on any value change and a context+reducer store can tear under concurrent rendering. `useSyncExternalStore` subscribes components to a store and, with a selector, re-renders only those whose slice changed, and it's concurrency-safe with SSR support. For small, localized shared state (theme, current user) Context is simpler and fine — I don't reach for the store until the re-render cost or tearing is real.

**Halfway through, the interviewer changes the requirement — the list is now 50,000 rows. What do you do?**
> I confirm the constraint, then switch the list rendering to virtualization (windowing) so DOM nodes stay bounded to the viewport regardless of size, keying by stable ids not index. I move data loading to cursor pagination or a windowed data source rather than holding all rows in one render, memoize row components, and ensure keyboard/screen-reader access still works (or provide a "load more" alternative). I say the trade-off out loud — added complexity for bounded memory and stable frame rate — and note I'd measure with a long-task/INP check on a throttled CPU before calling it done.
