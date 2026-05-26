---
name: reset-ai-status
description: Reset the AI Status custom field on a Jira ticket back to none. Use when the user wants to clear or reset the AI status on a ticket.
argument-hint: ticket-key
user-invokable: true
---

# Reset AI Status: $ARGUMENTS

Reset the AI Status field on Jira ticket **$ARGUMENTS** back to `none`.

## Steps

1. **Validate the ticket key**. It must match the format `ABC-1234` (project prefix, hyphen, number). If invalid, inform the user and stop.

2. **Get the cloud ID** using `mcp__atlassian__getAccessibleAtlassianResources`. Use the first result's `id` as the `cloudId`.

3. **Reset AI Status** using `mcp__atlassian__editJiraIssue` with:
   - `cloudId`: from step 2
   - `issueIdOrKey`: `$ARGUMENTS`
   - `fields`: `{ "customfield_15962": null }`

4. **Confirm** to the user that the AI Status on **$ARGUMENTS** has been reset to `none`.
