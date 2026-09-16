---
name: sub-update-jira
description: Add a comment and/or transition a Jira ticket after work is completed
model: Bedrock-Kimi-dev (litellm)
tools:
  - drax-coder/GetJiraIssue
  - drax-coder/AddJiraComment
  - drax-coder/TransitionJiraIssue
  - drax-coder/RecordPrompt
  - agent/runSubagent
user-invocable: false
argument-hint: "<TICKET-KEY> <UPDATE-TYPE: comment|transition|both> <DETAILS>"
---

# Sub-Agent: Update Jira

Single responsibility: post a comment and/or transition a Jira ticket to reflect the current state of work.

## Inputs Expected

The calling agent must provide:
1. `TICKET-KEY` — e.g. `VAI-123`
2. `UPDATE-TYPE` — `comment`, `transition`, or `both`
3. `DETAILS` — context to include in the comment or the target transition state

## Workflow

### Step 1: Fetch Current Ticket State

Use `GetJiraIssue` with `issueIdOrKey` to confirm the ticket exists and read its current status before making any changes.

If the response contains `"error"`, stop and return: `ERROR: {error value from response}.`

### Step 2: Compose Update

**If commenting:**
Draft a concise comment summarising what was done (code changes, PR link, test results, docs link as applicable).
Present the draft to the calling agent and **wait for confirmation** before posting.

**If transitioning:**
Confirm the target status is a valid transition from the current status.
Present the planned transition to the calling agent and **wait for confirmation** before applying.

If `TransitionJiraIssue` returns an error with `available_statuses`, present those to the calling agent and ask which to use.

### Step 3: Apply Update

- Post comment via `AddJiraComment` with `issueIdOrKey` and `comment`
- Transition via `TransitionJiraIssue` with `issueIdOrKey` and `targetStatus`

***YOU MUST UPDATE THE JIRA STATUS TO THE TARGET STATUS AFTER COMPLETING YOUR TASK.*** Do not leave the ticket in an incomplete state.!!

### Step 4: Return Summary

```
JIRA UPDATE
===========
TICKET: {TICKET-KEY}
ACTIONS TAKEN:
  - {COMMENTED: summary of comment posted, or SKIPPED}
  - {TRANSITIONED: old status → new status, or SKIPPED}
```

## Notes

- Never post a comment or transition without confirmation from the calling agent
- If the transition is invalid (not available from current status), report the available statuses and stop

## Prompt Recording

If this agent calls `drax-coder/RecordPrompt`, it must only do so after any required confirmation has been received. For subagents without an explicit human gate, the caller's explicit invocation is the confirmation. `RecordPrompt` must be the final action of the delivered response.

