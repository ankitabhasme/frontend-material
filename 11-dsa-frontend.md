# DSA for Frontend — The Coding Round

> Data structures and algorithms as they actually show up in a frontend interview: utility functions, tree traversals, and the handful of patterns that cover 80% of screens.

## Big-O Primer

Big-O describes how work grows as input `n` grows — it ignores constants and lower-order terms, so it measures *scaling*, not raw speed. A lead should reason about it out loud during the screen, not recite it.

- **Time complexity** — number of operations as a function of `n`.
- **Space complexity** — extra memory allocated, *including recursion stack depth* (a common miss).

### Common complexities (best → worst)

| Notation | Name | Typical source | Example |
|---|---|---|---|
| `O(1)` | constant | hash/array index, `Map.get` | LRU lookup |
| `O(log n)` | logarithmic | halving the search space | binary search, balanced tree |
| `O(n)` | linear | single pass | array scan, BFS/DFS over a tree |
| `O(n log n)` | linearithmic | sort-based | `Array.sort`, merge sort |
| `O(n²)` | quadratic | nested loop over same input | naive dedupe, bubble sort |
| `O(2ⁿ)` | exponential | branching recursion w/o memo | naive Fibonacci, subsets |
| `O(n!)` | factorial | permutations | brute-force TSP |

### How to reason about it

- **Count nested passes over the input.** Two independent loops = `O(n + m)`, nested loops over the same data = `O(n²)`.
- **A hashmap turns "search inside a loop" from `O(n²)` into `O(n)`** by trading space for time — the single most common FE optimization.
- **Recursion space = max stack depth.** A DFS on a skewed tree is `O(h)` space where `h` can be `n`.
- **Amortized vs worst-case.** `Array.push` is amortized `O(1)` despite occasional resizes. Say "amortized" when it applies — interviewers notice.
- **Sorting is the usual `n log n` floor.** If your solution needs a sort, that's your lower bound unless you can bucket/count.

## Trees and Graphs Are Everywhere in FE

The mental model that makes FE DSA click:

- **The DOM / component tree is a tree** — nodes with a single parent and ordered children. Rendering, event bubbling, reconciliation, and `querySelector` are all tree walks.
- **JSON is a tree** (or a graph once you allow shared references / cycles). Deep clone, serialization, and state normalization are traversals.
- **Traversal matters:** BFS (queue, level-order) vs DFS (stack/recursion, depth-first). Choose BFS for "nearest / shallowest" and DFS for "does a path exist / full subtree."

---

## FE Utility Challenges

These are the "implement X from scratch" questions. Interviewers care about edge cases (`this` binding, cancellation, cycles) as much as the happy path.

### `debounce` — leading/trailing + cancel

Delays invocation until `wait` ms of silence. `leading` fires on the first call, `trailing` fires after the pause.

```ts
type DebounceOpts = { leading?: boolean; trailing?: boolean };

function debounce<T extends (...args: any[]) => any>(
  fn: T,
  wait: number,
  { leading = false, trailing = true }: DebounceOpts = {}
) {
  let timer: ReturnType<typeof setTimeout> | null = null;
  let lastArgs: Parameters<T> | null = null;
  let lastThis: any;

  function invoke() {
    fn.apply(lastThis, lastArgs!);
    lastArgs = null;
  }

  function debounced(this: any, ...args: Parameters<T>) {
    lastArgs = args;
    lastThis = this;
    const callNow = leading && !timer;
    if (timer) clearTimeout(timer);
    timer = setTimeout(() => {
      timer = null;
      if (trailing && lastArgs) invoke();
    }, wait);
    if (callNow) invoke();
  }

  debounced.cancel = () => {
    if (timer) clearTimeout(timer);
    timer = null;
    lastArgs = null;
  };
  debounced.flush = () => {
    if (timer && lastArgs) invoke();
    debounced.cancel();
  };
  return debounced;
}
```

Each call is `O(1)`. Key traps: preserve `this` (arrow methods break it), forward args, and reset `timer = null` inside the timeout so `leading` can fire again.

### `throttle`

Guarantees at most one call per `wait` window — unlike debounce, it fires *during* a burst, not after.

