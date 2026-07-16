# JavaScript & TypeScript Core Fundamentals

> The deep-fundamentals screening round: event loop, closures, `this`, prototypes, async, memory, and the TypeScript type system — at the depth a lead is expected to reason from first principles and explain trade-offs.

## The Event Loop

JavaScript runs on a single thread with one **call stack**. Concurrency is achieved by offloading work (timers, I/O, network) to the host (browser/Node) and scheduling callbacks back onto the stack via queues.

### The pieces

- **Call stack** — synchronous execution frames. The engine runs to completion; nothing else touches the stack until it empties.
- **Macrotask queue** (a.k.a. task queue) — `setTimeout`/`setInterval` callbacks, DOM events, `MessageChannel`, I/O. One macrotask is processed per loop tick.
- **Microtask queue** — Promise reactions (`.then`/`.catch`/`.finally`), `await` continuations, `queueMicrotask`, `MutationObserver`. Drained **completely** after each macrotask and after the initial script, before rendering.
- **Render steps** — `requestAnimationFrame` callbacks, then style/layout/paint, run between macrotasks (roughly aligned to ~60fps), *after* microtasks are drained.

### One loop iteration (conceptually)

1. Pop and run one macrotask to completion (the initial script counts as the first).
2. Drain the **entire** microtask queue (microtasks can enqueue more microtasks — those run too).
3. If it's time to render: run `requestAnimationFrame` callbacks, then style/layout/paint.
4. Go back to step 1.

### Why Promise callbacks beat `setTimeout(0)`

`setTimeout(0)` schedules a **macrotask**; the Promise `.then` schedules a **microtask**. After the current synchronous run finishes, the engine drains all microtasks *before* picking up the next macrotask. So the Promise callback always runs first — and the `4ms` minimum clamp on nested timers makes `setTimeout` even later in practice.

### Ordering example

```js
console.log('1: sync start');

setTimeout(() => console.log('2: setTimeout'), 0);

Promise.resolve()
  .then(() => console.log('3: promise then'))
  .then(() => console.log('4: promise then chained'));

queueMicrotask(() => console.log('5: queueMicrotask'));

(async () => {
  console.log('6: async sync part');
  await null;                       // suspends; continuation is a microtask
  console.log('7: after await');
})();

console.log('8: sync end');
```

Output:

```
1: sync start
6: async sync part
8: sync end
3: promise then
5: queueMicrotask
7: after await
4: promise then chained
2: setTimeout
```

Explanation: all synchronous code (`1, 6, 8`) runs first. The body up to the first `await` runs synchronously, which is why `6` prints before `8`. Then microtasks drain in enqueue order: `3` (first `.then`), `5` (`queueMicrotask`), `7` (await continuation). The second `.then` (`4`) was only enqueued once `3` resolved, so it lands after the initial batch. Finally the macrotask `2` runs.

### Starvation

Because microtasks drain *fully* before the next macrotask or render, a microtask that keeps enqueuing more microtasks starves the event loop — timers never fire and the page never paints (frozen UI). This is the classic footgun of recursive `Promise.resolve().then(...)` loops. For long-running work that must yield, prefer `setTimeout`, `MessageChannel`, `scheduler.postTask`, or chunking via `requestIdleCallback`.

**How you'd verify:** log with `performance.now()` timestamps, or use Chrome DevTools Performance panel — the flame chart separates "Run Microtasks" from task boundaries and shows dropped frames when the main thread is blocked.

## Closures, Scope, Hoisting

A **closure** is a function together with the lexical environment it was defined in. The inner function retains access to outer variables even after the outer function returns — because it holds a live reference to that scope, not a copy.

```js
function counter() {
  let count = 0;                    // captured by closure
  return () => ++count;
}
const next = counter();
next(); // 1
next(); // 2  — same lexical environment persists
```

### The classic loop gotcha

```js
// var: single shared binding — all callbacks see the final value
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 0); // 3, 3, 3
}

// let: a fresh binding per iteration
for (let i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 0); // 0, 1, 2
}
```

`var` is function-scoped, so every callback closes over the same `i`. `let` creates a new binding on each iteration, so each closure captures its own.

### Hoisting and the TDZ

- `var` declarations are hoisted and initialized to `undefined`.
- `let`/`const` are hoisted but **not initialized** — they sit in the **Temporal Dead Zone** from the start of the block until the declaration executes. Accessing them there throws `ReferenceError`.
- `function` declarations are fully hoisted (usable before their line). Function *expressions* and arrow functions follow their variable's rules.

