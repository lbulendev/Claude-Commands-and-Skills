---
name: implement
description: Implement a Jira ticket end-to-end. Reads the ticket, creates a plan, implements code in a worktree, runs quality checks, creates a GitLab merge request, and monitors the pipeline with auto-fixing of failures.
argument-hint: [ticket-key]
user-invocable: true
---

# Implement Jira Ticket $ARGUMENTS

Execute the full implementation workflow for Jira ticket **$ARGUMENTS**, from reading the ticket through creating a merge request.

---

## Phase 0: Runtime Detection

Before starting any work, detect the project environment by running these commands:

1. **Project root and repo name:**
   ```bash
   PROJECT_ROOT=$(git rev-parse --show-toplevel)
   REPO_NAME=$(basename "$PROJECT_ROOT")
   echo "PROJECT_ROOT=$PROJECT_ROOT"
   echo "REPO_NAME=$REPO_NAME"
   ```

2. **Default branch:**
   ```bash
   DEFAULT_BRANCH=$(git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's@^refs/remotes/origin/@@')
   if [ -z "$DEFAULT_BRANCH" ]; then
     git remote set-head origin --auto 2>/dev/null
     DEFAULT_BRANCH=$(git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's@^refs/remotes/origin/@@')
   fi
   DEFAULT_BRANCH=${DEFAULT_BRANCH:-main}
   echo "DEFAULT_BRANCH=$DEFAULT_BRANCH"
   ```

3. **GitLab remote URL:**
   ```bash
   REMOTE_URL=$(git remote get-url origin)
   # Extract host and project path from SSH or HTTPS URL
   # SSH:   git@gitlab.example.com:group/project.git
   # HTTPS: https://gitlab.example.com/group/project.git
   GITLAB_HOST=$(echo "$REMOTE_URL" | sed -E 's#^(https?://|git@)([^:/]+)[:/].*#\2#')
   GITLAB_PROJECT_PATH=$(echo "$REMOTE_URL" | sed -E 's#^(https?://|git@)[^:/]+[:/](.+?)(\.git)?$#\2#')
   GITLAB_PROJECT_PATH_ENCODED=$(echo "$GITLAB_PROJECT_PATH" | sed 's/\//%2F/g')
   echo "GITLAB_HOST=$GITLAB_HOST"
   echo "GITLAB_PROJECT_PATH=$GITLAB_PROJECT_PATH"
   echo "GITLAB_PROJECT_PATH_ENCODED=$GITLAB_PROJECT_PATH_ENCODED"
   ```

4. **Package manager:**
   ```bash
   if [ -f "$PROJECT_ROOT/pnpm-lock.yaml" ]; then
     PKG_MANAGER="pnpm"
   elif [ -f "$PROJECT_ROOT/yarn.lock" ]; then
     PKG_MANAGER="yarn"
   elif [ -f "$PROJECT_ROOT/bun.lockb" ] || [ -f "$PROJECT_ROOT/bun.lock" ]; then
     PKG_MANAGER="bun"
   else
     PKG_MANAGER="npm"
   fi
   echo "PKG_MANAGER=$PKG_MANAGER"
   ```

5. **Build system and quality check commands:**
   ```bash
   if [ -f "$PROJECT_ROOT/nx.json" ]; then
     BUILD_SYSTEM="nx"
   else
     BUILD_SYSTEM="scripts"
   fi
   echo "BUILD_SYSTEM=$BUILD_SYSTEM"
   ```

   Derive quality commands based on the build system:

   | Build System | Test | Lint | Typecheck | Format |
   |---|---|---|---|---|
   | **nx** | `npx nx affected -t test --base=<DEFAULT_BRANCH>` | `npx nx affected -t lint --base=<DEFAULT_BRANCH>` | `npx nx affected -t typecheck --base=<DEFAULT_BRANCH>` | `npx nx affected -t format --base=<DEFAULT_BRANCH>` |
   | **scripts** | `<PKG_MANAGER> run test` | `<PKG_MANAGER> run lint` | `<PKG_MANAGER> run typecheck` | `<PKG_MANAGER> run format` |

   For `scripts` mode, check `package.json` to see which scripts actually exist. Skip any quality check whose script is not defined.

6. Store all detected values as variables for use throughout the remaining phases. Display a summary to the user for confirmation before proceeding.

---

## Phase 1: Read the Jira Ticket

1. Validate that `$ARGUMENTS` matches the format `ABC-1234`. If not, inform the user and **stop immediately**.

