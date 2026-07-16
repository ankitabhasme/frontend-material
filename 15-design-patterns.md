# Design Patterns for Frontend

> Classic GoF patterns, their React idioms, SOLID applied to components, and knowing when a pattern earns its keep versus when it is over-engineering.

## Why Patterns Matter at Lead Level

A lead is not judged on knowing pattern names — they are judged on **naming the force that made the pattern necessary**. Every pattern is a response to a change vector: "this will have many implementations" (strategy/factory), "this will have many observers" (observer), "this crosses a boundary we do not own" (adapter/facade). If you cannot name the change vector, you are decorating code, not designing it.

Frontend has its own dialect: most GoF patterns survive, but React reframes them as **composition of functions and components** rather than class hierarchies.

---

## Creational Patterns

### Module Pattern

The original JS encapsulation mechanism — a closure that exposes a public surface and hides internals. Largely subsumed by ES modules, but still relevant for singletons and controlled state.

```js
const analytics = (() => {
  let queue = [];              // private
  const flush = () => { /* ... */ queue = []; };
  return {                     // public surface
    track: (evt) => { queue.push(evt); if (queue.length > 20) flush(); },
    flush,
  };
})();
```

ES modules give you the same privacy for free — anything not `export`ed is module-private. Reach for the IIFE form only when you need runtime-constructed privacy (e.g. one instance parameterized at creation).

### Singleton

One shared instance for the whole app. Common for: HTTP clients, feature-flag clients, logger, i18n, a store.

```ts
// A module is already a singleton — the module cache guarantees one instance.
export const apiClient = createClient({ baseURL: import.meta.env.VITE_API });
```

**Trade-off / trap:** singletons are global mutable state. They are the enemy of testability (hard to reset between tests) and of SSR (state leaks across requests — never keep per-user state in a module-level singleton on the server). On the server, prefer request-scoped instances via a context/DI container.

### Factory

Centralize "which concrete thing do I build?" behind one call, so callers stay ignorant of the concrete types.

```ts
function createStorage(kind: "local" | "memory" | "session"): Storage {
  switch (kind) {
    case "local":   return new LocalStorageAdapter();
    case "session": return new SessionStorageAdapter();
    default:        return new MemoryStorage();
  }
}
```

Use when construction logic is non-trivial or varies by environment (test vs prod storage, mock vs real transport). Do **not** wrap a single `new` in a factory — that is ceremony.

---

## Structural Patterns

### Decorator

Wrap an object/function to add behavior without touching the original. In JS this is usually higher-order functions.

```ts
const withRetry = <A extends any[], R>(fn: (...a: A) => Promise<R>, tries = 3) =>
  async (...args: A): Promise<R> => {
    let lastErr: unknown;
    for (let i = 0; i < tries; i++) {
      try { return await fn(...args); } catch (e) { lastErr = e; }
    }
    throw lastErr;
  };

const fetchUser = withRetry(rawFetchUser);
```

React HOCs are the decorator pattern applied to components. Middleware chains (Redux, Express) are decorators too.

### Facade

A single simplified interface over a messy subsystem. Your `api/` layer is a facade over `fetch` + auth + retry + serialization. A custom hook like `useAuth()` is a facade over token storage, refresh timers, and context.

```ts
// Facade hides fetch, auth header injection, error normalization, JSON parsing.
export const api = {
  getOrders: () => http.get<Order[]>("/orders"),
  cancel:    (id: string) => http.post(`/orders/${id}/cancel`),
};
```

### Adapter

Make an incompatible interface fit the one you want. The key distinction from facade: **adapter conforms to a target interface you already depend on**; facade just simplifies.

```ts
// Third-party analytics has a different shape; adapt it to our AnalyticsPort.
interface AnalyticsPort { track(name: string, props?: object): void; }

class SegmentAdapter implements AnalyticsPort {
  constructor(private seg: Segment) {}
  track(name: string, props?: object) { this.seg.enqueue({ event: name, properties: props }); }
}
```

Adapters are how you keep vendor lock-in at the edges — swap SDKs without touching call sites.

### Proxy

