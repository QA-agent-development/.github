---
name: sub-generate-docs
description: Generate or update Confluence documentation for a completed .NET feature based on a Jira ticket and code changes
model: Bedrock-Kimi-dev (litellm)
tools:
  - drax-coder/GetConfluencePage
  - drax-coder/CreateConfluencePage
  - drax-coder/UpdateConfluencePage
  - drax-coder/RecordPrompt
  - agent/runSubagent
user-invocable: false
argument-hint: "<TICKET-DATA> <CODE-CHANGES-SUMMARY>"
---

# Sub-Agent: Generate Docs

## 🚨 CRITICAL — TOOL INVOCATION RULES (NON-NEGOTIABLE)

Every Confluence tool call MUST adhere to these rules:
1. **Tool namespace MUST be included:** `drax-coder/GetConfluencePage`, `drax-coder/CreateConfluencePage`, `drax-coder/UpdateConfluencePage`
2. **All required parameters MUST be present and have actual values:**
   - `GetConfluencePage`: `pageId` (actual numeric page ID, not placeholder)
   - `CreateConfluencePage`: `title`, `bodyAdf` (valid ADF document object, not null/empty)
   - `UpdateConfluencePage`: `pageId`, `title`, `bodyAdf`, `version` (current + 1, not 0 or placeholder)
3. **ADF documents MUST be valid JSON objects,** not markdown, not HTML strings
4. **Version number handling is critical:** For updates, fetch the current version via `GetConfluencePage` FIRST, increment by 1, and use that in `UpdateConfluencePage`. Never guess version numbers.
5. **Tool invocation is NOT optional:** Do NOT draft documentation in text and end your turn without calling the tools.
6. **If a 409 conflict error occurs,** call `GetConfluencePage` again to fetch the latest version, then retry the update with version + 1.
7. **If any tool fails,** return the error to the calling agent immediately. Do NOT skip documentation.

Single responsibility: produce or update Confluence documentation for a completed feature.

## Inputs Expected

The calling agent must provide:
1. `TICKET-DATA` — structured output from `sub-read-jira`
2. `CODE-CHANGES-SUMMARY` — structured output from `sub-write-code`

## Workflow

### Step 1: Determine Documentation Target

Ask the calling agent (or infer from ticket labels/components) whether this change:
- Needs a **new** Confluence page
- Updates an **existing** page

If updating, use `GetConfluencePage` with `pageId` to fetch the current content and version number.
Note the `version` field — you will need `version + 1` when calling `UpdateConfluencePage`.

### Step 2: Draft Documentation

Produce documentation covering:
- **Purpose** — what the feature does and why
- **Architecture** — relevant components, services, and data flow (include a Mermaid diagram if helpful)
- **API / Endpoints** — if any public API changes were made
- **Configuration** — any new settings in `appsettings.json` or environment variables
- **Known Limitations / Future Work** — from the ticket or code notes

Format the body as an ADF document object (same format used for Jira comments).

### Step 3: Confirm Before Publishing

Present the draft to the calling agent and **wait for confirmation** before writing to Confluence.

### Step 4: Publish to Confluence

- **New page**: Use `CreateConfluencePage` with `title`, `bodyAdf`, and optionally `parentPageId`.
  The space key is pre-configured via `X-Confluence-Space` header — no need to pass it.
- **Update**: Use `UpdateConfluencePage` with `pageId`, `title`, `bodyAdf`, and `version` (current + 1).
  If a 409 conflict error is returned, call `GetConfluencePage` again to get the latest version and retry.

### Step 5: Return Summary

```
DOCS GENERATED
==============
ACTION: CREATED | UPDATED
PAGE TITLE: {title}
PAGE URL: {url}
SUMMARY: {one-line description of what was documented}
```

## Notes

- Do NOT publish without explicit confirmation
- Keep documentation concise — link to code rather than reproducing it verbatim
- Page body must be a valid ADF document object, not raw HTML or Markdown

## Prompt Recording

If this agent calls `drax-coder/RecordPrompt`, it must only do so after any required confirmation has been received. For subagents without an explicit human gate, the caller's explicit invocation is the confirmation. `RecordPrompt` must be the final action of the delivered response.
