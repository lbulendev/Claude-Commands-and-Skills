---
name: jira-comment
description: Add a comment to a Jira ticket. Use when the user wants to post a comment on a Jira issue.
argument-hint: [ticket-key]
user-invocable: true
---

# Add Comment to Jira Ticket

Add a comment to Jira ticket **$ARGUMENTS**.

## Steps

1. **Validate the ticket key**. It must match the format `ABC-1234` (project prefix, hyphen, number). If invalid, inform the user and stop.

2. **Ask the user** what the comment should contain using the AskUserQuestion tool. Provide options for common comment types:
   - **Implementation Decisions** - Questions and answers from planning sessions (prefixed with `**IMPLEMENTATION DECISIONS**`)
   - **Implementation Plan** - A structured plan for implementation (prefixed with `**IMPLEMENTATION PLAN**`)
   - **Custom comment** - Free-form comment

3. **Compose the comment**. If the user selected Implementation Decisions or Implementation Plan, prefix the comment with the appropriate header (`**IMPLEMENTATION DECISIONS**` or `**IMPLEMENTATION PLAN**`).

4. **Post the comment** using the `mcp_atlassian_addCommentToJiraIssue` tool with:
   - The ticket key: `$ARGUMENTS`
   - The composed comment body

5. **Confirm** to the user that the comment was posted successfully.