```js
console.log(x); // undefined  (var hoisted)
console.log(y); // ReferenceError (TDZ)
var x = 1;
let y = 2;
```

| Keyword | Scope | Hoisted | Re-assign | Re-declare |
|---|---|---|---|---|
| `var` | function | yes → `undefined` | yes | yes |
| `let` | block | yes → TDZ | yes | no |
| `const` | block | yes → TDZ | no | no |

`const` prevents rebinding the variable, not mutation of the referenced object — `const o = {}; o.a = 1` is legal.

## `this` Binding

`this` is determined by **how a function is called**, not where it's defined (except arrow functions). Resolution order:

1. **`new` binding** — `new Fn()` sets `this` to the freshly created object.
2. **Explicit binding** — `fn.call(obj)`, `fn.apply(obj)`, `fn.bind(obj)` force `this = obj`. `bind` wins over later `call`.
3. **Implicit binding** — `obj.method()` sets `this = obj`. Beware: extracting the method (`const m = obj.method`) loses the receiver.
4. **Default binding** — plain call `fn()` gives `undefined` in strict mode, the global object in sloppy mode.
5. **Arrow functions** — no own `this`; they capture `this` lexically from the enclosing scope. `call`/`bind` cannot change it.

```js
const obj = {
  val: 42,
  regular() { return this.val; },
  arrow: () => this?.val,           // `this` = enclosing (module/undefined)
};
obj.regular();               // 42  (implicit)
const f = obj.regular;
f();                         // undefined / throws (default, lost receiver)
f.call(obj);                 // 42  (explicit)
```

### call / apply / bind

- `call(thisArg, a, b)` — invoke now, args listed.
- `apply(thisArg, [a, b])` — invoke now, args as array.
- `bind(thisArg, ...partial)` — return a new permanently-bound function (also enables partial application).

Common lead-level catch: arrow functions are the right tool for callbacks/class fields that need lexical `this` (e.g. React handlers), but the *wrong* tool for object methods relying on the call-site receiver, and they can't be used as constructors or generators.

## Prototypes & Inheritance

Every object has an internal `[[Prototype]]` link. Property lookups walk the **prototype chain** until found or `null`.

- `prototype` — a property on **constructor functions**; the object that becomes the `[[Prototype]]` of instances created via `new`.
- `__proto__` — a (legacy) accessor for an object's `[[Prototype]]`. Prefer `Object.getPrototypeOf` / `Object.setPrototypeOf`.

```js
function Animal(name) { this.name = name; }
Animal.prototype.speak = function () { return `${this.name} makes a sound`; };

const dog = new Animal('Rex');
Object.getPrototypeOf(dog) === Animal.prototype; // true
dog.speak(); // "Rex makes a sound" — found via the chain
```

### `class` is syntactic sugar

```js
class Animal {
  constructor(name) { this.name = name; }
  speak() { return `${this.name} makes a sound`; }   // on Animal.prototype
}
class Dog extends Animal {
  speak() { return `${super.speak()} (woof)`; }       // super → parent prototype
}
```

Under the hood, methods live on `Animal.prototype`, `extends` wires the prototype chains (both instance and static), and `super` references the parent prototype. Differences from function constructors: class bodies are strict-mode, non-enumerable methods, not hoisted (TDZ), and must be called with `new`. `#private` fields are true hard-private (enforced by the engine, not convention).

## Promises, async/await, Concurrency

A Promise is a state machine: **pending → fulfilled | rejected**, settling exactly once. `.then` callbacks are always async (microtasks), even on an already-resolved promise — guaranteeing consistent ordering.

```js
const p = new Promise((resolve, reject) => {
  // executor runs synchronously
  setTimeout(() => resolve('done'), 100);
});
p.then(v => v.toUpperCase()); // returns a NEW promise; chaining flattens thenables
```

### async/await

`async` functions return a promise; `await` suspends the function and schedules its continuation as a microtask when the awaited value settles. It's syntactic sugar over `.then`, but reads sequentially.

```js
async function load() {
  try {
    const res = await fetch('/api');
    if (!res.ok) throw new Error(`HTTP ${res.status}`);
    return await res.json();
  } catch (err) {
    // catches both rejections and thrown errors
    throw new AppError('load failed', { cause: err });
  } finally {
    // always runs — cleanup
  }
}
```