2. Get the cloudId first using `mcp__atlassian__getAccessibleAtlassianResources`. This returns an array of accessible resources - use the first one's `id` as the `cloudId`.

3. Use `mcp__atlassian__getJiraIssue` to fetch ticket `$ARGUMENTS`:
   - Required parameters:
     - `cloudId`: the cloudId from step 2
     - `issueIdOrKey`: `$ARGUMENTS` (e.g., "ABC-1234")

4. If the ticket is not found or returns an error, inform the user and **stop immediately**. Do not continue the workflow.

5. Extract from the response:
   - **Title**: `fields.summary`
   - **Description**: `fields.description`
   - **Comments**: Use `mcp__atlassian__getJiraIssue` with `expand: "renderedFields,names,schema,operations,editmeta,changelog,versionedRepresentations,comments"` to get all comments

6. Scan comments for:
   - A comment starting with `**IMPLEMENTATION PLAN**` → store as `existing_plan`
   - A comment starting with `**IMPLEMENTATION DECISIONS**` → store as `existing_decisions`

---

## Phase 2: Implementation Planning

### If an existing plan was found:

Display the existing plan to the user and confirm they want to proceed with it. If the user wants modifications, iterate on the plan with them.

Skip to **Phase 3**.

### If no existing plan was found:

#### 2a. Incorporate Decisions

If `existing_decisions` was found, use those Q&A pairs to supplement the description. When information conflicts between the description and decisions, **prefer the decisions**.

#### 2b. Explore the Codebase

Use the Task tool with `subagent_type=Explore` to understand the relevant parts of the codebase:
- Identify files and directories related to the ticket's requirements
- Understand existing patterns, conventions, and architecture
- Find related tests and test patterns
- Check AGENTS.md for relevant project conventions

#### 2c. Ask Clarifying Questions

Based on the description, decisions (if any), and codebase exploration:
- Identify ambiguities or gaps in the requirements
- **Do not make assumptions** - ask about anything unclear
- Ask about LaunchDarkly feature flags if one is not specified but seems relevant
- Use the AskUserQuestion tool to ask the user interactively
- Collect all answers

#### 2d. Post Implementation Decisions to Jira

If any clarifying questions were asked, post a comment to ticket `$ARGUMENTS` using `mcp__atlassian__addCommentToJiraIssue`:

**Parameters:**
- `cloudId`: the cloudId from Phase 1
- `issueIdOrKey`: `$ARGUMENTS` (e.g., "ABC-1234")
- `commentBody`: the formatted comment text (see below)

**Comment format:**
```
**IMPLEMENTATION DECISIONS**

Q: [question]
A: [answer]

Q: [question]
A: [answer]
```

**Important:** If NO clarifying questions were asked (user had no questions or all information was clear), SKIP this step entirely and proceed to 2e. Do not post an empty decisions comment.

#### 2e. Build the Implementation Plan

Create a plan that includes:

1. **Summary** - Brief overview of what will be implemented
2. **Files to Modify** - List of files to create or modify, with a description of changes for each
3. **Implementation Steps** - Ordered steps with enough detail to execute
4. **Tests** - Tests covering all acceptance criteria:
   - Positive tests (happy path)
   - Negative tests (error cases, edge cases, boundary conditions)
5. **Risks and Considerations** - Potential risks or trade-offs

#### 2f. Get User Approval

Present the plan to the user. Iterate based on their feedback until approved.

#### 2g. Post Plan to Jira

Once approved, post the plan as a comment using `mcp__atlassian__addCommentToJiraIssue`:

**Parameters:**
- `cloudId`: the cloudId from Phase 1
- `issueIdOrKey`: `$ARGUMENTS` (e.g., "ABC-1234")
- `commentBody`: the formatted plan (see below)

**Comment format:**
```
**IMPLEMENTATION PLAN**

[full plan content]
```

---

## Phase 3: Create Worktree and Branch

1. Derive the branch name from the ticket: `$ARGUMENTS` key + abbreviated title in kebab-case.
   - Example: ticket `JAR-1234` with title "Add new folder functionality" → branch `JAR-1234-add-new-folder-functionality`

2. Ensure the main repository is on the default branch and up to date:
   ```bash
   git fetch origin <DEFAULT_BRANCH>
   git checkout <DEFAULT_BRANCH>
   git pull origin <DEFAULT_BRANCH>
   ```

3. Create the worktree:
   ```bash
   git worktree add ../<REPO_NAME>-$ARGUMENTS -b <branch-name>
   ```