```ts
function throttle<T extends (...args: any[]) => any>(fn: T, wait: number) {
  let last = 0;
  let timer: ReturnType<typeof setTimeout> | null = null;
  let lastArgs: Parameters<T> | null = null;

  return function (this: any, ...args: Parameters<T>) {
    const now = Date.now();
    const remaining = wait - (now - last);
    lastArgs = args;
    if (remaining <= 0) {
      if (timer) { clearTimeout(timer); timer = null; }
      last = now;
      fn.apply(this, args);
    } else if (!timer) {
      // trailing edge: fire once more at the end of the window
      timer = setTimeout(() => {
        last = Date.now();
        timer = null;
        fn.apply(this, lastArgs!);
      }, remaining);
    }
  };
}
```

`debounce` = "wait until it's quiet"; `throttle` = "at most once per interval." Scroll/resize → throttle; search-as-you-type → debounce.

### `Promise.all`

Resolves with an ordered array once *all* resolve; rejects on the *first* rejection.

```ts
function promiseAll<T>(promises: Iterable<Promise<T> | T>): Promise<T[]> {
  const items = [...promises];
  return new Promise((resolve, reject) => {
    const results: T[] = new Array(items.length);
    let remaining = items.length;
    if (remaining === 0) return resolve(results);
    items.forEach((p, i) => {
      Promise.resolve(p).then(
        (val) => {
          results[i] = val;          // index preserves order, not completion time
          if (--remaining === 0) resolve(results);
        },
        reject                       // first rejection wins; later ones are ignored
      );
    });
  });
}
```

`O(n)` to schedule; wall-clock is the slowest promise. The order/index detail is the whole point of the question.

### `Promise.allSettled`

Never rejects — reports each outcome.

```ts
type Settled<T> =
  | { status: 'fulfilled'; value: T }
  | { status: 'rejected'; reason: unknown };

function allSettled<T>(promises: Array<Promise<T> | T>): Promise<Settled<T>[]> {
  return promiseAll(
    promises.map((p) =>
      Promise.resolve(p).then(
        (value) => ({ status: 'fulfilled' as const, value }),
        (reason) => ({ status: 'rejected' as const, reason })
      )
    )
  );
}
```

Use when you want *all* results regardless of failures (e.g. dashboard widgets that each load independently).

### `retry` with exponential backoff

```ts
async function retry<T>(
  fn: () => Promise<T>,
  { retries = 3, baseMs = 200, factor = 2, jitter = true }: {
    retries?: number; baseMs?: number; factor?: number; jitter?: boolean;
  } = {}
): Promise<T> {
  let attempt = 0;
  for (;;) {
    try {
      return await fn();
    } catch (err) {
      if (attempt >= retries) throw err;
      const delay = baseMs * factor ** attempt;
      const wait = jitter ? delay * (0.5 + Math.random() / 2) : delay;
      await new Promise((r) => setTimeout(r, wait));
      attempt++;
    }
  }
}
```

Backoff avoids hammering a struggling server; **jitter** prevents the thundering-herd where all clients retry in lockstep. Lead-level nuance: only retry *idempotent* / transient failures (network, 429/503), never a 400.

### `memoize`

```ts
function memoize<A extends unknown[], R>(
  fn: (...args: A) => R,
  keyFn: (...args: A) => string = (...a) => JSON.stringify(a)
): (...args: A) => R {
  const cache = new Map<string, R>();
  return (...args: A): R => {
    const key = keyFn(...args);
    if (cache.has(key)) return cache.get(key)!;
    const result = fn(...args);
    cache.set(key, result);
    return result;
  };
}
```

Trades space for time — `O(1)` amortized on cache hits. Warn about unbounded growth (pair with an LRU) and that `JSON.stringify` keys are order-sensitive and drop functions/`undefined`.

### Deep clone (recursive, cycle-safe)

```ts
function deepClone<T>(value: T, seen = new WeakMap()): T {
  if (value === null || typeof value !== 'object') return value;
  if (seen.has(value as object)) return seen.get(value as object); // cycle guard
  if (value instanceof Date) return new Date(value) as T;
  if (value instanceof RegExp) return new RegExp(value.source, value.flags) as T;
  if (value instanceof Map) {
    const out = new Map();
    seen.set(value as object, out);
    value.forEach((v, k) => out.set(k, deepClone(v, seen)));
    return out as T;
  }
  if (value instanceof Set) {
    const out = new Set();
    seen.set(value as object, out);
    value.forEach((v) => out.add(deepClone(v, seen)));
    return out as T;
  }
  const out: any = Array.isArray(value) ? [] : {};
  seen.set(value as object, out);
  for (const key of Reflect.ownKeys(value as object)) {
    out[key] = deepClone((value as any)[key], seen);
  }
  return out;
}
```