**Sequential vs parallel** — a common perf bug is awaiting independent work serially:

```js
// SLOW: sequential, total = a + b
const x = await fetchA();
const y = await fetchB();

// FAST: concurrent, total = max(a, b)
const [x, y] = await Promise.all([fetchA(), fetchB()]);
```

### Combinators

| Combinator | Resolves when | Rejects when | Result |
|---|---|---|---|
| `Promise.all` | all fulfill | first rejection | array of values / first error (short-circuits) |
| `Promise.allSettled` | all settle | never | array of `{status, value|reason}` |
| `Promise.race` | first settles | first settles (if rejection) | first settled outcome |
| `Promise.any` | first fulfills | all reject | first value / `AggregateError` |

Use `allSettled` when you want every result regardless of individual failures (e.g. dashboards). Use `all` when any failure invalidates the batch. Use `race` for timeouts (race against a `setTimeout` reject) and `any` for redundant fallbacks (first healthy replica wins).

### Concurrency control

Unbounded `Promise.all` over thousands of items exhausts connections. Bound it with a worker-pool pattern:

```js
async function mapLimit(items, limit, fn) {
  const results = new Array(items.length);
  let i = 0;
  const workers = Array.from({ length: limit }, async () => {
    while (i < items.length) {
      const idx = i++;                 // atomic on single thread
      results[idx] = await fn(items[idx], idx);
    }
  });
  await Promise.all(workers);
  return results;
}
```

**Error handling pitfalls:** an unhandled rejection is a runtime hazard (listen for `unhandledrejection`); `await` inside a `.forEach` doesn't wait (use `for...of`); and swallowing errors in a `.catch` that returns a value silently converts a rejection to a fulfillment.

## Generators & Iterators

An **iterator** has a `next()` returning `{ value, done }`; an **iterable** exposes `[Symbol.iterator]()`. `for...of`, spread, and destructuring consume iterables.

**Generators** (`function*`) produce iterators and can pause with `yield`, enabling lazy sequences and cooperative control flow.

```js
function* range(start, end) {
  for (let i = start; i < end; i++) yield i;   // lazy — computed on demand
}
[...range(0, 3)]; // [0, 1, 2]
```

`yield` is bidirectional (`const x = yield v` receives the value passed to `.next(x)`), which is how libraries like redux-saga model effects. Async iterators (`for await...of`, `async function*`) stream async data such as paginated APIs.

## Memory Leaks & Garbage Collection

JS engines use **mark-and-sweep**: starting from roots (global, stack, closures), the collector marks all reachable objects; unmarked objects are swept. An object leaks when it stays *reachable* but is no longer *needed*.

### Common leak sources in the browser

- **Detached DOM nodes** — you remove a node from the document but a JS variable (or an array/cache) still references it, keeping it and its subtree alive.
- **Forgotten timers/listeners** — `setInterval` and `addEventListener` hold references to their callbacks (and everything the callbacks close over) until explicitly cleared/removed. Always pair `add`/`remove`, `set`/`clear`; use `AbortController` for listeners.
- **Closures over large objects** — a small long-lived callback closing over a big object pins that object for the callback's lifetime.
- **Unbounded caches / global maps** — Maps keyed by objects that are never deleted grow forever.

```js
// Leak: interval keeps `bigData` alive forever
const id = setInterval(() => process(bigData), 1000);
// Fix: clearInterval(id) when done, e.g. in cleanup / useEffect return
```

### Weak references

- **`WeakMap` / `WeakSet`** — keys are held *weakly*; if the key object is otherwise unreachable it (and its map entry) can be collected. Not iterable, no `size`. Ideal for associating metadata with DOM nodes or caching by object identity without preventing GC.
- **`WeakRef`** — a weak reference to a single object; `.deref()` returns the object or `undefined` if collected. Use sparingly (with `FinalizationRegistry` for cleanup) — GC timing is non-deterministic.

**How you'd verify:** Chrome DevTools → Memory → take a **heap snapshot**, interact, take another, and use the "Comparison" view or the **Detached** filter to find detached nodes; the **Allocation instrument on timeline** shows retained size growing over repeated actions. A leak shows as a sawtooth that trends upward instead of returning to baseline after GC.

## Coercion, Equality, Copying

