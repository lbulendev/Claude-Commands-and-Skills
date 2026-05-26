---
name: ticket-discovery
description: Entry point for async Jira-driven implementation. Reads ticket state (AI Status field, labels) and routes to the correct workflow skill.
argument-hint: ticket-key
user-invokable: true
---

# Ticket Discovery: $ARGUMENTS

Entry point for all webhook-driven and manual invocations. Reads the current state of Jira ticket **$ARGUMENTS** and routes to the appropriate workflow skill.

---

## Jira Bootstrap

**IMPORTANT: Always use `mcp__atlassian__*` tools for ALL Jira operations. Never use curl or direct REST API calls for Jira.**

1. Validate that `$ARGUMENTS` matches the format `ABC-1234` (project prefix, hyphen, number). If not, stop immediately.
2. Get the cloudId using `mcp__atlassian__getAccessibleAtlassianResources`. Use the first result's `id` as the `cloudId`.
3. Fetch the ticket using `mcp__atlassian__getJiraIssue` with `issueIdOrKey: $ARGUMENTS`, `expand: "renderedFields"`, and `fields: ["summary", "description", "status", "labels", "comment", "reporter", "customfield_15962"]`. This single call returns all fields and comments needed — do not make additional calls for this data.
4. Extract `fields.reporter.accountId` for tagging in comments.

---

## Step 1: Check for Waiting Label

1. Read `fields.labels` from the ticket.
2. If the labels array contains `waiting_for_user`:
   - Log: "Ticket $ARGUMENTS has `waiting_for_user` label — user has not responded yet. Stopping."
   - **Stop.** Do not proceed further.

---

## Step 2: Read AI Status

1. Read the AI Status custom field: `fields.customfield_15962`.
2. Extract the current value. If the field is null or empty, treat as `none`.

---

## Step 3: Route Based on AI Status

| AI Status | Action |
|---|---|
| `none` / `null` / empty | Update AI Status to `repo_discovery` via `mcp__atlassian__editJiraIssue` with `fields: { "customfield_15962": "repo_discovery" }`, then invoke: `Skill tool: skill: "repo-discovery", args: "$ARGUMENTS"` |
| `repo_discovery` | Invoke: `Skill tool: skill: "repo-discovery", args: "$ARGUMENTS"` |
| `implementation_discovery` | Invoke: `Skill tool: skill: "implementation-discovery", args: "$ARGUMENTS"` |
| `implementation_planning` | Invoke: `Skill tool: skill: "implementation-planning", args: "$ARGUMENTS"` |
| `implementing` | Invoke: `Skill tool: skill: "implement-ticket", args: "$ARGUMENTS"` |
| `needs_assistance` | Update AI Status to `implementing` via `mcp__atlassian__editJiraIssue` with `fields: { "customfield_15962": "implementing" }`, then invoke: `Skill tool: skill: "implement-ticket", args: "$ARGUMENTS"` |
| `done` | Log: "Ticket $ARGUMENTS is already complete. No action needed." **Stop.** |
