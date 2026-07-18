# Frontend Testing Strategy (Lead-Level)

> How a lead decides what to test, at which layer, and how to keep the suite fast, trustworthy, and enforced in CI — not just how to write a test.

## Trophy vs Pyramid

The classic **pyramid** (lots of unit, some integration, few E2E) was written for backends where units are pure and integration is expensive. On the frontend, the **testing trophy** (Kent C. Dodds) fits better: the bulk of value sits in **integration/component tests** that render real components with real user interactions, because that's where FE bugs actually live — wiring, props, state, and DOM.

```
        /\        E2E            few, high-confidence, slow
       /--\       Integration    <-- the fat middle (trophy)
      /----\      Unit
     /------\     Static (TS + ESLint)  <-- widest, cheapest
```

- **Static** (TypeScript, ESLint, typed props) catches a whole class of bugs for free — treat it as the base layer, not an afterthought.
- **Weight the middle.** A component test that renders a form, types into it, submits, and asserts the result catches integration bugs a hundred isolated unit tests miss.
- **Return on investment is the deciding axis:** confidence gained per millisecond of runtime and per minute of maintenance. E2E gives the most confidence but the worst ROI per test, so keep it thin and reserved for critical journeys.

| Layer | Speed | Confidence | Weight | Tool |
|---|---|---|---|---|
| Static | instant | low-med | max | TypeScript, ESLint |
| Unit | ms | low | some | Vitest / Jest |
| Component/Integration | 10s of ms | **high** | **most** | RTL + Vitest + MSW |
| E2E | seconds | highest | few | Playwright / Cypress |

## Unit Testing (Vitest / Jest)

For pure logic — reducers, selectors, formatters, utility functions, custom hook math. Fast, deterministic, no DOM needed.

- **Vitest** is the default for Vite projects: native ESM, same config as the app, instant HMR-style watch, Jest-compatible API. **Jest** remains fine for CRA/Webpack/Next legacy stacks.
- Keep units genuinely pure; if a "unit" needs heavy mocking to exist, it's really an integration test wearing a disguise — move it up a layer.

```ts
import { describe, it, expect } from 'vitest';
import { formatCurrency } from './format';

describe('formatCurrency', () => {
  it('renders USD with two decimals', () => {
    expect(formatCurrency(1234.5, 'USD')).toBe('$1,234.50');
  });
  it('handles zero and negatives', () => {
    expect(formatCurrency(0, 'USD')).toBe('$0.00');
    expect(formatCurrency(-5, 'USD')).toBe('-$5.00');
  });
});
```

## Component Testing with React Testing Library

The guiding principle: **"The more your tests resemble the way your software is used, the more confidence they give you."** Test behavior, not implementation.

### Query by role / accessible name (ties to a11y)

Prefer queries in this priority order — they double as an accessibility audit, because if a screen reader can't find the element, neither can your test:

1. `getByRole('button', { name: /submit/i })` — role + accessible name (best)
2. `getByLabelText` / `getByPlaceholderText` — form fields
3. `getByText` — non-interactive content
4. `getByTestId` — last resort escape hatch only

```tsx
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { LoginForm } from './LoginForm';

it('submits credentials the user typed', async () => {
  const onSubmit = vi.fn();
  const user = userEvent.setup();
  render(<LoginForm onSubmit={onSubmit} />);

  await user.type(screen.getByLabelText(/email/i), 'a@b.com');
  await user.type(screen.getByLabelText(/password/i), 'hunter2');
  await user.click(screen.getByRole('button', { name: /log in/i }));

  expect(onSubmit).toHaveBeenCalledWith({ email: 'a@b.com', password: 'hunter2' });
});
```

### Behavior, not implementation

- **Use `@testing-library/user-event`, not `fireEvent`.** `user-event` dispatches the full sequence of real events (focus, keydown, input, change) so it exercises what a user actually triggers.
- **Never assert on state, instance methods, private variables, or CSS classes.** Assert on what the user sees: text, roles, enabled/disabled, presence/absence.
- **Don't test `useState`/`useEffect` directly** — test the observable outcome they produce. If you refactor `useState` to `useReducer`, a behavior test shouldn't break. That's the signal you're testing the right thing.
- Avoid `container.querySelector` and snapshotting the whole DOM — both couple tests to structure.

## Network Mocking with MSW

**Mock Service Worker** intercepts requests at the network layer (Service Worker in the browser, request interceptor in Node), so your code makes real `fetch`/`axios` calls and gets mocked *responses*.

