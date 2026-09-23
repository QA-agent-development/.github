---
name: Test Automation Instructions
description: "Use when creating, editing, or reviewing Playwright + TypeScript test files and Page Objects. Enforces Page Object Model (POM), resilient locator hierarchy, auto-retrying assertions, execution honesty, and surgical maintenance."
applyTo: "tests/**/*.spec.ts,**/.agent-workspace/**/playwright/**/*.spec.ts"
---

# Instructions: Test Automation

These instructions govern all Playwright + TypeScript automated tests and Page Objects in this repository and workspace.

## 1. Source of Truth

- Treat the test cases created in TestRail (`TESTRAIL-CASES-{KEY}.json`), the approved test case proposal (`QA-TEST-CASES-{KEY}.md`), and `AUTHORITATIVE-AC` as the authoritative requirement evidence for tests being generated, executed, or updated.
- The test execution agent gets and executes against the test cases created in TestRail. Each automated test corresponds directly to a TestRail test case with its TestRail case ID (`[C{id}]`) embedded in the title.
- Never edit the source requirements, Jira acceptance criteria, or approved test case definitions from this workflow. Report any gaps or ambiguities back to the orchestrator.
- Search existing test code and page objects before authoring new ones to reuse configuration, fixtures, helpers, and page methods.
- Never turn a missing selector, route, or data value into an assumption. Extract selectors from real application markup or flag missing items.

## 2. Page Object Model (POM) Architecture

- Group user interactions, element locators, and page-specific actions into cohesive Page Object classes under `page-objects/`.
- Keep spec files (`tests/*.spec.ts`) declarative: test cases instantiate page objects, invoke domain actions, and assert observable outcomes.
- **Method Reuse**: Always inspect existing page objects before creating new methods. Never duplicate existing interaction methods.
- Ensure page object methods return locators, promises, or page instances cleanly without embedding hard-coded credentials or brittle selectors.

## 3. Locators, Assertions, and Test Data

- **Playwright mechanics are governed by the "Playwright Test Authoring Rules" section of `.github/copilot-instructions.md`**, which applies repository-wide: locator priority and strict mode, web-first auto-retrying assertions, the bans on fixed waits / `networkidle` / ElementHandles / `force: true`, action-before-wait ordering, config-level timeouts, and the triage procedure for a timeout. Those rules are deliberately not repeated here — a second copy drifts, and a drifted copy is worse than no copy. Read that file.
- **Explicit Test Data**: Use deterministic synthetic data provided via `TEST-DATA` or fixtures. Never use real customer records, real credentials, or PII.

## 4. Execution and Reporting Honesty

- Run every generated or affected test through the Playwright CLI before reporting a result. Never claim a test passed without an actual CLI run in the current session.
- Never disable (`test.skip`), mark as fixme (`test.fixme`), soft-assert, or comment out assertions to force a passing result.
- For every failure, consult the captured Playwright trace (`trace.zip`), JSON report, and failure console diffs before diagnosing root causes.
- Report the first actionable failure reason (exact assertion diff, timeout, missing element) and provide copy-pasteable Playwright CLI commands (`npx playwright show-trace <trace.zip>`, `npx playwright show-report`, `npx playwright test --debug`).

## 5. Maintenance Scope & Surgical Updates

- When the application changes (renamed selectors, route updates, UI flows):
  - Identify and update **only** the affected page-object methods and test specs.
  - Strictly **never rewrite or reformat unrelated, currently passing tests**.
  - Re-run only the affected tests after maintenance edits and report the updated result.
- If a scenario's underlying application UI element, route, or flow was removed and is no longer testable:
  - Do NOT invent artificial workarounds or guess selectors.
  - Mark the scenario as **Blocked** with the concrete application change reason.

## 6. Output Discipline

- When in-repository testing is authorized (`TARGET-LOCATION=REPO`), write specs to `tests/` and page objects to `page-objects/`.
- When isolated workspace testing is active (`TARGET-LOCATION=WORKSPACE`), write specs to `.agent-workspace/{ticket-lower}/playwright/tests/` and page objects to `.agent-workspace/{ticket-lower}/playwright/page-objects/`.
- Never modify application production source code or deployment manifests.
