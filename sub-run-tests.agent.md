---
name: sub-run-tests
description: Run the pre-written TDD test suite against completed production code, fix failures, and confirm a green build
model: Bedrock-Kimi-dev (litellm)
tools:
  - read/readFile
  - edit
  - search/codebase
  - search/textSearch
  - search/fileSearch
  - terminal
  - agent/runSubagent
user-invocable: false
argument-hint: "<CODE-CHANGES> <TEST-FILES>"
---

# Sub-Agent: Run Tests

Single responsibility: **build the project**, then run the test suite written by `sub-write-tests` against the production code produced by `sub-write-code`, fix any build errors or test failures, and confirm a green build.

## Inputs Expected

The calling agent must provide:
1. `CODE-CHANGES` — structured output from `sub-write-code` (list of files created/modified)
2. `TEST-FILES` — structured output from `sub-write-tests` (list of test files and what they cover)

## Workflow

### Step 0: Load Conventions & Detect Project Type (HARD RULE — do this first)

A green `dotnet test` is not enough — **the project MUST compile first**. Before running anything, determine what you are building and load the matching skill so you fix bugs the correct way.

1. **Locate the solution/project.** Find the `*.sln` (solution root) and the main app `*.csproj` referenced by it via `search/fileSearch`.
2. **Detect the project type** from the main `.csproj` and `CODEBASE-SUMMARY`:
   | Signal in `.csproj` / code | Project type | Skill to read |
   |----------------------------|--------------|---------------|
   | `<TargetFramework>net*-android/-ios/-maccatalyst`, `UseMaui`, `Syncfusion.Maui.*`, `MauiProgram.cs` | **.NET MAUI** | `.github/skills/peoplewith-coding-standards/SKILL.md` |
   | `Microsoft.NET.Sdk.Web`, `AddControllersWithViews`, `Controller` base classes, `.cshtml` views, `DbContext`, `*.Mvc.csproj` | **ASP.NET Core MVC** | `.github/skills/dotnet-mvc-coding-standards/SKILL.md` |
3. **Read the matching skill with `readFile`** (and `.github/skills/token-efficient-workflow/SKILL.md`) once, before building. Use the skill's build/test conventions to guide every fix — do NOT apply MAUI rules to MVC code or vice versa.
4. Record the detected project type, the main `.csproj` path, and the test `.csproj` path for use in later steps and the summary.
5. **Graphify-First Read Gate (MANDATORY before any raw `read_file` of source files to fix a build/test bug).** The STRICT FILE READING PROTOCOL in `.github/copilot-instructions.md` applies to this worker. Before reading a `.cs`/`.cshtml` file to fix an error: (a) run `graphify query "<failing symbol or file>" --budget 10000` from `{WORKSPACE_ROOT}` and use the returned `source_file`/line locations to target the read; OR (b) if the query layer is unavailable (no LLM key), `grep`/`Select-String` `graphify-out/graph.json` for the failing symbol and read the matching node's `source_file` verbatim — workspaces may nest the project under an outer git-root folder, so never reconstruct a path from the repo name; trust the `source_file` value. Read at most 3 distinct source files per turn, each with a targeted `startLine`/`endLine` range. If `graphify-out/graph.json` is missing/empty, STOP and return `ERROR: verified Graphify graph input is missing or invalid`. Do not install, rebuild, or update Graphify.

### Step 1: Build the Project & Fix Compilation Bugs (HARD RULE)

Build the main project **before** touching tests. A build that does not succeed is a failed run until you fix it.

1. **Build the targeted project** (never the full MAUI solution — that produces 5000+ warnings; per `token-efficient-workflow`):
   ```
   dotnet build <MainProject>.csproj --no-restore
   ```
   For MVC, this is the web `.csproj`; for MAUI, build the single app `.csproj` (add `-f <tfm>` if multi-targeted, e.g. `-f net9.0-android`).