```ts
import { http, HttpResponse } from 'msw';
import { setupServer } from 'msw/node';

export const server = setupServer(
  http.get('/api/user/:id', ({ params }) =>
    HttpResponse.json({ id: params.id, name: 'Ada' })
  )
);
// vitest.setup.ts
beforeAll(() => server.listen({ onUnhandledRequest: 'error' }));
afterEach(() => server.resetHandlers());
afterAll(() => server.close());
```

**Why MSW over `jest.mock('axios')`:**

- You mock the *contract* (URL, method, response shape), not the *client library*, so tests survive swapping `fetch` for `axios` or React Query.
- **The same handlers work in unit tests, Storybook, and the running dev app** — one source of mock truth.
- No brittle `mockResolvedValueOnce` chains that leak between tests and hide real request/serialization bugs.
- `onUnhandledRequest: 'error'` catches requests you forgot to mock — a real network call in a test is a failure, not a silent pass.

## Integration, E2E, and Beyond

### Integration

Multiple components + real store/router/providers + MSW. This is the trophy's sweet spot: render a page, click through a flow, assert the outcome — no browser needed (jsdom/happy-dom).

### E2E — Playwright vs Cypress

| | Playwright | Cypress |
|---|---|---|
| Architecture | Out-of-process driver (CDP/WebDriver) | Runs in the browser event loop |
| Browsers | Chromium, Firefox, WebKit | Chromium family + Firefox (WebKit experimental) |
| Parallelism | Built-in, free, workers | Paid Dashboard for orchestration |
| Multi-tab / multi-origin | Native | Limited/awkward |
| Auto-wait | Yes (web-first assertions) | Yes |
| Debugging DX | Trace viewer, codegen | Excellent time-travel UI |
| Language | TS/JS/Python/.NET/Java | JS/TS only |

- **Lead call today:** Playwright for most new projects — true cross-browser (WebKit/Safari coverage matters), free parallelism, better CI story, first-class API/network control. Cypress still wins on interactive debugging DX and a gentler learning curve for teams new to E2E.
- Keep E2E to **critical revenue/auth journeys** (login, checkout, core CRUD). They're the slowest and flakiest; every one you add is a maintenance liability.
- Seed state via API/DB, not by clicking through setup UI — faster and less flaky.

### BDD with Cucumber / Gherkin

Gherkin specs (`Feature` / `Scenario` / `Given-When-Then`) are meant to be readable and, ideally, writable by non-engineers — PM, BA, QA — as executable acceptance criteria.

```gherkin
Feature: Login
  Scenario: Valid credentials log the user in
    Given I am on the login page
    When I enter valid credentials
    And I submit the form
    Then I should see the dashboard
```

```ts
// step definitions — the glue code mapping Gherkin lines to real test actions
Given('I am on the login page', () => { cy.visit('/login'); });
When('I enter valid credentials', () => {
  cy.get('[name=email]').type('a@b.com');
  cy.get('[name=password]').type('hunter2');
});
Then('I should see the dashboard', () => { cy.url().should('include', '/dashboard'); });
```

- **Tooling**: `@badeball/cypress-cucumber-preprocessor` wires Gherkin into Cypress; `playwright-bdd` does the same for Playwright — both compile feature files into runnable specs against the same underlying runner covered above.
- **Trade-off, stated plainly**: BDD-the-practice (writing acceptance criteria as executable specs before coding, collaboratively) is genuinely valuable for cross-functional alignment. Cucumber-the-tool is a real cost: step definitions are an indirection layer that duplicates logic already living in page objects/helpers, feature files drift from their step definitions as the code evolves, and a purely-engineering team ends up writing "Given/When/Then" ceremony that only engineers ever read. **Lead call**: adopt it when there's a genuine non-technical audience authoring or reviewing scenarios; skip it and write plain Playwright/Cypress specs when the team is engineers-only — direct specs are faster to write, refactor, and debug.

### Visual regression

Screenshot diffing (Playwright snapshots, Chromatic, Percy, Loki) catches CSS/layout regressions that DOM assertions can't. Run against a component library / Storybook. Watch for flakiness from fonts, animations, and antialiasing — freeze time, disable animations, pin viewport.

### Contract testing

Consumer-driven contracts (Pact) verify the FE's assumptions about the API match what the backend actually ships — catches breaking API changes *before* integration, decoupling FE and BE release cycles. Alternative lightweight approach: validate MSW handlers against the real OpenAPI/GraphQL schema so mocks can't drift.

### Accessibility testing