4. Install dependencies in the worktree:
   ```bash
   cd ../<REPO_NAME>-$ARGUMENTS
   <PKG_MANAGER> install
   ```

5. All subsequent commands in Phases 4-7 must run **inside the worktree directory**.

---

## Phase 4: Implement the Changes

Working from the worktree directory (`../<REPO_NAME>-$ARGUMENTS`):

1. Follow the implementation plan step by step.

2. For each step:
   - Read existing files before modifying them
   - Follow existing code patterns and conventions
   - Write clean, focused code - no over-engineering
   - Write tests as specified in the plan

3. Use the Task tool with sub-agents for complex implementation steps to keep context focused.

---

## Phase 5: Quality Checks

Working from the worktree directory:

Run each quality check using the commands detected in Phase 0. Skip any check that does not exist for the detected build system.

### 5a. Run Tests

Run the detected test command. If tests fail, fix the implementation and re-run until all pass.

### 5b. Run Linting

Run the detected lint command. Fix any linting errors and re-run until clean.

### 5c. Run Type Checking

Run the detected typecheck command. Fix any type errors and re-run until clean.

### 5d. Run Formatting

Run the detected format command. Fix any formatting issues. If the build system supports a `format:fix` variant, try that for auto-fix.

---

## Phase 6: Commit and Push

Working from the worktree directory:

1. Check status:
   ```bash
   git status
   ```

2. Stage the changes (stage specific files, not `git add -A`):
   ```bash
   git add <file1> <file2> ...
   ```

3. Commit with the ticket number prefix:
   ```bash
   git commit -m "$(cat <<'EOF'
   $ARGUMENTS: Brief imperative description of changes

   Co-Authored-By: Claude AI
   EOF
   )"
   ```

4. Push the branch:
   ```bash
   git push -u origin <branch-name>
   ```

---

## Phase 7: Create Merge Request

1. Compose the MR title: `$ARGUMENTS: Brief imperative description` (under 70 characters).

2. Compose the MR description: clear, concise, imperative mood, no emojis. Use this format:

   ```markdown
   ## Summary

   - Key change 1
   - Key change 2
   - Key change 3

   Generated with [Claude Code](https://claude.com/claude-code)
   ```

   Include a brief paragraph about complex decisions if applicable.

3. Create the MR:
   ```bash
   curl --request POST \
     --header "PRIVATE-TOKEN: $GITLAB_TOKEN" \
     --header "Content-Type: application/json" \
     --data '{
       "source_branch": "<branch-name>",
       "target_branch": "<DEFAULT_BRANCH>",
       "title": "<mr-title>",
       "description": "<mr-description>",
       "remove_source_branch": true,
       "squash": true
     }' \
     "https://<GITLAB_HOST>/api/v4/projects/<GITLAB_PROJECT_PATH_ENCODED>/merge_requests"
   ```

4. Parse the JSON response and extract `web_url`.

5. Display the MR URL to the user.

6. Parse the JSON response and extract:
   - `web_url` — the MR URL to display
   - `iid` — the merge request internal ID
   - `project_id` — the **numeric** project ID (e.g., `5627`)

   **Important:** Store the numeric `project_id` and use it for **all** subsequent GitLab API calls in Phase 8. Do NOT use the URL-encoded project path — it may return 404 depending on token permissions. The API base for Phase 8 is:
   ```
   https://<GITLAB_HOST>/api/v4/projects/<PROJECT_ID>
   ```

---

## Phase 8: Monitor and Fix Pipeline

Working from the worktree directory.

**API base for all Phase 8 calls:** `https://<GITLAB_HOST>/api/v4/projects/<PROJECT_ID>` (using the numeric `project_id` from Phase 7).

### 8a. Wait for Pipeline to Start

After the MR is created, GitLab may take a few moments to create the pipeline.

1. Query the MR pipelines (no initial sleep — just query immediately):
   ```bash
   curl --silent --header "PRIVATE-TOKEN: $GITLAB_TOKEN" \
     "https://<GITLAB_HOST>/api/v4/projects/<PROJECT_ID>/merge_requests/<iid>/pipelines"
   ```

2. Parse the full JSON response first. If the response contains an error or is not a JSON array, log the raw response for debugging.

3. Extract the most recent pipeline ID from the response (the first item in the array).

4. If the array is empty, wait 10 seconds and retry. If no pipeline is found after 3 attempts, inform the user and skip to Phase 9.

### 8b. Monitor Pipeline Status

Set up a monitoring loop with these parameters:
- Maximum monitoring time: 30 minutes
- Poll interval: 30 seconds
- Maximum auto-fix attempts: 3