Stand in front of an object to intercept access — lazy loading, caching, access control, reactivity. **Vue's reactivity and MobX are built on `Proxy`.**

```ts
const cached = new Proxy(expensiveService, {
  get(target, key) {
    const val = target[key as keyof typeof target];
    return typeof val === "function" ? memoize(val.bind(target)) : val;
  },
});
```

---

## Behavioral Patterns

### Observer & Pub-Sub

**Observer:** subject holds a list of observers and notifies them directly. **Pub-Sub:** a broker/event bus decouples publishers from subscribers — they do not know each other. Pub-Sub is looser coupling; observer is more direct.

```ts
type Listener<T> = (value: T) => void;
class Observable<T> {
  private listeners = new Set<Listener<T>>();
  subscribe(l: Listener<T>) { this.listeners.add(l); return () => this.listeners.delete(l); }
  next(v: T) { this.listeners.forEach((l) => l(v)); }
}
```

This is the backbone of every state store, RxJS, the DOM event system, and `useSyncExternalStore`. **Trade-off:** great for decoupling, terrible for traceability — "who fired this and why" becomes hard. Prefer explicit data flow unless you genuinely have N-to-M fan-out.

### Strategy

Encapsulate interchangeable algorithms behind one interface; pick at runtime. Kills big `switch`/`if` ladders.

```ts
const sorters: Record<string, (a: Row, b: Row) => number> = {
  price: (a, b) => a.price - b.price,
  name:  (a, b) => a.name.localeCompare(b.name),
  date:  (a, b) => +a.date - +b.date,
};
const sorted = rows.sort(sorters[selectedKey]);
```

In React, strategy often appears as **passing a function/component as a prop** (`renderItem`, `sortFn`).

### Command

Wrap an action as an object so you can queue, log, undo, or replay it. Undo/redo, optimistic mutation queues, and macro recording are command-pattern.

```ts
interface Command { do(): void; undo(): void; }
class AddText implements Command {
  constructor(private doc: Doc, private text: string) {}
  do()   { this.doc.append(this.text); }
  undo() { this.doc.truncate(this.text.length); }
}
// history stack of Commands gives you undo/redo for free
```

---

## Dependency Injection & Inversion of Control

**IoC:** high-level modules should not depend on low-level details; both depend on abstractions. **DI** is one way to achieve it — pass dependencies in rather than constructing them inside.

Frontend does DI without heavy frameworks:
- **Constructor/function args** — the simplest DI.
- **React Context as a DI container** — provide an implementation at the root, consume abstractly.
- **Props** — a component receiving `onSave` is having its persistence strategy injected.

```tsx
const RepoContext = createContext<UserRepo>(httpUserRepo);
// tests inject a fake; prod injects the real repo — call site is unchanged.
function Profile() {
  const repo = useContext(RepoContext);
  // ...
}
```

The payoff is **testability and swappability**. The anti-pattern is a full IoC container (Angular-style) in a small React app — it adds indirection most teams do not need. Match ceremony to team size and lifespan.

---

## Architectural Patterns: MVC / MVVM / Flux

| Pattern | Data flow | View↔state binding | Frontend home |
|---|---|---|---|
| **MVC** | Controller mediates Model↔View | manual | Rails-era server apps, Backbone |
| **MVVM** | ViewModel exposes bindable state | two-way binding | Angular, Vue, Knockout |
| **Flux/Redux** | strictly one-way: action → dispatcher → store → view | one-way | React ecosystem |

**Why Flux won in React:** two-way binding makes "what changed and why" ambiguous at scale. Unidirectional flow makes state changes traceable and time-travel debuggable. The cost is boilerplate — which is why Redux Toolkit, Zustand, and signals emerged to reclaim ergonomics without abandoning one-way flow.

---

## React Idioms (Patterns Reframed)

### Custom Hooks — logic reuse
The primary reuse mechanism. Replaces mixins, most HOCs, and most render props. Extract stateful logic, keep it composable.

```tsx
function useDebouncedValue<T>(value: T, ms = 300) {
  const [debounced, setDebounced] = useState(value);
  useEffect(() => {
    const id = setTimeout(() => setDebounced(value), ms);
    return () => clearTimeout(id);
  }, [value, ms]);
  return debounced;
}
```

