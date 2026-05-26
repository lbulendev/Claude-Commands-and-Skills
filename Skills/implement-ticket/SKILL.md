---
name: implement-ticket
description: Orchestrate implementation across multiple repos for a Jira ticket. Clones repos, detects types, delegates to frontend/backend executors, and reports results back to Jira.
argument-hint: ticket-key
user-invokable: true
---

# Implement Ticket: $ARGUMENTS

Orchestrate the implementation of Jira ticket **$ARGUMENTS** across all affected repositories.

---

## Jira Bootstrap

**IMPORTANT: Always use `mcp__atlassian__*` tools for ALL Jira operations. Never use curl or direct REST API calls for Jira.**

1. Validate that `$ARGUMENTS` matches the format `ABC-1234` (project prefix, hyphen, number). If not, stop immediately.
2. Get the cloudId using `mcp__atlassian__getAccessibleAtlassianResources`. Use the first result's `id` as the `cloudId`.
3. Fetch the ticket using `mcp__atlassian__getJiraIssue` with `issueIdOrKey: $ARGUMENTS`, `expand: "renderedFields"`, and `fields: ["summary", "description", "status", "labels", "comment", "reporter", "customfield_15962"]`. This single call returns all fields and comments needed — do not make additional calls for this data.
4. Extract `fields.reporter.accountId` for tagging in comments.

---

## Step 1: Parse Implementation Plan

1. Using the comments already fetched in Jira Bootstrap, scan for the `**IMPLEMENTATION PLAN**` comment.
2. If no plan is found, stop immediately — the planning phase has not been completed.
3. Parse the plan into per-repo sections. Each section should be identified by repo name or URL.

---

## Step 2: Parse Repo Decisions

1. Using the comments already fetched in Jira Bootstrap, scan for the `**REPO DECISIONS**` comment.
2. Extract the list of repos with their GitLab URLs.
3. Match each repo URL to its corresponding plan section.

---

## Step 3: Clone and Detect Repos

For each repo from the REPO DECISIONS:

1. **Clone the repo** (if not already local) to a temporary directory:
   ```bash
   CLONE_DIR="/tmp/claude-impl-$ARGUMENTS/$(basename <repo-url> .git)"
   if [ ! -d "$CLONE_DIR" ]; then
     git clone --depth=50 <repo-url> "$CLONE_DIR"
   fi
   ```

2. **Detect repo type** by checking project files:
   ```bash
   # Check for mobile project indicators
   HAS_IOS=false
   HAS_ANDROID=false
   if [ -d "$CLONE_DIR/swift" ] || [ -d "$CLONE_DIR/ios" ] || find "$CLONE_DIR" -maxdepth 2 -name "*.xcodeproj" | grep -q .; then
     HAS_IOS=true
   fi
   if ([ -d "$CLONE_DIR/android" ] && ([ -f "$CLONE_DIR/android/settings.gradle.kts" ] || [ -f "$CLONE_DIR/android/settings.gradle" ])) || \
      ([ -f "$CLONE_DIR/settings.gradle.kts" ] || [ -f "$CLONE_DIR/settings.gradle" ]) && find "$CLONE_DIR" -maxdepth 2 -name "AndroidManifest.xml" | grep -q .; then
     HAS_ANDROID=true
   fi

   if [ "$HAS_IOS" = true ] && [ "$HAS_ANDROID" = true ]; then
     REPO_TYPE="mobile"
   elif [ "$HAS_IOS" = true ]; then
     REPO_TYPE="ios"
   elif [ "$HAS_ANDROID" = true ]; then
     REPO_TYPE="android"
   elif [ -f "$CLONE_DIR/package.json" ] || [ -f "$CLONE_DIR/nx.json" ]; then
     REPO_TYPE="frontend"
   elif [ -f "$CLONE_DIR/pyproject.toml" ] || [ -f "$CLONE_DIR/setup.py" ]; then
     REPO_TYPE="backend"
   else
     REPO_TYPE="unknown"
   fi
   ```

3. If `REPO_TYPE` is `unknown`, attempt to infer from file extensions and directory structure. If still unknown, treat as a failure for this repo.

---

## Step 4: Delegate to Executors

For each repo, invoke the appropriate executor skill:

- **mobile** -> `Skill tool: skill: "implement-mobile", args: "$ARGUMENTS"` (run from within `CLONE_DIR`)
- **ios** -> `Skill tool: skill: "implement-ios", args: "$ARGUMENTS <CLONE_DIR>"`
- **android** -> `Skill tool: skill: "implement-android", args: "$ARGUMENTS <CLONE_DIR>"`
- **frontend** -> `Skill tool: skill: "implement-frontend", args: "$ARGUMENTS <CLONE_DIR>"`
- **backend** -> `Skill tool: skill: "implement-backend", args: "$ARGUMENTS <CLONE_DIR>"`

Collect the result from each invocation:
- **Success:** MR URL
- **Failure:** error details

---

## Step 5: Report Results

### If all repos succeeded:

1. Update the AI Status field:
   ```
   mcp__atlassian__editJiraIssue with fields: { "customfield_15962": "done" }
   ```

2. Post a summary comment to the ticket using `mcp__atlassian__addCommentToJiraIssue`:
   ```
   **IMPLEMENTATION COMPLETE**

   All repositories have been implemented and merge requests created:

   - <repo-name-1>: <MR-URL-1>
   - <repo-name-2>: <MR-URL-2>
   ```

### If any repo failed:

1. Update the AI Status field:
   ```
   mcp__atlassian__editJiraIssue with fields: { "customfield_15962": "needs_assistance" }
   ```

2. Add the `waiting_for_user` label:
   - Fetch current labels from the ticket
   - Add `waiting_for_user` to the array
   - Update via `mcp__atlassian__editJiraIssue` with `fields: { "labels": [...existing, "waiting_for_user"] }`

3. Post a comment using `mcp__atlassian__addCommentToJiraIssue`, tagging the reporter:
   ```
   **IMPLEMENTATION QUESTIONS**

   [~accountId:<reporter-accountId>] Implementation encountered issues in the following repositories:

   ### <repo-name>
   - **Error:** <error details>
   - **What was tried:** <description of auto-fix attempts>
   - **Help needed:** <specific ask>

   Successful repositories:
   - <repo-name>: <MR-URL>
   ```