`O(n)` in node count. The `WeakMap` of visited nodes is what makes it cycle-safe and preserves shared references. Mention `structuredClone()` as the built-in that also handles cycles (but not functions/DOM nodes).

### `flatten` — array and object-to-dot-paths

```ts
// Nested array → flat, to a given depth
function flattenArray<T>(arr: any[], depth = Infinity): T[] {
  const out: T[] = [];
  const walk = (a: any[], d: number) => {
    for (const item of a) {
      if (Array.isArray(item) && d > 0) walk(item, d - 1);
      else out.push(item);
    }
  };
  walk(arr, depth);
  return out;
}

// Nested object → { 'a.b.c': value }
function flattenObject(obj: Record<string, any>, prefix = ''): Record<string, any> {
  const out: Record<string, any> = {};
  for (const [k, v] of Object.entries(obj)) {
    const path = prefix ? `${prefix}.${k}` : k;
    if (v && typeof v === 'object' && !Array.isArray(v)) {
      Object.assign(out, flattenObject(v, path));
    } else {
      out[path] = v;
    }
  }
  return out;
}
```

Both are `O(n)` tree walks. Dot-path flattening is the classic "build a form-state / i18n key map" question.

### EventEmitter / pub-sub

```ts
type Handler = (...args: any[]) => void;

class EventEmitter {
  private events = new Map<string, Set<Handler>>();

  on(event: string, handler: Handler): () => void {
    if (!this.events.has(event)) this.events.set(event, new Set());
    this.events.get(event)!.add(handler);
    return () => this.off(event, handler); // return unsubscribe
  }
  once(event: string, handler: Handler): () => void {
    const wrap: Handler = (...args) => { this.off(event, wrap); handler(...args); };
    return this.on(event, wrap);
  }
  off(event: string, handler: Handler) {
    this.events.get(event)?.delete(handler);
  }
  emit(event: string, ...args: any[]) {
    // copy to a snapshot so handlers can safely unsubscribe during emit
    this.events.get(event) && [...this.events.get(event)!].forEach((h) => h(...args));
  }
}
```

`Set` gives `O(1)` add/remove and dedupes handlers. Returning an unsubscribe fn and snapshotting before emit are the details that separate senior answers.

### LRU cache (Map-based)

JS `Map` preserves insertion order, so the oldest key is always `map.keys().next().value`.

```ts
class LRUCache<K, V> {
  private map = new Map<K, V>();
  constructor(private capacity: number) {}

  get(key: K): V | undefined {
    if (!this.map.has(key)) return undefined;
    const val = this.map.get(key)!;
    this.map.delete(key);      // re-insert to mark most-recently-used
    this.map.set(key, val);
    return val;
  }
  set(key: K, value: V) {
    if (this.map.has(key)) this.map.delete(key);
    else if (this.map.size >= this.capacity) {
      this.map.delete(this.map.keys().next().value); // evict LRU
    }
    this.map.set(key, value);
  }
}
```

`get`/`set` are `O(1)`. The `Map` insertion-order trick avoids hand-rolling a doubly-linked list + hashmap (which is the "harder" version if they forbid `Map` ordering).

### `curry`

```ts
function curry(fn: (...args: any[]) => any) {
  return function curried(this: any, ...args: any[]) {
    return args.length >= fn.length
      ? fn.apply(this, args)
      : (...rest: any[]) => curried.apply(this, [...args, ...rest]);
  };
}
// curry((a,b,c)=>a+b+c)(1)(2)(3) === 6; also (1,2)(3), (1)(2,3)
```

Uses `fn.length` (declared arity) to decide when to invoke. `O(1)` per partial application.

### `pipe` / `compose`

```ts
const pipe = (...fns: Function[]) => (x: any) => fns.reduce((acc, fn) => fn(acc), x);
const compose = (...fns: Function[]) => (x: any) => fns.reduceRight((acc, fn) => fn(acc), x);
// pipe(f,g)(x) === g(f(x));  compose(f,g)(x) === f(g(x))
```

`pipe` reads left-to-right (data flow), `compose` right-to-left (math convention). Same `O(n)` in the number of functions.

### `once`

```ts
function once<T extends (...args: any[]) => any>(fn: T): T {
  let called = false;
  let result: ReturnType<T>;
  return function (this: any, ...args: Parameters<T>) {
    if (!called) { called = true; result = fn.apply(this, args); }
    return result;
  } as T;
}
```