- **`jest-axe` / `@axe-core/playwright`** run the axe engine against rendered output and fail on WCAG violations.
- Bake it into CI as a gate on key pages/components.
- Automated axe catches ~30–40% of a11y issues (contrast, missing labels, ARIA misuse) — the rest needs manual keyboard + screen-reader testing. Don't oversell coverage.

```ts
import { axe, toHaveNoViolations } from 'jest-axe';
expect.extend(toHaveNoViolations);

it('has no a11y violations', async () => {
  const { container } = render(<Card />);
  expect(await axe(container)).toHaveNoViolations();
});
```

## Coverage Philosophy

- **Coverage is a signal, not a target.** 100% coverage with weak assertions is theater; it proves lines *executed*, not that behavior is *correct*. Goodhart's law applies: the moment coverage is a KPI, people write tests that pad the number.
- Set a **floor** (e.g. 70–80%) to catch untested modules, not a ceiling to chase. Track *trend* and *critical-path* coverage, not the global percentage.
- **Mutation testing** (Stryker) is the real coverage-quality check: it mutates your source (flip `>` to `>=`, remove a line) and fails if no test catches it. High mutation score means assertions actually assert. It's expensive — run it on critical modules, not the whole repo, and often nightly rather than per-PR.

## Test Hygiene a Lead Owns

### Flaky test management

- **Quarantine, don't ignore.** Tag flaky tests, move them out of the blocking gate, file a ticket, fix or delete within an SLA. A permanently-`skip`ped test is dead weight.
- **Retries mask flake — cap them.** Playwright/Vitest retries in CI (e.g. `retries: 2`) reduce noise, but rising retry counts are a metric to watch, not a fix.
- **Root causes:** real timers/`setTimeout`, unmocked network, shared mutable state between tests, animation/transition timing, test-order dependence, race conditions in `waitFor`. Enforce test isolation (`resetHandlers`, clear mocks, fresh render per test).

### Test data / factories

Use factories (`@faker-js/faker` + a builder, or Fishery) over inline literals so tests state only what they care about:

```ts
const buildUser = (overrides: Partial<User> = {}): User => ({
  id: faker.string.uuid(),
  name: faker.person.fullName(),
  role: 'member',
  ...overrides,
});
// test only pins the field under test:
const admin = buildUser({ role: 'admin' });
```

### Snapshot testing pitfalls

- Huge auto-generated snapshots get **blind-approved** on `--updateSnapshot` — they detect *change*, not *correctness*, and reviewers rubber-stamp the diff.
- Prefer small, **inline snapshots** for stable serialized output (a formatted string, a reducer result), never a whole rendered component tree.
- Delete snapshots that break on every trivial refactor — they cost more than they catch.

### Async, timers, context/providers

```ts
// Async: assert on the eventual state with findBy* / waitFor, never arbitrary sleeps
expect(await screen.findByText('Saved')).toBeInTheDocument();

// Timers: use fake timers for debounce/throttle logic
vi.useFakeTimers();
// ...trigger debounced action...
vi.advanceTimersByTime(300);
vi.useRealTimers();

// Providers: wrap once via a custom render
function renderWithProviders(ui: React.ReactElement) {
  return render(
    <QueryClientProvider client={new QueryClient()}>
      <ThemeProvider>{ui}</ThemeProvider>
    </QueryClientProvider>
  );
}
```

- **Testing hooks:** use `renderHook` from RTL for reusable hooks, but prefer testing the hook *through a component* when it's tightly coupled to UI.
- Wrap `user-event` with fake timers carefully — pass `{ advanceTimers: vi.advanceTimersByTime }` so events don't hang.

### Test IDs vs accessible queries

- Default to **accessible queries** (role/label) — they enforce a11y and resist refactors.
- Use `data-testid` only when there's genuinely no accessible handle (a chart canvas, a purely decorative container). Treat a rising `getByTestId` count as tech-debt signal that components lack semantic markup.

## CI Integration & Team Strategy

What a lead actually enforces:

- **Gates on PR:** type-check + lint + unit/component + a11y must pass to merge. E2E smoke on merge to main; full E2E on a schedule or pre-release.
- **Speed = adoption.** A slow suite gets skipped and disabled. Enforce **sharding/parallelism** (`--shard=1/4`, Playwright workers, matrix jobs) and run only affected tests on PRs (Nx/Turborepo affected, Vitest `--changed`).
- **Determinism is non-negotiable:** no live network (MSW everywhere), fixed clock/timezone/locale, seeded random. A test that fails once per 50 runs erodes trust in the whole suite.
- **Fail the build on coverage *regression*, not on an absolute number.** Report coverage/mutation trends as PR comments for visibility.
- **Ownership:** CODEOWNERS on critical test suites; flaky-test dashboards; a "you broke it, you fix it or quarantine it same-day" rule so red main never becomes normalized.
- **Strategy over volume:** the lead's job is to decide *what confidence each layer must provide* and prune tests that don't earn their runtime — a 10-minute reliable suite beats a 40-minute flaky one every time.