### Compound Components
Related components share implicit state via context; the consumer composes the layout. Gives flexibility without prop explosion.

```tsx
<Tabs defaultValue="a">
  <Tabs.List>
    <Tabs.Trigger value="a">A</Tabs.Trigger>
    <Tabs.Trigger value="b">B</Tabs.Trigger>
  </Tabs.List>
  <Tabs.Panel value="a">...</Tabs.Panel>
</Tabs>
```

### Render Props & HOCs
Older reuse patterns. **Render props** pass a function child that receives state; **HOCs** wrap a component to inject props. Both mostly superseded by hooks, but you will still see them (React Router, older libs) and must recognize the "wrapper hell" and prop-collision problems that pushed the ecosystem to hooks.

### Provider Pattern
Context provider at the root supplies theme, auth, query client, i18n. This is DI + singleton in React clothing. Watch the re-render cost — split contexts by change frequency, memoize the value.

### Container / Presentational
Separate "how it works" (data, effects) from "how it looks" (pure render). Less rigid post-hooks, but the discipline still pays off for **testability and Storybook** — presentational components are trivial to snapshot and design in isolation.

### Reducer Pattern
`useReducer` for state whose transitions are the interesting part. Colocates the transition logic and makes invalid intermediate states harder.

```tsx
function reducer(s: State, a: Action): State {
  switch (a.type) {
    case "submit":  return { ...s, status: "loading" };
    case "success": return { ...s, status: "done", data: a.data };
    case "error":   return { ...s, status: "error", error: a.error };
  }
}
```

### Controlled vs Uncontrolled
Controlled: React state is the source of truth (`value` + `onChange`). Uncontrolled: the DOM owns it (`ref`/`defaultValue`). **Trade-off:** controlled gives you validation/derived UI on every keystroke at a re-render cost; uncontrolled is cheaper and simpler for fire-and-forget forms. Leads pick per-field, not per-app.

### Headless Components
Logic and accessibility without markup/styles (React Aria, TanStack Table, Downshift, Radix primitives). You get correct ARIA/keyboard behavior and full styling control. This is the strategy + facade combination — behavior is fixed, presentation is injected.

### State Machines
XState / explicit FSMs for flows with many states and illegal transitions (checkout, media player, multi-step wizard). Makes impossible states unrepresentable and gives a diagrammable spec. Over-engineering for a two-state toggle; essential for a 12-state upload flow.

---

## SOLID Applied to Frontend

- **S — Single Responsibility:** a component either fetches, or lays out, or renders a leaf — not all three. A 600-line component doing data + layout + business rules is the most common SRP violation. Split by reason-to-change.
- **O — Open/Closed:** extend via props, composition, and `children` — not by editing the component for each new case. A `Button` that grows a new boolean prop per variant is closed wrong; a `variant` union + slots is open right.
- **L — Liskov Substitution:** a wrapper/variant must honor the base contract. If your `IconButton` drops `onClick` or breaks `disabled`, substitution is broken. Spreading `...rest` onto the underlying element preserves the contract.
- **I — Interface Segregation:** do not force a component to accept a giant prop object it half-ignores. Prefer focused props; split "god props" into smaller typed groups.
- **D — Dependency Inversion:** components depend on abstractions (a `useOrders` hook, an `AnalyticsPort`), not on `fetch` or a concrete SDK. This is what makes swapping backends or mocking in tests painless.

---

## When Patterns Help vs Over-Engineering

| Signal it helps | Signal it is over-engineering |
|---|---|
| You have ≥3 concrete variants today | You are building for one hypothetical future case |
| Change vector is proven (churn in git history) | "We might need to swap X someday" with no evidence |
| Pattern removes a growing conditional | Pattern adds an indirection with one implementation |
| It improves testability at a real boundary | It exists to look sophisticated in review |

Rule of thumb: **write the duplication first, extract the pattern on the third occurrence** (rule of three). Abstractions are expensive to reverse; duplication is cheap to reverse. Prefer the reversible mistake.

---

### Interview Questions — Design Patterns

**What is the difference between the facade and adapter patterns, and where does each show up in a frontend codebase?**