Memoizes the first result and ignores subsequent calls — the basis for lazy init / singletons.

### `groupBy`

```ts
function groupBy<T, K extends PropertyKey>(
  items: T[],
  keyFn: (item: T) => K
): Record<K, T[]> {
  return items.reduce((acc, item) => {
    const key = keyFn(item);
    (acc[key] ??= []).push(item);
    return acc;
  }, {} as Record<K, T[]>);
}
```

Single `O(n)` pass — the canonical hashmap-bucketing pattern.

### Concurrency limiter (`pLimit` / async pool)

Runs at most `n` async tasks at once — essential for batching API calls without flooding the browser's connection pool.

```ts
function pLimit(concurrency: number) {
  let active = 0;
  const queue: Array<() => void> = [];

  const next = () => {
    if (active >= concurrency || queue.length === 0) return;
    active++;
    queue.shift()!();
  };

  return function run<T>(task: () => Promise<T>): Promise<T> {
    return new Promise<T>((resolve, reject) => {
      const exec = () => {
        task().then(resolve, reject).finally(() => {
          active--;
          next();
        });
      };
      queue.push(exec);
      next();
    });
  };
}
// const limit = pLimit(3);
// const results = await Promise.all(urls.map(u => limit(() => fetch(u))));
```

Preserves result order via `Promise.all` while capping in-flight work. Talk about backpressure and why the browser caps ~6 connections/host anyway.

---

## Core Algorithm Patterns

### Two pointers

Converging or same-direction indices — turns many `O(n²)` scans into `O(n)`.

```ts
// Is a string a palindrome? O(n) time, O(1) space
function isPalindrome(s: string): boolean {
  let i = 0, j = s.length - 1;
  while (i < j) if (s[i++] !== s[j--]) return false;
  return true;
}
```

### Sliding window

Maintain a moving range with an aggregate — for "longest/shortest substring/subarray" problems.

```ts
// Longest substring without repeating characters — O(n)
function longestUnique(s: string): number {
  const seen = new Map<string, number>();
  let start = 0, best = 0;
  for (let end = 0; end < s.length; end++) {
    const c = s[end];
    if (seen.has(c) && seen.get(c)! >= start) start = seen.get(c)! + 1;
    seen.set(c, end);
    best = Math.max(best, end - start + 1);
  }
  return best;
}
```

### Hashmap frequency

```ts
// First non-repeating character — O(n)
function firstUnique(s: string): string | null {
  const freq = new Map<string, number>();
  for (const c of s) freq.set(c, (freq.get(c) ?? 0) + 1);
  for (const c of s) if (freq.get(c) === 1) return c;
  return null;
}
```

Two passes, still `O(n)`. Frequency maps power anagrams, dedupe, and "find the pair" questions.

### Tree BFS / DFS — the FE flavor

```ts
interface TreeNode { value: string; children?: TreeNode[]; }

// DFS: serialize a tree to a flat, order-preserving list — O(n)
function serialize(root: TreeNode): string[] {
  const out: string[] = [];
  const dfs = (node: TreeNode) => {
    out.push(node.value);
    node.children?.forEach(dfs);
  };
  dfs(root);
  return out;
}

// BFS: find the shallowest node matching a predicate (like DOM querySelector) — O(n)
function findNode(root: TreeNode, pred: (n: TreeNode) => boolean): TreeNode | null {
  const queue: TreeNode[] = [root];
  while (queue.length) {
    const node = queue.shift()!;
    if (pred(node)) return node;
    if (node.children) queue.push(...node.children);
  }
  return null;
}
```

DFS = stack/recursion (serialize, compute subtree size, event capture/bubble order). BFS = queue (nearest match, level-order, shortest path in an unweighted graph). Both `O(n)` time; DFS is `O(h)` stack space, BFS is `O(w)` queue space (width).

### Recursion vs iteration

- **Recursion** is cleaner for trees and divide-and-conquer, but risks stack overflow on deep inputs (browsers cap ~10k–15k frames). Convert to an explicit stack when depth is unbounded (e.g. deeply nested JSON).
- **Iteration** avoids call-stack limits and is often faster (no frame overhead) but harder to read for tree work.
- **Tail-call optimization is not reliable in JS engines** — don't count on it. Prefer an explicit stack/queue for production tree walks over user-controlled data.

### Basic dynamic programming intuition

DP = recursion + memoization when subproblems overlap. Two moves: (1) find the recurrence, (2) cache overlapping subproblems (top-down memo) or build a table (bottom-up).