### Interview Questions — Testing

**How do you decide what to test at each layer, and how do you weight the suite?**

> I use the testing-trophy model rather than the classic pyramid because frontend bugs cluster in the wiring — props, state, DOM, and network integration — not in isolated pure functions. So static analysis (TypeScript + ESLint) is the free base layer, and the bulk of my investment goes into component/integration tests that render real components, drive them with `user-event`, and mock the network with MSW, since those give the best confidence per millisecond. Unit tests are reserved for genuinely pure logic like reducers and formatters. E2E stays thin — only critical journeys like login and checkout — because it's the slowest and flakiest layer with the worst ROI per test. The deciding axis is always confidence gained versus runtime and maintenance cost.

**What does "test behavior, not implementation" mean in practice, and why does it matter for a lead?**

> It means asserting on what a user observes — visible text, roles, whether a button is enabled — rather than on internal state, hooks, private methods, or CSS classes. Concretely I query by accessible role and name, drive interactions with `user-event`, and never touch `useState` or `container.querySelector`. It matters at the team level because implementation-coupled tests break on every refactor without catching real regressions, so the team learns to distrust and disable them. Behavior tests survive refactors — swapping `useState` for `useReducer` shouldn't break a single test — and as a bonus, role/label queries double as an accessibility audit, so the test suite pushes the team toward accessible markup for free.

**Why MSW instead of mocking the HTTP client directly?**

> Mocking the client (`jest.mock('axios')`) couples tests to the library and to brittle per-call `mockResolvedValueOnce` chains that leak state and hide serialization bugs. MSW intercepts at the network layer, so my code makes real requests and I mock the *contract* — URL, method, response shape. That means tests survive swapping fetch for axios or React Query, the exact same handlers power Storybook and the dev app so there's one source of mock truth, and with `onUnhandledRequest: 'error'` any request I forgot to mock fails loudly instead of silently hitting the real network. It tests closer to production reality while being less brittle.

**Coverage is at 60% and leadership wants 100%. What's your response?**

> I'd push back on the number as a target. Coverage measures which lines executed, not whether behavior is correct — you can hit 100% with assertion-free tests, and the moment coverage becomes a KPI, Goodhart's law kicks in and people pad it. I'd set a floor around 70–80% to flag entirely untested modules, but focus energy on critical-path coverage and, for the modules that matter, mutation testing with Stryker, which actually proves the assertions catch injected bugs. I'd gate CI on coverage *regression* rather than an absolute ceiling and report trends as PR comments, so we invest in tests that earn their runtime instead of chasing a vanity metric.

**A test fails intermittently in CI. How do you handle it as the person owning test strategy?**

> First, flaky tests are worse than no test because they erode trust in the whole suite, so I treat them as incidents. Short term I quarantine the test — tag it, pull it out of the blocking gate, and file a ticket with an SLA — rather than leaving a permanent skip. Then I root-cause: the usual culprits are real timers instead of fake ones, unmocked network calls, shared mutable state between tests, animations, or a race in a `waitFor`. The fix is enforcing isolation — fresh render per test, `resetHandlers`, cleared mocks, a fixed clock and locale, and no live network via MSW. I allow a small capped retry count in CI to cut noise but I treat a rising retry rate as a metric to investigate, not a solution, and I keep a flaky-test dashboard so we fix root causes instead of normalizing a red main.

**When would you introduce Cucumber/Gherkin into an E2E suite, and what's the real cost?**

> I separate BDD-the-practice from Cucumber-the-tool. Writing acceptance criteria collaboratively as Given/When/Then before coding is valuable whenever there's a genuine non-technical audience — PM, BA, QA — authoring or reviewing scenarios; it becomes shared, living documentation. But wiring that into Cucumber adds a real indirection layer: step definitions duplicate logic that already lives in page objects, feature files drift from their glue code as specs evolve, and an engineering-only team ends up maintaining ceremony that only engineers read anyway. So my call is: adopt Cucumber when cross-functional authorship is real, and write plain Playwright/Cypress specs directly when it's not — the direct specs are faster to write, refactor, and debug, and I'd rather not pay the indirection tax for an audience that doesn't exist.