### `==` vs `===`

`===` compares type and value with no coercion. `==` applies the abstract equality algorithm (type coercion), which produces surprising results. Rule of thumb: **always `===`**, with one pragmatic exception — `x == null` cleanly matches both `null` and `undefined`.

```js
0 == '';        // true
0 == '0';       // true
'' == '0';      // false   (non-transitive!)
null == undefined; // true
NaN === NaN;    // false   (use Number.isNaN / Object.is)
[] == ![];      // true    (coercion chaos)
```

Use `Object.is` for edge cases: it distinguishes `+0`/`-0` and treats `NaN` as equal to itself.

### Immutability

`const` binds; `Object.freeze` shallow-freezes (nested objects stay mutable). Deep immutability requires recursive freezing or a library. Immutable update patterns (spread, not mutation) are what make change-detection cheap in React/Redux.

### Shallow vs deep copy

```js
const shallow = { ...obj };              // top level copied, nested shared
const shallow2 = Object.assign({}, obj); // same
const deepJSON = JSON.parse(JSON.stringify(obj)); // loses Date/Map/undefined/functions, breaks cycles
const deep = structuredClone(obj);       // handles Date, Map, Set, cycles, typed arrays
```

`structuredClone` is the modern deep-copy primitive: it uses the structured-clone algorithm, preserves cyclic references and many built-in types, but **cannot** clone functions, DOM nodes, or class prototypes (you get a plain object) and throws on non-cloneables.

## Modules: ESM vs CommonJS

| | ESM (`import`/`export`) | CommonJS (`require`/`module.exports`) |
|---|---|---|
| Loading | static, async, hoisted | dynamic, synchronous |
| Bindings | live read-only *bindings* | copy of the value at require time |
| Analysis | statically analyzable → tree-shaking | runtime, harder to tree-shake |
| `this` at top level | `undefined` | `module.exports` |
| Interop | can import CJS | `require` of ESM needs dynamic `import()` |

ESM is the standard for the browser and modern tooling; its static structure enables tree-shaking and top-level `await`. CJS still dominates legacy Node. Dual-package hazards and `"type": "module"` / `.mjs`/`.cjs` resolution are frequent real-world friction points.

## TypeScript

### Structural typing

TS types are **structural** ("duck typing"): compatibility is by shape, not by name. A value is assignable if it has at least the required members. This is why an object literal with the right fields satisfies an interface it never declared — though *excess property checks* apply to fresh object literals.

### `any` vs `unknown` vs `never`

- **`any`** — opts out of checking; infectious, disables safety. Avoid.
- **`unknown`** — the type-safe top type: accepts anything, but you must narrow before use. The correct type for untrusted input (JSON, `catch`).
- **`never`** — the bottom type: no value inhabits it. Return type of functions that never return, the type of exhaustively-handled union branches, and empty-array/`throw` positions.

```js
function assertNever(x: never): never { throw new Error(`Unexpected: ${x}`); }
```

### Generics

```ts
function identity<T>(x: T): T { return x; }

// constraints + inference
function prop<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}
```

Use `extends` for constraints, default type params (`<T = string>`), and let inference do the work — over-annotating generic call sites is a smell.

### Utility types cheat-sheet

| Utility | Effect |
|---|---|
| `Partial<T>` | all props optional |
| `Required<T>` | all props required |
| `Readonly<T>` | all props readonly |
| `Pick<T, K>` | subset of keys `K` |
| `Omit<T, K>` | `T` without keys `K` |
| `Record<K, V>` | object with keys `K`, values `V` |
| `ReturnType<F>` | return type of function type `F` |
| `Parameters<F>` | tuple of param types |
| `Exclude<U, X>` / `Extract<U, X>` | filter union members |
| `NonNullable<T>` | remove `null`/`undefined` |
| `Awaited<T>` | unwrap Promise |

### Discriminated unions + exhaustiveness

```ts
type Shape =
  | { kind: 'circle'; r: number }
  | { kind: 'rect'; w: number; h: number };

function area(s: Shape): number {
  switch (s.kind) {                 // discriminant narrows each branch
    case 'circle': return Math.PI * s.r ** 2;
    case 'rect':   return s.w * s.h;
    default:       return assertNever(s); // compile error if a case is missed
  }
}
```

This is the single most valuable modeling tool: adding a new variant forces every `switch` to be updated (via the `never` check).