```ts
// Fibonacci: naive is O(2ⁿ); memoized is O(n) time / O(n) space
function fib(n: number, memo = new Map<number, number>()): number {
  if (n < 2) return n;
  if (memo.has(n)) return memo.get(n)!;
  const val = fib(n - 1, memo) + fib(n - 2, memo);
  memo.set(n, val);
  return val;
}
```

The FE-relevant framing: memoization is DP; a selector/derived-state cache (Reselect) is the same idea applied to render pipelines.

### Complexity cheat-sheet

| Pattern | Time | Space | When |
|---|---|---|---|
| Two pointers | `O(n)` | `O(1)` | sorted array, palindrome, pair-sum |
| Sliding window | `O(n)` | `O(k)` | longest/shortest subarray/substring |
| Hashmap frequency | `O(n)` | `O(n)` | dedupe, anagram, first-unique |
| BFS | `O(n)` | `O(w)` | shallowest match, level order |
| DFS | `O(n)` | `O(h)` | serialize, path exists, subtree |
| Binary search | `O(log n)` | `O(1)` | sorted lookup |
| DP (memo) | `O(states)` | `O(states)` | overlapping subproblems |

### Interview Questions — DSA

**Debounce vs throttle — when do you reach for each, and how would you implement leading-edge behavior?**

> Debounce collapses a burst into a single call *after* activity stops — ideal for search-as-you-type, resize-then-recompute, or auto-save. Throttle guarantees a steady cadence *during* a burst — at most one call per interval — ideal for scroll handlers, drag, and rate-limited analytics. Implementation-wise both hold a timer; debounce clears and resets it on every call so it only fires once things go quiet, while throttle records the last invocation time and ignores calls until the window elapses. Leading-edge debounce fires immediately when there's no active timer (first keystroke feels responsive) and then suppresses the trailing call unless configured otherwise; I always expose `cancel`/`flush` so React effects can clean up on unmount and avoid setting state after teardown.

**Walk me through how you'd deep-clone an object that may contain cycles.**

> A naive recursive clone recurses forever on a cycle (`a.self = a`). The fix is a `WeakMap` of already-visited source objects mapped to their clones: before recursing into an object I record its clone in the map, and if I ever revisit a node I return the stored clone instead of recursing. That both breaks cycles and preserves shared references so the clone's graph shape matches the original. I also special-case `Date`, `RegExp`, `Map`, and `Set` since a plain property copy loses their semantics. In modern environments I'd default to the built-in `structuredClone`, which handles cycles and typed data, and only hand-roll when I need to clone things it rejects — like functions or DOM nodes — where I'd document that those can't be truly cloned.

**Why is a `Map` enough to build an LRU cache in `O(1)`, and when isn't it?**

> JS `Map` preserves insertion order, so the least-recently-used key is always the first key from `map.keys()`. On `get`, I delete and re-insert the entry to move it to the "newest" end; on `set` past capacity, I evict `keys().next().value`. Delete and set are `O(1)`, so every operation is constant time without a manual linked list. It stops being sufficient when I need something the ordering doesn't give me cheaply — LFU (frequency-based) eviction, TTL expiry with a heap, or concurrent access — at which point I'd combine a hashmap with a doubly-linked list or a priority structure and accept the extra complexity.

**How do you keep a page responsive when firing 200 API calls?**

> Unbounded `Promise.all` over 200 requests floods the browser's ~6-connections-per-host limit, and the rest queue at the network layer while the main thread churns. I use a concurrency limiter (`pLimit`) that keeps at most N tasks in flight and pulls the next off a queue as each settles, so I get controlled parallelism with backpressure while still resolving results in order via `Promise.all`. I'd tune N to the workload, add retry-with-backoff-and-jitter for transient failures, and for truly large sets prefer server-side batching or pagination so the client isn't the bottleneck at all.

**When would you convert a recursive tree walk to an iterative one?**

> When the tree depth is unbounded or attacker/user-controlled — deeply nested JSON, an arbitrarily deep component tree — recursion risks a stack overflow because JS engines cap the call stack around 10–15k frames and don't reliably do tail-call optimization. I convert to an explicit stack (for DFS) or queue (for BFS) so depth is limited by heap, not call-stack, memory. For well-bounded trees I'll keep recursion because it's more readable, and I choose BFS when I want the shallowest match or level-order and DFS when I need full subtree work or path existence.