2. **Truncate output to errors only** — pipe through `Select-String -Pattern 'error|Error'` or `Select-Object -Last 20`. Never capture the full warning list.
3. **Fix every compilation error** in the production/test files from `CODE-CHANGES` / `TEST-FILES`, following the skill loaded in Step 0. Common causes: wrong namespace, missing `using`, renamed symbol, type mismatch, missing project reference on the test project.
4. **Re-run the build after fixing the root cause** — do not re-run the same failing build without a change (per `token-efficient-workflow` rule 5). Repeat until the build succeeds.
5. Also build the test project (or rely on `dotnet test` in Step 3, which builds it implicitly). If the test project cannot reference the main project because of an incompatible MAUI TFM, note it in FLAGGED ISSUES.

Only proceed to Step 2 once the project **compiles cleanly**.

### Step 2: Confirm Tests Now Compile

Verify that the production classes referenced in the test files now exist (they are listed in `CODE-CHANGES`). If any referenced class is missing, stop and return an error listing the missing items — do not attempt to run.

### Step 3: Run the Full Test Suite

```
dotnet test --logger "console;verbosity=normal"
```

Capture all output, filtered to errors/failures only.

### Step 4: Triage Failures

For each failing test, classify the failure:

| Failure type | Action |
|--------------|--------|
| Production code does not satisfy the AC | Fix the production code in the relevant file from `CODE-CHANGES` |
| Test assertion is incorrect (wrong expected value, wrong mock setup) | Fix the test |
| Test references a class/method that was renamed during implementation | Update the test to match the actual name |
| Unrelated pre-existing failure | Note it in FLAGGED ISSUES, do not fix |

### Step 5: Fix and Re-run

Apply fixes and re-run:
```
dotnet test --logger "console;verbosity=normal"
```

Repeat until all tests introduced by this ticket pass. Do not attempt to fix pre-existing failures unrelated to this ticket.

### Step 6: Return Summary

```
TEST RUN RESULTS
================
TICKET: {KEY}
PROJECT TYPE: .NET MAUI | ASP.NET Core MVC
SKILL APPLIED: {skill file used to guide fixes}

BUILD: SUCCEEDED | FAILED
OUTCOME: PASSED | FAILED

TESTS RUN: {total}
PASSED: {count}
FAILED: {count}
SKIPPED: {count}

FAILURES FIXED:
  - {test method}: {root cause} -> {fix applied}

FLAGGED ISSUES (not fixed):
  - {pre-existing failure or out-of-scope issue}

PRODUCTION FILES MODIFIED DURING FIX:
  {list, or "None" -- any production code changes must be minimal and directly caused by a failing test}

FINAL STATE: All ticket tests GREEN | Residual failures exist (see flagged issues)
```

## Safety Constraints

Do NOT:
- **Skip the build.** Never run tests before the project compiles. A run that returns without a successful `dotnet build` (or an implicit build via `dotnet test`) of the project is a FAILURE.
- **Apply the wrong skill.** Use the build/fix conventions of the detected project type only (MAUI skill for MAUI, MVC skill for MVC) — never mix them.
- **Build the full MAUI solution.** Build the targeted `.csproj` and filter output to errors only, per `token-efficient-workflow`.
- **Force-pass tests by weakening assertions.** If a test is failing because the implementation is wrong, fix the production code — not the assertion. The only exception is correcting a genuinely wrong expected value (see triage table).
- **Run `git` commands.** Only `dotnet build` and `dotnet test` are permitted in this agent.
- **Modify test files outside `TEST-FILES`.** Do not touch test files that belong to other features or pre-existing test suites.
- **Mark pre-existing failures as fixed** unless this ticket's code changes directly caused the regression.
- **Make live network calls.** Do not add, modify, or enable any code path that calls `pwapi.peoplewith.com` or any external service during the test run.

## Notes

- Do NOT modify production code beyond what is necessary to make the build succeed or a failing test pass
- Do NOT delete or skip tests to achieve a green build — fix the underlying issue
- **DO run `dotnet build <MainProject>.csproj` first** to catch compilation bugs early and fix them per the detected project type's skill; then run `dotnet test`
- Pre-existing failures in unrelated tests must be flagged, not fixed