> Both wrap a subsystem, but for different reasons. A facade *simplifies* — it exposes a smaller, friendlier surface over something complex (an `api` object over fetch + auth + retry). An adapter *conforms* — it makes an incompatible interface match a target interface you already depend on (wrapping a vendor analytics SDK to implement your `AnalyticsPort`). Practically: facade reduces surface area; adapter changes shape to fit a contract. Adapters are how I keep vendor code at the edges so I can swap SDKs without touching call sites.

**When would you reach for `useReducer` over `useState`, and when is that a mistake?**

> `useReducer` wins when the *transitions* are the interesting part — multiple related fields that change together, or state that must move through a defined set of statuses (idle → loading → success/error). Colocating transitions in a reducer prevents invalid intermediate combinations and makes the logic testable in isolation. It is a mistake for a single boolean or an independent value — you have added an action-type indirection for nothing. The tell is whether reading the reducer explains the feature better than reading scattered setters.

**Explain how React Context relates to dependency injection. What are the costs?**

> Context is dependency injection: you provide an implementation at the root and consumers depend on the abstraction, not the concrete thing — so tests inject a fake and prod injects the real one with no change at the call site. The costs are re-render fan-out (every consumer re-renders when the value changes, so you split contexts by change frequency and memoize the value) and implicitness (dependencies become invisible in the component signature, which hurts traceability). I use it for genuinely cross-cutting concerns — theme, auth, query client — not to avoid passing a prop one level down.

**Custom hooks largely replaced HOCs and render props. What problems did those older patterns have?**

> HOCs cause wrapper hell (deeply nested trees in DevTools), prop-name collisions (two HOCs both inject `data`), and unclear provenance of props. Render props cause callback nesting ("render-prop pyramid") and awkward composition. Hooks solved both by letting you compose stateful logic as flat function calls with no extra tree nodes and no prop namespace conflicts. I still need to recognize HOCs/render props because major libraries use them, and some cross-cutting concerns (error boundaries) still require components.

**How do you apply the Open/Closed Principle to a component library button?**

> Closed-wrong is adding a new boolean prop for every visual case — `isPrimary`, `isDanger`, `isGhost` — which forces editing and re-testing the component for each new need and allows nonsensical combinations. Open-right is a discriminated `variant` union plus composition slots (`leftIcon`, `children`, `...rest` spread to the underlying element). New variants become data or configuration, and new behaviors come from what you compose in, so the component is extended without being modified. The Liskov angle matters too: spreading `...rest` and forwarding refs keeps the button substitutable for a native `<button>`.

**When is a state machine (XState) worth the overhead, and when is it over-engineering?**

> Worth it when a flow has many states with illegal transitions and real cost to getting them wrong — checkout, a media player, a multi-step upload with ret/cancel/error. A machine makes impossible states unrepresentable, gives a diagram that doubles as spec and stakeholder communication, and centralizes transition logic. Over-engineering for a two-or-three-state toggle, where a machine adds a dependency and cognitive load for no safety gain. The heuristic: count the states and the illegal transitions — if both are small, `useState`/`useReducer` is enough.

**Your teammate wants to introduce a full IoC container into a mid-size React app. How do you respond?**

> I would push back and ask what problem it solves that Context, hooks, and plain function arguments do not already solve. React apps get DI cheaply through those mechanisms; a heavyweight container adds indirection, a learning curve, and decorator/reflection machinery that most React teams do not need. If the real pain is testability or swappable implementations, I would address it with ports-and-adapters and Context injection. I match ceremony to team size and app lifespan — a container may earn its place in a very large, long-lived app with many teams, but rarely mid-size.

**How do you decide when to extract a pattern versus tolerate duplication?**

> I follow the rule of three: write the duplication, and only extract an abstraction on the third occurrence, once the shape has stabilized and the change vector is proven — ideally visible in git churn. Premature abstraction is expensive because it is hard to reverse and it couples call sites to a guess about the future; duplication is cheap and reversible. So I bias toward the reversible mistake early, and I name the specific force (many implementations, many observers, a boundary I do not own) before committing to a pattern. If I cannot name the force, I am decorating, not designing.
