# Angular & NgRx

> This JD leads with "Angular **and** React" — treat this file as parity prep with 01-react-internals.md, not an afterthought. Emphasis on the concepts that map to (and contrast with) what you already know cold in React.

## Component & Module Architecture

- **`@Component` decorator**: template + styles + metadata, conceptually a React function component with its JSX/CSS colocated by convention instead of by file.
- **NgModules vs standalone components**: classically every component belonged to an `NgModule` (declarations/imports/exports/providers) — the DI and compilation boundary. **Standalone components** (default since Angular 17, stable since 14) drop the module requirement — a component declares its own imports directly. Know both: most enterprise Angular code you'll inherit is still module-based; new code is standalone-first, closer to how a React component just imports what it needs.
- **Templates & binding syntax**:
  - Interpolation: `{{ value }}`
  - Property binding: `[src]="imageUrl"`
  - Event binding: `(click)="onSave()"`
  - Two-way binding: `[(ngModel)]="name"` — sugar over `[value]` + `(input)`, the thing React deliberately has no equivalent for (controlled inputs are explicit in React for a reason: traceability).
- **Directives**: structural (`*ngIf`, `*ngFor`, or the newer built-in control flow `@if`/`@for`/`@switch` in Angular 17+, which compiles to more efficient instructions than the old structural-directive microsyntax) vs attribute (`[ngClass]`, `[ngStyle]`, or a custom `@Directive` that mutates host element behavior — no real React parallel since React has no host-element mutation primitive; you'd reach for a custom hook + ref instead).
- **Pipes**: template-only pure transforms — `{{ price | currency }}`, `{{ items | async }}`. Pure pipes only re-run when the reference changes (structurally identical to React memoization reasoning); mark a pipe `pure: false` only when it must react to internal mutation, and know that's a performance trap for the same reason mutating state without a new reference is a trap in React.

## Dependency Injection

The biggest structural difference from React. Angular has a **real hierarchical DI container**; React fakes DI with Context.

- `@Injectable({ providedIn: 'root' })` registers a service as an app-wide singleton, tree-shakable if unused.
- Injectors are hierarchical: root injector → module injector → component injector. A service provided at a component level gets a **new instance per component subtree** — this is how you scope state to a feature without a global store (e.g., a `WizardStateService` provided on the wizard's root component, alive only for that subtree's lifetime).
- `InjectionToken` for non-class dependencies (config objects, primitives) — the DI-container equivalent of a typed Context key.
- **Constructor injection** is how you receive dependencies — Angular resolves the token, walking up the injector tree.
- **Contrast with React**: Context + `useContext` is a *pull* model (component asks for what's above it); Angular DI is closer to a real IoC container resolving a dependency graph at instantiation. This is the concrete answer if asked "how does DI in Angular differ from what you do in React."

## Change Detection

- **Zone.js (classic default)**: monkey-patches async APIs (`setTimeout`, `Promise`, DOM events, XHR) so Angular knows *something happened* and reruns change detection for the whole component tree from the root, checking every binding top-down. This is why an Angular app can feel like it "just works" for reactivity with zero manual subscriptions — and why large trees historically needed **`ChangeDetectionStrategy.OnPush`** to opt a subtree out of the default check-everything pass.
- **`OnPush`**: a component only re-checks when (a) an `@Input` reference changes, (b) an event originates from within it, or (c) you manually call `markForCheck()`/use the `async` pipe. This is Angular's version of React's "don't re-render unless props/state actually changed" — same underlying discipline (treat data as immutable, pass new references on change), different mechanism (dirty-marking a tree vs re-running a function and diffing).
- **Angular Signals (16+, the real paradigm shift)**: fine-grained reactive primitives (`signal()`, `computed()`, `effect()`) that track *exactly* which bindings depend on them and update only those — no zone, no tree walk. This is Angular converging toward the same fine-grained-reactivity model as Solid/Vue 3, explicitly to escape Zone.js overhead. Zoneless Angular (`provideExperimentalZonelessChangeDetection` → stabilizing) is where the framework is heading. If asked "what's new in Angular," lead with signals — it's the framework's biggest recent shift and shows currency.
- **Interview framing**: change detection strategy (Zone+OnPush vs Signals) is Angular's answer to the same problem React solves with the Fiber reconciler and memoization — deciding the minimum work needed to keep the DOM in sync with state. Naming that parallel is a strong senior-level signal.

## RxJS — the Operators That Actually Come Up

RxJS is unavoidable in Angular (`HttpClient` returns Observables, forms, router events); know the flattening operators cold, they're the classic interview trap.

| Operator | Behavior on new emission while inner is active | Use case |
|---|---|---|
| `switchMap` | **Cancels** the previous inner observable, switches to the new one | Typeahead/search — only the latest request matters |
| `mergeMap` | Runs **all** inner observables concurrently, merges results | Independent parallel requests (e.g. fire N uploads) |
| `concatMap` | **Queues** — waits for the current inner to complete before starting the next | Ordered sequential requests (must-happen-in-order writes) |
| `exhaustMap` | **Ignores** new emissions while an inner is active | Prevent double-submit on a save button |

```ts
searchInput.valueChanges.pipe(
  debounceTime(300),
  distinctUntilChanged(),
  switchMap(term => this.api.search(term)),   // cancels stale in-flight search on new keystroke
  takeUntil(this.destroy$)                     // unsubscribe on component destroy — the #1 memory-leak fix
).subscribe(results => this.results = results);
```

- **`takeUntil(this.destroy$)` + `ngOnDestroy` calling `destroy$.next()`** is the classic manual-unsubscribe pattern to prevent leaks — the equivalent discipline to a React `useEffect` cleanup function. Newer code leans on the **`async` pipe** in templates instead, which subscribes/unsubscribes automatically tied to the view's lifecycle — prefer it over manual `.subscribe()` in the component class wherever the value is only needed in the template.
- `combineLatest`/`forkJoin`: combine multiple streams — `combineLatest` re-emits whenever *any* source emits (ongoing sync), `forkJoin` waits for *all* sources to complete once (like `Promise.all`).

## Forms

- **Template-driven**: `[(ngModel)]` in the template, Angular builds the form model implicitly. Fast for simple forms, harder to unit test, validation logic lives in the template.
- **Reactive forms**: `FormBuilder`/`FormGroup`/`FormControl` built explicitly in the component class, validators as pure functions (`Validators.required`, custom validator functions), form state as an observable you can react to (`form.valueChanges`, `form.statusChanges`).
- **Lead default**: reactive forms for anything beyond a trivial form — testable without rendering the DOM, composable custom validators, and the explicit model mirrors why React favors controlled inputs (traceable, testable state) over uncontrolled DOM-owned form state.

## NgRx vs Redux/RTK

Structurally the same Flux pattern you already know; the vocabulary and boilerplate differ.

| Concept | NgRx | Redux/RTK |
|---|---|---|
| State container | Store (RxJS `Observable` of state) | Store (plain JS, subscribe callback) |
| State change intent | Action (`createAction`) | Action (`createAction` in RTK too) |
| Pure state transition | Reducer (`createReducer`, `on()`) | Reducer (RTK `createSlice`) |
| Derived/memoized state | Selector (`createSelector`, memoized like `reselect`) | Selector (`reselect`, or RTK Query) |
| Side effects (API calls) | **Effects** — a separate, explicit RxJS stream that listens for actions and dispatches new ones | Thunks / RTK Query / Redux-Saga |
| Reading state in a component | `store.select(selectX)` returns an `Observable`, consumed via `async` pipe | `useSelector` returns a plain value directly |

- **Effects are the biggest conceptual difference**: in NgRx, side effects are *isolated* from reducers by design — an `Effect` is an RxJS stream (`actions$.pipe(ofType(loadUsers), switchMap(...)))` that reacts to an action and dispatches a result action. Reducers stay 100% pure. Redux gets there with middleware (thunk/saga) but it's a convention, not an enforced architectural boundary the way Effects are in NgRx.
- **Everything is push-based via Observables** — components subscribe to store slices rather than re-rendering on every store change like `useSelector` triggers a re-render. Combined with `OnPush` + `async` pipe, this is how large NgRx apps stay performant without manual memoization.
- **When NOT to reach for NgRx** (same lead judgment as "don't reach for Redux by default" in [08-frontend-architecture.md](08-frontend-architecture.md)): local component state, a single feature's UI state, or state with a short lifetime doesn't need the global store — Angular services with a `BehaviorSubject` are a lighter-weight store for feature-scoped state, same as reaching for Zustand/Context before Redux in React.

## Routing & HTTP

- **Angular Router**: route config maps paths to components, `RouterModule.forRoot`/`forChild` for lazy-loaded feature modules (`loadChildren: () => import(...)`), **route guards** (`CanActivate`, `CanDeactivate`, functional guards in modern Angular) — the direct equivalent of a React Router loader/protected-route wrapper, but built into the framework rather than composed by convention.
- **`HttpClient` + interceptors**: an `HttpInterceptor` sits in a chain around every request/response — the Angular-native place to attach an auth token, handle 401 refresh-and-retry, or normalize errors, exactly the role an Axios interceptor or a `fetch` wrapper plays in a React app. Framing an answer as "this is Angular's built-in version of what I'd hand-roll in a fetch wrapper" shows you're translating concepts, not learning from zero.

## Testing

- **Jasmine + Karma (classic)** vs **Jest** (increasingly the modern default, faster, better watch mode, same API shape you already use for React). Many enterprise Angular shops are mid-migration from Karma to Jest — worth mentioning if asked about legacy pain points.
- **`TestBed`**: Angular's DI-aware test harness — configures a mini `NgModule` for the test, resolves dependencies through the same injector mechanism as the real app, so you can inject a fake service where a component expects a real one. Conceptually parallel to wrapping a React component in test providers (`renderWithProviders` in [12-testing.md](12-testing.md)), but DI-driven rather than prop/context-driven.

## Angular vs React — the Comparison You'll Be Asked to Make

| | Angular | React |
|---|---|---|
| What it is | Full framework (router, forms, DI, HTTP all included) | UI library — you assemble router/forms/state from the ecosystem |
| Language | TypeScript-first, by design | JS by default, TypeScript by convention |
| Reactivity (classic) | Zone.js, dirty-checking a tree | Virtual DOM diff / Fiber reconciliation |
| Reactivity (modern) | Signals (fine-grained) | Signals-inspired experiments exist (e.g. in some meta-frameworks), but core React still re-renders + diffs |
| DI | First-class, hierarchical injector | None built-in — Context simulates it |
| Data binding | Two-way (`ngModel`) available | One-way only, by design — traceability over convenience |
| Learning curve | Steeper upfront (DI, RxJS, modules), more structure enforced | Lower upfront, more architectural decisions left to the team |

- **Honest talking point for a dual-stack role**: Angular's opinionated structure (DI, modules, one true router/forms/HTTP stack) is a genuine advantage for large teams and long-lived enterprise apps — less bikeshedding, more consistency across teams — which is exactly why banks/enterprises standardize on it for internal tooling. React's flexibility wins for product teams that need to move fast and pick best-of-breed tools per problem. Neither is "better" in the abstract; the fit depends on team size, app lifespan, and how much structure you want the framework to enforce vs the team to enforce by convention. This is the answer that reads as senior, not as "I have a favorite."

---

### Interview Questions — Angular & NgRx

**Walk me through Angular's change detection, and how does it compare to what React does?**

> Classic Angular uses Zone.js to patch async APIs so it knows when something might have changed, then walks the whole component tree checking bindings — `OnPush` lets a component skip that check unless its inputs change by reference or an internal event fires, which is the same "don't do work unless data actually changed" discipline React gets from immutable state plus memoization. The real shift is Angular Signals — fine-grained reactive primitives that track exactly which bindings depend on a value and update only those, no tree walk, no zone. That's converging toward the same fine-grained-reactivity model other frameworks use, and it's the direction Angular is explicitly heading with zoneless change detection. Structurally it's the same problem React's Fiber reconciler solves — minimum work to keep the DOM in sync with state — just a different mechanism.

**Explain the difference between `switchMap`, `mergeMap`, `concatMap`, and `exhaustMap` with a real use case for each.**

> They differ in how they handle a new emission while an inner observable is still active. `switchMap` cancels the in-flight inner observable and switches — the right choice for a search-as-you-type, where only the latest query result matters and I want stale requests cancelled. `mergeMap` runs all inner observables concurrently, which fits independent parallel work like firing several uploads at once. `concatMap` queues, running one at a time in order — for sequential writes where order matters and I can't let a later request finish before an earlier one. `exhaustMap` ignores new emissions while one is in flight — my default for a save button, so a double-click can't fire two submits.

**How does dependency injection in Angular differ from how you achieve similar goals in React?**

> Angular has a real hierarchical DI container — services are registered with a scope (root, module, or component-level), and Angular resolves the dependency graph through constructor injection, walking up the injector tree. A service provided at a component gets its own instance for that subtree, which is a clean way to scope state without a global store. React has no DI container; Context plus `useContext` fakes it as a pull model — a component asks for whatever's above it in the tree, but there's no formal registration, scoping, or resolution graph. For testing, that means Angular's `TestBed` can swap a real service for a fake through the same injector mechanism the app uses, versus wrapping a component in test providers in React.

**Compare NgRx effects to how you'd handle side effects in Redux.**

> NgRx enforces the separation by construction: an Effect is a distinct RxJS stream that listens for an action type and dispatches a new action as a result, and reducers stay 100% pure with no async code ever allowed in them. Redux gets to the same place with thunks, sagas, or RTK Query, but it's convention-enforced, not architecturally enforced — nothing stops a team from putting async logic somewhere it shouldn't be. Both end up push-based and observable — NgRx state is read via `store.select()` returning an Observable, consumed through the `async` pipe, which combined with `OnPush` avoids manual memoization the way `useSelector` plus `reselect` does in Redux.

**When would you recommend Angular over React for a new enterprise project, and vice versa?**

> Angular's opinionated structure — one true router, one forms system, built-in DI, TypeScript by default — pays off on large, long-lived teams where consistency across squads matters more than flexibility; less bikeshedding, a shallower onboarding path once you're past the initial DI/RxJS learning curve. React wins when a team needs to move fast, pick best-of-breed tools per concern, or the app's lifespan/team size doesn't justify the structure Angular enforces upfront. I don't treat it as one being objectively better — it's a fit question against team size, app lifespan, and how much architectural decision-making you want the framework to make for you versus leaving to convention.

**You inherit an Angular app with heavy `ngModel` two-way binding and no `OnPush` anywhere, and it's slow. What do you do?**

> First profile — Angular DevTools' change-detection profiler or the browser's Performance panel — to confirm the slowness is actually change-detection churn and find the worst offenders, not guess. Then I'd apply `OnPush` to leaf/list components first, since those are usually re-checked most often, making sure their `@Input`s are passed as new references on change (immutable update, same discipline as React). For search/typeahead- style two-way bindings, I'd move the reactive-forms + `debounceTime`/`distinctUntilChanged` route instead of raw `ngModel`, and prefer the `async` pipe over manual subscriptions so the view's own lifecycle manages unsubscription and Angular only checks bindings tied to that stream. Longer-term, if the team is on Angular 16+, migrating hot paths to Signals removes the zone-driven tree walk entirely for that reactive state.
