# Copilot Instructions

Repository-wide context. Applies to every file.

The Playwright authoring rules below are the single source for how test code is
written in this repository — they are stated here and nowhere else, because two
copies drift and the stale one is the one that gets followed. This repository
carries no path-scoped `.github/instructions/*.instructions.md` files: this file
is the whole of the repository-wide context, and the agent specs in
`.github/agents/` are the rest.

## What is in this repository

| Path | What it is |
|---|---|
| `.github/agents/` | The QA agent workflow: `qa-agent-dev.agent.md` orchestrates, `sub-*.agent.md` are its workers. These specs ARE the contract those agents run under. |
| `.github/skills/` | Skill files. The orchestrator loads `test-design` only (Rule 17); the rest are superseded by the agent specs. |
| `.github/client-config/` | Client profiles (`<client>.yaml`), `defaults.yaml`, and the `schema.json` they validate against. The active profile supplies the tracker, the environment, and the credentials contract. |
| `.github/scripts/` | Workflow helpers — evidence attachment (`qa-evidence/`), TestRail case normalization and acceptance-criterion backfill, priority mapping, application readiness preflight. |
| `.agent-workspace/<ticket>/` | Per-ticket working directory the run writes: normalized cases, results, the generated Playwright harness, and its `test-results/`. Not source. |

**This repository contains no application source.** It is the QA harness only.
The system under test runs elsewhere and is reached over HTTP at
`environment.applicationUrl` in the active client profile, which is the only
authority on it. Do not look for application source of any kind here, and never
report its absence as a finding: there is nothing to find, and reading
application source to explain a failure is forbidden anyway (`sub-create-defect`,
Step 1). A run diagnoses the application through its own evidence — screenshot,
recording, trace — not through its code.

## Editing the agent workflow

- An agent spec is a prompt, not documentation. Changing its wording changes
  behavior, so make the smallest edit that states the new rule and leave the
  surrounding rules alone.
- **A running chat session does not pick up edits to an agent spec.** The file is
  read into the prompt when the session starts and replayed from cache for the rest
  of it. After editing a spec, start a new chat — re-running the same session just
  replays the old rules.
- A rule the QA workflow must enforce belongs in the agent spec that performs the
  step. A skill file will not be read unless Rule 17 selects it.
- Never weaken an execution-honesty or evidence rule to make a run go green.

## Playwright Test Authoring Rules (MANDATORY)

You are generating and fixing Playwright tests. The single largest source of
failures in this repo is code that interacts with the page before it is ready.
Playwright already solves this. Your job is to use its built-in waiting and
never to reimplement it.

### 0. The one rule everything else follows from

**Never wait for time. Always wait for a condition.**

If your test contains a duration that you chose, it is wrong.

### 1. Hard bans — never emit these

These are forbidden in test code. If you are about to write one, stop and use
the replacement in section 2.

| Banned | Why |
|---|---|
| `await page.waitForTimeout(...)` | Fixed sleep. Passes locally, fails in CI. |
| `setTimeout` / `sleep()` / `new Promise(r => setTimeout(r, n))` | Same thing, hand-rolled. |
| `await page.waitForLoadState('networkidle')` | Discouraged by Playwright; never fires on apps with polling/websockets/analytics. |
| `page.$(...)`, `page.$$(...)`, `elementHandle`, `$eval`, `$$eval` | ElementHandles are a snapshot. They go stale on re-render and do not auto-wait. |
| `while` / `for` retry loops around a check | Reimplements retrying assertions, badly. |
| `try { await x } catch { await page.waitForTimeout(...); await x }` | Retry-by-sleep. Hides the real bug. |
| `{ force: true }` | Skips the actionability checks that are the fix for this problem. |
| `{ timeout: 0 }` or a per-call timeout added just to make a failure go away | Turns a fast failure into a hung job. |
| `await expect(await locator.isVisible()).toBe(true)` | `isVisible()` resolves immediately; awaiting the boolean kills all retrying. |
| `await expect(await locator.textContent()).toBe('X')` | Same — reads once, no retry. |

### 2. Replacements — always emit these instead

**Use web-first assertions.** Every `expect(locator)` assertion polls until it
passes or the expect timeout expires. This is the wait.

```ts
// WRONG
await page.waitForTimeout(2000);
expect(await page.locator('.total').textContent()).toBe('$42.00');

// RIGHT
await expect(page.getByTestId('total')).toHaveText('$42.00');
```

Mapping from intent to the correct API:

| You want to wait for | Use |
|---|---|
| Element to appear / render | `await expect(loc).toBeVisible()` |
| Element to disappear (spinner, toast, modal) | `await expect(loc).toBeHidden()` |
| Text / value to settle | `toHaveText`, `toContainText`, `toHaveValue` |
| List to finish loading | `await expect(rows).toHaveCount(n)` |
| Button to become usable | `await expect(btn).toBeEnabled()` |
| Navigation to complete | `await expect(page).toHaveURL(/\/orders\/\d+/)` |
| Page title | `await expect(page).toHaveTitle(...)` |
| Attribute / class / state | `toHaveAttribute`, `toHaveClass`, `toBeChecked` |
| Non-DOM condition (JS state, API, localStorage) | `await expect.poll(async () => ...).toBe(x)` |
| A block of assertions to eventually all hold | `await expect(async () => { ... }).toPass()` |
| A specific network response | start `page.waitForResponse(...)` **before** the action, then await both |

**Do not add explicit waits before actions.** `click`, `fill`, `check`,
`selectOption`, `hover`, `press` already wait for the element to be attached,
visible, stable (not animating), enabled, and able to receive events. Writing
`await page.waitForSelector(sel)` before `await page.click(sel)` is redundant —
delete it.

```ts
// WRONG
await page.waitForSelector('#submit');
await page.click('#submit');

// RIGHT
await page.getByRole('button', { name: 'Submit' }).click();
```

`waitForSelector` is only acceptable when you genuinely need to wait for
something you are not about to assert on or interact with. In practice that is
almost never. Prefer `expect(...).toBeVisible()` because its failure message
names the element and shows the actual state.

### 3. Locators, not handles

Always build a `Locator` and let it resolve lazily at the moment of use. A
locator re-queries the DOM on every retry, which is what makes auto-waiting
survive re-renders.

Selector priority, in order:
1. `page.getByRole(...)` with an accessible name
2. `getByLabel`, `getByPlaceholder`, `getByText`
3. `getByTestId` (`data-testid`)
4. CSS — last resort, never structural (`div > div:nth-child(3)` is banned)
5. XPath — never

Never chain off a locator you captured before an action that re-renders the
region; re-declare it or scope it: `const row = table.getByRole('row', { name: 'ACME' })`.

Respect strict mode. If a locator matches multiple elements, **fix the locator**
— scope it with `.filter({ hasText })` or a parent. Do not paper over it with
`.first()` unless "the first one" is genuinely the assertion.

### 4. The race you must get right

Actions that trigger a request/navigation race the wait. Register the wait
before the action, then await together.

```ts
// WRONG — response may land before the wait is registered
await page.getByRole('button', { name: 'Save' }).click();
await page.waitForResponse('**/api/save');

// RIGHT
const saved = page.waitForResponse(r => r.url().includes('/api/save') && r.ok());
await page.getByRole('button', { name: 'Save' }).click();
await saved;
```

Better still, when a UI signal exists, assert on the UI instead of the network —
it tests what the user sees:

```ts
await page.getByRole('button', { name: 'Save' }).click();
await expect(page.getByRole('status')).toHaveText('Saved');
```

For a spinner-driven load, wait for the spinner to go **and** the content to
arrive — never just one:

```ts
await expect(page.getByTestId('spinner')).toBeHidden();
await expect(page.getByRole('row')).toHaveCount(25);
```

### 5. No conditional flakiness

`isVisible()`, `count()`, `textContent()`, `isEnabled()` return the state *right
now* and never retry. Never branch on them to decide what the test does:

```ts
// WRONG — races the render; silently skips the assertion
if (await page.locator('.banner').isVisible()) {
  await page.locator('.banner .close').click();
}
```

If the element is always expected, assert it. If it is genuinely optional, make
that explicit and bounded, and say so in a comment:

```ts
const banner = page.getByRole('alert', { name: 'Cookie notice' });
await banner.waitFor({ state: 'visible', timeout: 5_000 }).catch(() => {});
if (await banner.isVisible()) await banner.getByRole('button', { name: 'Close' }).click();
```

This pattern is allowed **only** for third-party/optional UI, never for the
app's own state.

### 6. Timeouts belong in config, not in test bodies

Set them once:

```ts
// playwright.config.ts
export default defineConfig({
  timeout: 30_000,                 // per test
  expect: { timeout: 10_000 },     // per web-first assertion
  use: {
    actionTimeout: 10_000,
    navigationTimeout: 30_000,
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
    video: 'retain-on-failure',
  },
  webServer: {                     // this is how you wait for the app to boot
    command: 'npm run start',
    url: 'http://localhost:3000',
    reuseExistingServer: !process.env.CI,
    timeout: 120_000,
  },
  retries: process.env.CI ? 2 : 0,
});
```

