---
name: jira-read
description: Read a Jira ticket's title, description, and comments. Use when the user wants to view or understand a Jira ticket.
argument-hint: [ticket-key]
user-invocable: true
---

# Read Jira Ticket

Read and display the contents of Jira ticket **$ARGUMENTS**.

## Steps

1. **Validate the ticket key**. It must match the format `ABC-1234` (project prefix, hyphen, number). If invalid, inform the user and stop.

2. **Get the cloudId** using `mcp__corteva__atlassian__getAccessibleAtlassianResources`. Use the first result's `id` as the `cloudId`.

3. **Fetch the ticket** using `mcp__corteva__atlassian__getJiraIssue` with `cloudId: <cloudId>`, `issueIdOrKey: $ARGUMENTS`, `expand: "renderedFields"`, `fields: ["summary", "description", "status", "labels", "comment", "reporter"]`, and `responseContentFormat: "markdown"`. This single call returns all fields and comments needed — do not make additional calls for this data.

4. If the ticket cannot be found or the tool returns an error, inform the user that the ticket was not found and stop immediately.

5. **Display the ticket details** in this format:

   - **Key**: The ticket key
   - **Title**: The ticket summary/title
   - **Status**: The current status
   - **Description**: The full description
   - **Comments**: List all comments, noting the author and timestamp

6. **Highlight special comments**. Look for comments that begin with:
   - `**IMPLEMENTATION PLAN**` - Flag this as an existing implementation plan
   - `**IMPLEMENTATION DECISIONS**` - Flag this as existing implementation decisions

7. Provide a brief summary of the ticket's requirements at the end.