For each poll cycle:

1. Query the pipeline status:
   ```bash
   curl --silent --header "PRIVATE-TOKEN: $GITLAB_TOKEN" \
     "https://<GITLAB_HOST>/api/v4/projects/<PROJECT_ID>/pipelines/<pipeline-id>"
   ```

2. Parse the full JSON response. If parsing fails or the response contains an error message, log the raw response and retry once before treating it as a failure.

3. Check the `status` field:
   - `pending` or `running` → continue monitoring
   - `success` → inform user and skip to Phase 9
   - `failed` → proceed to 8c
   - `canceled` or `skipped` → inform user and skip to Phase 9
   - `manual` → inform user that manual action is needed and skip to Phase 9

4. Display periodic status updates to the user (every 2-3 polls) so they know the pipeline is being monitored.

### 8c. Analyze Pipeline Failure

When the pipeline fails:

1. Query all jobs in the pipeline:
   ```bash
   curl --silent --header "PRIVATE-TOKEN: $GITLAB_TOKEN" \
     "https://<GITLAB_HOST>/api/v4/projects/<PROJECT_ID>/pipelines/<pipeline-id>/jobs"
   ```

2. Identify failed jobs (status = `failed`).

3. For each failed job:
   - Extract the job ID and name
   - Fetch the job trace (log output):
     ```bash
     curl --silent --header "PRIVATE-TOKEN: $GITLAB_TOKEN" \
       "https://<GITLAB_HOST>/api/v4/projects/<PROJECT_ID>/jobs/<job-id>/trace"
     ```

4. Analyze each failed job trace to determine:
   - **Type of failure**: test failure, lint error, type error, build error, etc.
   - **Specific error messages**: extract relevant error details
   - **Root cause**: identify what needs to be fixed
   - **Is it auto-fixable?**: determine if this is something that can be fixed programmatically

### 8d. Attempt Auto-Fix

If the failure appears auto-fixable and we haven't exceeded the maximum auto-fix attempts:

1. **Categorize the failure** and apply the appropriate fix strategy:

   **Test Failures:**
   - Read the test file and implementation
   - Analyze why the test is failing
   - Fix the implementation or test as appropriate
   - Re-run tests locally using the detected test command from Phase 0

   **Linting Errors:**
   - Try the lint auto-fix variant if available (e.g., `lint:fix` target for nx/turbo)
   - If auto-fix doesn't work, read the errors and fix manually
   - Verify using the detected lint command from Phase 0

   **Type Errors:**
   - Read the files with type errors
   - Fix type issues
   - Verify using the detected typecheck command from Phase 0

   **Formatting Issues:**
   - Try the format auto-fix variant if available (e.g., `format:fix` target)
   - Verify using the detected format command from Phase 0

   **Build Errors:**
   - Read the error output carefully
   - Identify missing dependencies, syntax errors, or configuration issues
   - Fix the issues
   - Verify with the appropriate build command

   **Dependency Issues:**
   - If dependencies are missing or mismatched, run `<PKG_MANAGER> install`
   - Check for lock file conflicts

2. **Verify the fix locally** by running the same check that failed in CI.

3. **Commit and push the fix:**
   ```bash
   git status
   git add <files>
   git commit -m "$(cat <<'EOF'
   $ARGUMENTS: Fix pipeline failure - <brief description>

   Co-Authored-By: Claude AI
   EOF
   )"
   git push
   ```

4. Increment the auto-fix attempt counter.

5. **Wait for new pipeline** (10 seconds), then query for the new pipeline ID and return to 8b to monitor it.

### 8e. Handle Non-Fixable Failures

If the failure is not auto-fixable or maximum attempts have been reached:

1. Display a summary to the user:
   - Failed job names
   - Error messages
   - Why it couldn't be auto-fixed
   - Suggestions for manual intervention

2. Provide the command to view the pipeline:
   ```
   View pipeline: https://<GITLAB_HOST>/<GITLAB_PROJECT_PATH>/-/pipelines/<pipeline-id>
   ```

3. Continue to Phase 9.

---

## Phase 9: Cleanup

1. Return to the main repository directory:
   ```bash
   cd <PROJECT_ROOT>
   ```

2. Inform the user that the workflow is complete with:
   - Jira ticket key
   - Branch name
   - MR URL
   - Pipeline status (success/failed/monitoring stopped)
   - Number of auto-fix attempts made (if any)
   - Summary of what was implemented
   - Worktree location (in case they need to make manual adjustments)