A per-call timeout is acceptable only when that specific step is legitimately
slower than the default (a report export, a large upload). When you use one, add
a comment saying why. Never raise a timeout to fix a failure you have not
diagnosed.

Use the `webServer` config to wait for the app — never a sleep at the top of a
test, and never a bare `page.goto` against a server that may not be up.

### 7. Navigation

- `page.goto()` already waits for `load`. Don't follow it with a wait.
- Never use `waitForNavigation` (deprecated). Assert on the destination:
  `await expect(page).toHaveURL(...)` plus a visible element from the new page.
- After a redirect-heavy login, wait for the first element that only exists when
  authenticated, not for the URL alone.
- Prefer `storageState` / a setup project for auth over logging in via the UI in
  every test — fewer steps, fewer races.

### 8. Visual and animation stability

- Assertions already wait for the element to stop moving. Don't sleep for CSS
  transitions.
- For screenshots use `await expect(page).toHaveScreenshot()` — it retries until
  two consecutive frames match. Never `page.screenshot()` + manual compare.
- Disable animations where they are pure noise: `toHaveScreenshot({ animations: 'disabled' })`.

### 9. When a test fails with a timeout — triage, do not sedate

Forbidden responses to a timeout: adding a sleep, raising the timeout, adding
`force: true`, adding a retry loop, changing the selector to something broader
until it matches.

Required procedure:
1. Read the error. Playwright states whether the element was **not found**,
   found but **not visible**, **not stable**, **not enabled**, or **intercepted
   by another element**. Each has a different fix.
2. Open the trace (`trace: 'on-first-retry'`, `npx playwright show-trace`) and
   look at the DOM snapshot at the moment of failure.
3. Classify:
   - *Not found* → wrong locator, or the app genuinely never rendered it (a real bug — report it, don't hide it).
   - *Not visible* → you are waiting on the wrong element; wait for the one that actually signals readiness.
   - *Intercepted* → an overlay/spinner/toast is on top. Wait for it to be hidden; do not use `force`.
   - *Not stable* → animation in progress; the assertion will handle it, so the real problem is elsewhere.
   - *Not enabled* → wait for `toBeEnabled()`, and check whether the app's enable condition is what you think.
4. Fix the cause. If the cause is a product bug or a genuinely missing loading
   state, say so explicitly in your response instead of making the test pass.

### 10. Test structure

- One `test.step()` per logical phase; it makes traces readable.
- Every test independent and runnable in parallel — no shared mutable state, no
  ordering assumptions, no reliance on data left behind by another test.
- Set up state via API/fixtures, not by clicking through the UI.
- Prefer `beforeEach` fixtures over copy-pasted setup.
- Use unique data per test run (timestamps/uuids) so parallel workers don't collide.
- Assert something meaningful. A test that only navigates and never asserts is
  not a test.

### 11. Self-check before you return any test code

Answer all of these. If any answer is wrong, rewrite before returning.

- [ ] Zero occurrences of `waitForTimeout`, `sleep`, `setTimeout`, `networkidle`.
- [ ] Zero `page.$`, `$$`, `$eval`, `elementHandle`, `force: true`.
- [ ] Every wait is a web-first assertion, `expect.poll`, `toPass`, or a
      `waitFor*` registered **before** the triggering action.
- [ ] No value is read with `textContent()` / `isVisible()` / `count()` and then
      asserted — the assertion is on the locator itself.
- [ ] Every `expect` on a locator is `await`ed. (A missing `await` is a silently
      passing test.)
- [ ] Locators use role/label/testid, and each resolves to exactly one element.
- [ ] Any per-call timeout has a comment justifying it.
- [ ] After each action that changes the page, there is an assertion proving the
      new state before the next action.
- [ ] Tests are independent and parallel-safe.

### 12. Reference snippet — the shape every test should have

```ts
import { test, expect } from '@playwright/test';

test('user can submit an order', async ({ page }) => {
  await test.step('open the order form', async () => {
    await page.goto('/orders/new');
    await expect(page.getByRole('heading', { name: 'New order' })).toBeVisible();
  });

  await test.step('fill and submit', async () => {
    await page.getByLabel('Customer').fill('ACME Corp');
    await page.getByLabel('Quantity').fill('3');

    const created = page.waitForResponse(
      r => r.url().includes('/api/orders') && r.request().method() === 'POST' && r.ok()
    );
    await page.getByRole('button', { name: 'Create order' }).click();
    await created;
  });

  await test.step('confirm the result', async () => {
    await expect(page).toHaveURL(/\/orders\/\d+$/);
    await expect(page.getByTestId('order-status')).toHaveText('Pending');
    await expect(page.getByTestId('spinner')).toBeHidden();
  });
});
```