### Narrowing & type guards

TS narrows via `typeof`, `instanceof`, `in`, equality, and truthiness. Custom **type guards** and **assertion functions** extend this:

```ts
function isString(x: unknown): x is string {   // type predicate
  return typeof x === 'string';
}
function assert(cond: unknown, msg: string): asserts cond {
  if (!cond) throw new Error(msg);
}
```

### `keyof`, `typeof`, indexed access

```ts
const config = { host: 'x', port: 8080 } as const;
type Config = typeof config;          // { readonly host: 'x'; readonly port: 8080 }
type Keys = keyof Config;             // "host" | "port"
type Port = Config['port'];           // 8080  (indexed access)
type Values = Config[keyof Config];   // "x" | 8080
```

### Mapped & conditional types

```ts
// mapped
type Nullable<T> = { [K in keyof T]: T[K] | null };
type Getters<T> = { [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K] };

// conditional + infer
type ElementType<T> = T extends (infer U)[] ? U : T;
type Flatten<T> = T extends Array<infer U> ? U : never;
```

### `satisfies`

`satisfies` checks a value against a type **without widening** it — you keep the precise inferred literal types while still validating the shape. Contrast with a type annotation, which widens.

```ts
const routes = {
  home: '/',
  user: '/user/:id',
} satisfies Record<string, string>;
// routes.home is '/'  (literal preserved), and typos in values still error
```

### Declaration merging

Interfaces with the same name merge; namespaces merge with interfaces/functions. Used to augment third-party types (`declare module`), extend `Window`, or add properties to libraries.

```ts
declare global {
  interface Window { __APP_CONFIG__: AppConfig; }
}
```

### Enums vs union literals

Prefer **union of string literals** (`type Status = 'active' | 'idle'`) over `enum` in most code: no runtime footprint, better tree-shaking, direct JSON compatibility, and simpler narrowing. `enum` generates real runtime objects (numeric enums are reverse-mapped and error-prone); `const enum` inlines but breaks under isolated-modules/Babel. Reach for enums mainly when you need a named runtime namespace of values.

### Interview Questions — JavaScript & TypeScript

**Walk me through what prints and why: a `console.log`, a `setTimeout(0)`, a resolved `Promise.then`, and another `console.log`.**

> The two synchronous `console.log`s print first, in order, because synchronous code runs to completion on the call stack before anything queued. Then the event loop drains the microtask queue — the `Promise.then` callback runs next. The `setTimeout` callback is a macrotask and runs last, after all microtasks. The key insight is that after each macrotask (including the initial script) the engine fully drains microtasks before the next macrotask or a render, which is why a resolved promise always beats `setTimeout(0)`.

**What is a closure, and when has it bitten you in production?**

> A closure is a function bundled with the lexical scope it was defined in, keeping outer variables alive. The classic bug is a `var` loop where every deferred callback shares one binding and sees the final value — fixed with `let` (per-iteration binding). In production, the subtler bite is memory: a long-lived closure (an event listener, a cached callback) closing over a large object pins that object in memory for the listener's lifetime, which surfaces as a slow heap leak.

**Explain the `this` binding rules and why arrow functions are different.**

> `this` is set by the call site, resolved as: `new` > explicit (`call`/`apply`/`bind`) > implicit (`obj.method()`) > default (`undefined` in strict mode). Arrow functions break this rule — they have no own `this` and capture it lexically from the enclosing scope, and `call`/`bind` can't override it. So arrows are ideal for callbacks needing the surrounding `this` (React handlers, `setTimeout` bodies) but wrong for object methods that depend on the receiver, and they can't be constructors.

**Difference between `prototype` and `__proto__`, and what `class` actually compiles to.**

> `prototype` is a property on constructor functions — the object assigned as the `[[Prototype]]` of instances they create. `__proto__` is a legacy accessor for an object's actual `[[Prototype]]` link (prefer `Object.getPrototypeOf`). `class` is syntactic sugar: methods land on `Constructor.prototype`, `extends` wires up both the instance and static prototype chains, and `super` calls the parent prototype. The differences that matter: classes are strict-mode, non-hoisted (TDZ), must be called with `new`, and `#fields` are truly private.

**When would you use `Promise.allSettled` over `Promise.all`, and how do you bound concurrency?**

