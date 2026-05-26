---
name: repo-discovery
description: Determine which GitLab repos are affected by a Jira ticket. Analyzes the ticket, searches GitLab, verifies repos exist, and posts decisions or questions back to the ticket.
argument-hint: ticket-key
user-invokable: true
---

# Repo Discovery: $ARGUMENTS

Determine which GitLab repositories are affected by Jira ticket **$ARGUMENTS**.

---

## Jira Bootstrap

**IMPORTANT: Always use `mcp__atlassian__*` tools for ALL Jira operations. Never use curl or direct REST API calls for Jira.**

1. Validate that `$ARGUMENTS` matches the format `ABC-1234` (project prefix, hyphen, number). If not, stop immediately.
2. Get the cloudId using `mcp__atlassian__getAccessibleAtlassianResources`. Use the first result's `id` as the `cloudId`.
3. Fetch the ticket using `mcp__atlassian__getJiraIssue` with `issueIdOrKey: $ARGUMENTS`, `expand: "renderedFields"`, and `fields: ["summary", "description", "status", "labels", "comment", "reporter", "customfield_15962"]`. This single call returns all fields and comments needed — do not make additional calls for this data.
4. Extract `fields.reporter.accountId` for tagging in comments.

---

## Step 1: Check for Existing Repo Decisions

1. Using the comments already fetched in Jira Bootstrap, scan for a `**REPO DECISIONS**` comment.
2. If found, the repos have already been determined. Skip to Step 5 (advance to next skill).

---

## Step 2: Check for Previous Questions and Answers

1. Using the comments already fetched in Jira Bootstrap, scan for any `**REPO QUESTIONS**` comments.
2. For each found, look for user replies posted **after** that comment (by timestamp).
3. Collect all previously asked questions and their answers into context.
4. **If a user reply exists** after a `**REPO QUESTIONS**` comment, proceed to Step 4 (use the user's answer to post decisions).

---

## Step 3: Identify Repos

### 3a: Check for Explicit Repos in Description

Scan the ticket description and title for explicit repository references:
- GitLab URLs (e.g. `https://gitlab.example.com/group/repo`)
- Full project paths (e.g. `group/repo-name`)

If an explicit repo is found, verify it exists using MCP:
```
mcp__gitlab__get_project with project_id: "<project-path-encoded>"
```

If verified, **skip to Step 5** (post decisions and advance).

### 3b: Search GitLab (No Explicit Repo Found)

**IMPORTANT: Always use `mcp__gitlab__*` MCP tools for GitLab operations. Only fall back to curl if the MCP tool is unavailable or fails.**

Extract keywords from the ticket description, title, and any prior user answers:
- Project names, service names, or component names
- Technology stack hints (frontend/backend/API references)
- Any other keywords that might match GitLab project names

For each keyword, search GitLab:
```
mcp__gitlab__search_repositories with search: "<keyword>"
```

**Fallback** (only if MCP tool is unavailable):
```bash
curl --silent --header "PRIVATE-TOKEN: $GITLAB_TOKEN" \
  "https://$GITLAB_HOST/api/v4/search?scope=projects&search=<keyword>"
```

### 3c: Post Findings for User Confirmation

Whether candidates were found or not, **always ask the user to confirm** (do NOT post `**REPO DECISIONS**` yet):

1. Add the `waiting_for_user` label:
   - Fetch current labels from the ticket
   - Add `waiting_for_user` to the array
   - Update via `mcp__atlassian__editJiraIssue` with `fields: { "labels": [...existing, "waiting_for_user"] }`

2. Post a comment using `mcp__atlassian__addCommentToJiraIssue`, tagging the reporter:

   **If candidate repos were found:**
   ```
   **REPO QUESTIONS**

   [~accountId:<reporter-accountId>] Based on this ticket, I believe the following repositories are affected:

   - <repo-name-1>: <gitlab-url-1>
   - <repo-name-2>: <gitlab-url-2>

   Are these correct? If not, please provide the correct repository names or URLs.
   ```

   **If no candidates were found:**
   ```
   **REPO QUESTIONS**

   [~accountId:<reporter-accountId>] I was unable to identify which repositories are affected by this ticket.

   Could you provide the repository names or GitLab URLs?
   ```

3. **Do NOT update the AI Status field.** Leave it as-is so that the next run re-enters repo discovery.
4. **Stop.** The workflow will resume when the user replies.

---

## Step 4: Process User Answer from Previous Questions

This step runs when Step 2 found a user reply after a `**REPO QUESTIONS**` comment.

1. Extract repo names or URLs from the user's reply.
2. Verify each repo exists using MCP:
   ```
   mcp__gitlab__get_project with project_id: "<project-path-encoded>"
   ```
3. If verification fails, post another `**REPO QUESTIONS**` comment explaining the failure and asking for corrections. Add/keep `waiting_for_user` label. **Stop.**
4. If verified, proceed to Step 5.

---

## Step 5: Post Repo Decisions and Advance

Once repos are verified (either from an explicit mention in Step 3a, or from user confirmation in Step 4):

1. Update the AI Status field:
   ```
   mcp__atlassian__editJiraIssue with fields: { "customfield_15962": "implementation_discovery" }
   ```

2. Post a `**REPO DECISIONS**` comment using `mcp__atlassian__addCommentToJiraIssue`:
   ```
   **REPO DECISIONS**

   The following repositories are affected by this ticket:

   - <repo-name-1>: <gitlab-url-1>
   - <repo-name-2>: <gitlab-url-2>
   ```

3. Remove the `waiting_for_user` label if present:
   - Fetch current labels from the ticket
   - Filter out `waiting_for_user`
   - Update via `mcp__atlassian__editJiraIssue` with `fields: { "labels": [<filtered-labels>] }`

4. **Stop.** The comment will trigger the webhook, which invokes `/ticket-discovery`. The router will read the updated AI Status (`implementation_discovery`) and route to the next skill.
