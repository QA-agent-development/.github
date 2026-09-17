---
name: sub-read-jira
description: Fetch a Jira ticket and return structured data — summary, status, description, and acceptance criteria
model:  MAI-Code-1.1-Flash (copilot)
tools:
  - drax-coder/GetJiraIssue
  - drax-coder/RecordPrompt
user-invocable: false
argument-hint: "<TICKET-KEY>"
---

# Sub-Agent: Read Jira Ticket

Single responsibility: fetch a Jira ticket and return all relevant data in a structured format for the calling agent. `user-invocable: false` — this agent is only ever reached as a worker of `qa-agent-dev`, which has already run its own Token Budget, Authentication, and Client Configuration gates before invoking any worker. Do not repeat those gates here; re-running `AuthCheck` inside every worker call wastes a tool round-trip that the orchestrator already paid for.

## Prerequisites

The calling workspace must have the `drax-coder` MCP server configured in `.vscode/mcp.json`, including at minimum the Jira credentials this agent depends on:

```json
{
  "drax-coder": {
    "type": "http",
    "url": "http://localhost:3001/mcp",
    "headers": {
      "X-GitHub-Token": "<github-pat>",
      "X-Jira-Url": "https://yourcompany.atlassian.net",
      "X-Jira-Email": "you@example.com",
      "X-Jira-Token": "<atlassian-api-token>",
      "X-TestRail-Url": "<testrail-url>",
      "X-TestRail-Email": "<testrail-email>",
      "X-TestRail-Api-Key": "<testrail-api-key>",
      "X-Slack-Token": "<slack-bot-token>",
      "X-Slack-Channel": "agent-workflow"
    }
  }
}
```

The TestRail and Slack headers are not used by this agent directly, but by other workers sharing the same `drax-coder` server later in the same QA workflow — list them here so a fresh environment is configured once, correctly, rather than failing partway through Phase 2 or Phase 4.

## Workflow

### Step 1: Fetch Ticket

Use `GetJiraIssue` with:
- `issueIdOrKey`: the provided ticket key (e.g. `GPP-236`)

The Jira instance URL and credentials are taken automatically from the `X-Jira-*` headers in `mcp.json` — do not pass them as parameters.

If the tool is unavailable, or the response contains `"error"`, stop and return: `ERROR: {error value from response, or "GetJiraIssue is unavailable"}.`

### Step 2: Extract and Return Structured Data

Parse the JSON response from `GetJiraIssue`. Return the following structure clearly labeled so the calling agent can parse it:

```
TICKET: {key}
SUMMARY: {summary}
STATUS: {status}
DESCRIPTION:
{description}

ACCEPTANCE CRITERIA:
{acceptance_criteria, or "None" if absent}
```

Call `drax-coder/RecordPrompt` as the final tool action once the structured data (or the error) above has been produced.

## Notes

- Return all data verbatim — do not summarize or omit
- This agent does NOT post any comments or make any writes
- This agent never calls another agent — it holds no orchestration tool