> `Promise.all` short-circuits on the first rejection — use it when any failure invalidates the whole batch. `allSettled` never rejects and returns every outcome — use it for dashboards or best-effort fan-out where partial success is fine. To bound concurrency I don't hand `Promise.all` a thousand promises (that exhausts connections); I use a worker-pool: N async workers pulling from a shared index/queue, so at most N requests are in flight. `race` covers timeouts, `any` covers redundant fallbacks.

**How does `await` interact with the event loop?**

> `await` suspends the async function and registers its continuation as a microtask that resumes when the awaited value settles. Everything up to the first `await` runs synchronously. So the code reads sequentially but the continuations are microtasks, ordered relative to other promise callbacks. A frequent bug is awaiting independent operations one after another instead of `Promise.all`-ing them, turning `max(a,b)` latency into `a+b`.

**How do you find and fix a memory leak in a single-page app?**

> First reproduce it: perform the suspect action repeatedly and watch the heap in DevTools — a leak trends upward after GC instead of returning to baseline. Then take two heap snapshots (before/after) and diff them; the Detached filter surfaces DOM nodes still referenced by JS. Common culprits are forgotten `setInterval`/`addEventListener` (fix: pair with clear/remove or `AbortController`), detached nodes held by caches, and closures over large objects. For object-keyed caches I use `WeakMap` so entries are collectable when the key dies.

**`any` vs `unknown` vs `never` — when do you use each?**

> `unknown` is the safe top type — I use it for untrusted input (parsed JSON, `catch` clauses) and force narrowing before use. `any` disables checking and is infectious, so I ban it except at truly untyped boundaries. `never` is the bottom type: it's the return of functions that throw or never return, and I use it for exhaustiveness — an `assertNever(x: never)` in a `switch` default turns a missed discriminated-union case into a compile error.

**What are discriminated unions and why are they the workhorse of TS modeling?**

> A discriminated union is a set of object types sharing a literal-typed discriminant field (`kind`). Switching on that field narrows each branch to its exact shape. They're powerful because they make illegal states unrepresentable and, combined with a `never` exhaustiveness check, adding a new variant forces the compiler to flag every place that must handle it. That's compile-time-enforced completeness across the codebase.

**What does `satisfies` give you that a type annotation doesn't?**

> A type annotation validates *and widens* — `const x: Record<string,string> = {...}` loses the literal value types. `satisfies` validates the value against the type but preserves the narrow inferred type, so you get both the error-checking (typos, missing keys) and precise literal/key types for downstream inference. It's the right tool for config objects and lookup tables where you want validation without losing specificity.

**Explain structural typing and one place it surprises people.**

> TS checks assignability by shape, not by declared name — if a value has the required members it's compatible, even if it never named the interface. The surprise is *excess property checks*: a fresh object literal assigned directly is flagged for extra properties, but the same object via a variable passes, because structural compatibility only requires the target's members be present. It also means two unrelated types with identical shapes are interchangeable.

**Shallow vs deep copy — what does `structuredClone` handle and not handle?**

> Spread and `Object.assign` copy only the top level; nested objects stay shared references. `JSON.parse(JSON.stringify())` deep-copies but drops `undefined`/functions, mangles `Date` to strings, loses `Map`/`Set`, and throws on cycles. `structuredClone` is the modern primitive: it handles `Date`, `Map`, `Set`, typed arrays, and cyclic references, but can't clone functions or DOM nodes and downgrades class instances to plain objects. For truly custom deep-clone semantics I'd still reach for a purpose-built function.

**Why prefer union string literals over enums?**

> Union literals have zero runtime footprint, tree-shake cleanly, serialize directly to/from JSON, and narrow naturally in switches. `enum` emits a real runtime object; numeric enums add reverse mappings and accept any number, and `const enum` inlines but breaks under `isolatedModules`/Babel transpilation. I use enums only when I genuinely need a named runtime namespace of values — otherwise union literals are lighter and safer.

**How would you verify event-loop ordering assumptions rather than guessing?**

> I'd instrument with `performance.now()` timestamps around each callback and log them, which makes the microtask-before-macrotask boundary visible. In the browser I use the DevTools Performance panel — the flame chart labels "Run Microtasks" separately from task boundaries and shows dropped frames when microtask starvation blocks rendering. For starvation specifically, a recursive `queueMicrotask` loop will freeze the UI and never fire a pending `setTimeout`, which confirms microtasks drain fully before macrotasks.
