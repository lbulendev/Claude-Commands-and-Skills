---
name: implement-backend
description: Execute backend (Python) implementation for a single repo. Creates a worktree, implements changes from the Jira plan, runs quality checks, commits, creates a GitLab MR, and monitors the pipeline.
argument-hint: ticket-key repo-path
user-invokable: true
---

# Implement Backend: $ARGUMENTS

Execute the backend (Python) implementation for a single repository based on the implementation plan from a Jira ticket.

**Arguments:** `$ARGUMENTS` = `<ticket-key> <repo-path>`

Parse `$ARGUMENTS` to extract:
- `TICKET_KEY` — the first token (e.g., `ABC-1234`)
- `REPO_PATH` — the second token (e.g., `/tmp/repos/my-service`)

---

## Phase 0: Runtime Detection

Before starting any work, detect the project environment by running these commands from `REPO_PATH`:

1. **Project root and repo name:**
   ```bash
   cd <REPO_PATH>
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
   GITLAB_HOST=$(echo "$REMOTE_URL" | sed -E 's#^(https?://|git@)([^:/]+)[:/].*#\2#')
   GITLAB_PROJECT_PATH=$(echo "$REMOTE_URL" | sed -E 's#^(https?://|git@)[^:/]+[:/](.+?)(\.git)?$#\2#')
   GITLAB_PROJECT_PATH_ENCODED=$(echo "$GITLAB_PROJECT_PATH" | sed 's/\//%2F/g')
   echo "GITLAB_HOST=$GITLAB_HOST"
   echo "GITLAB_PROJECT_PATH=$GITLAB_PROJECT_PATH"
   echo "GITLAB_PROJECT_PATH_ENCODED=$GITLAB_PROJECT_PATH_ENCODED"
   ```

4. **Python package manager:**
   ```bash
   if [ -f "$PROJECT_ROOT/poetry.lock" ]; then
     PY_PKG_MANAGER="poetry"
     RUN_PREFIX="poetry run"
     INSTALL_CMD="poetry install"
   elif [ -f "$PROJECT_ROOT/uv.lock" ]; then
     PY_PKG_MANAGER="uv"
     RUN_PREFIX="uv run"
     INSTALL_CMD="uv sync"
   else
     PY_PKG_MANAGER="pip"
     RUN_PREFIX=""
     INSTALL_CMD="pip install -e '.[dev]'"
   fi
   echo "PY_PKG_MANAGER=$PY_PKG_MANAGER"
   echo "RUN_PREFIX=$RUN_PREFIX"
   echo "INSTALL_CMD=$INSTALL_CMD"
   ```

5. **Quality commands:**

   | Check | Command |
   |---|---|
   | Test | `$RUN_PREFIX pytest` |
   | Lint | `$RUN_PREFIX ruff check .` |
   | Lint fix | `$RUN_PREFIX ruff check . --fix` |
   | Format | `$RUN_PREFIX ruff format .` |
   | Format check | `$RUN_PREFIX ruff format . --check` |
   | Type check | `$RUN_PREFIX mypy .` |

6. Store all detected values as variables for use throughout the remaining phases.

---

## Phase 1: Read Plan

1. Get the cloudId using `mcp__atlassian__getAccessibleAtlassianResources`.
2. Fetch the Jira ticket using `mcp__atlassian__getJiraIssue` with `issueIdOrKey: TICKET_KEY`.
3. Scan comments for the `**IMPLEMENTATION PLAN**` comment.
4. Extract the section of the plan that pertains to this repo (match by repo name or URL).
5. This section is the plan for this repo — use it for all subsequent phases.
6. Transition the ticket to **In Progress**:
   - Get available transitions using `mcp__atlassian__getTransitionsForJiraIssue` with `cloudId` and `issueIdOrKey: TICKET_KEY`.
   - Find the transition whose `name` matches "In Progress" (case-insensitive).
   - Apply it using `mcp__atlassian__transitionJiraIssue` with `transition: { "id": "<transition-id>" }`.
   - If the ticket is already In Progress or the transition is not available, skip silently and continue.

---

## Phase 2: Create Worktree and Branch

1. Derive the branch name from the ticket: `TICKET_KEY` + abbreviated title in kebab-case.
   - Example: ticket `JAR-1234` with title "Add user validation" -> branch `JAR-1234-add-user-validation`

2. Ensure the repo is on the default branch and up to date:
   ```bash
   cd <REPO_PATH>
   git fetch origin <DEFAULT_BRANCH>
   git checkout <DEFAULT_BRANCH>
   git pull origin <DEFAULT_BRANCH>
   ```

3. Create the worktree:
   ```bash
   git worktree add ../<REPO_NAME>-<TICKET_KEY> -b <branch-name>
   ```

4. Install dependencies in the worktree:
   ```bash
   cd ../<REPO_NAME>-<TICKET_KEY>
   $INSTALL_CMD
   ```

5. All subsequent commands in Phases 3-8 must run **inside the worktree directory**.

---

## Phase 3: Implement Changes

Working from the worktree directory:

1. Follow the implementation plan step by step.

2. For each step:
   - Read existing files before modifying them
   - Follow existing code patterns and conventions
   - Write clean, focused code — no over-engineering
   - Write tests as specified in the plan

3. Use the Agent tool with sub-agents for complex implementation steps to keep context focused.

---

## Phase 4: Quality Checks

Working from the worktree directory. Run each check and fix failures before proceeding to the next.

### 4a. Run Tests
```bash
$RUN_PREFIX pytest
```
If tests fail, fix the implementation and re-run until all pass.

### 4b. Run Linting
```bash
$RUN_PREFIX ruff check .
```
If lint errors are found, run auto-fix first:
```bash
$RUN_PREFIX ruff check . --fix
```
If issues remain, fix manually. Re-run until clean.

### 4c. Run Formatting
```bash
$RUN_PREFIX ruff format . --check
```
If formatting issues are found, auto-fix:
```bash
$RUN_PREFIX ruff format .
```
Re-run the check to verify.

### 4d. Run Type Checking
```bash
$RUN_PREFIX mypy .
```
If type errors are found, fix them and re-run until clean.

---

## Phase 5: Commit and Push

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
   <TICKET_KEY>: Brief imperative description of changes

   Co-Authored-By: Claude AI
   EOF
   )"
   ```

4. Push the branch:
   ```bash
   git push -u origin <branch-name>
   ```

---

## Phase 6: Create MR

1. Compose the MR title: `<TICKET_KEY>: Brief imperative description` (under 70 characters).

2. Compose the MR description:
   ```markdown
   ## Summary

   - Key change 1
   - Key change 2
   - Key change 3

   Generated with [Claude Code](https://claude.com/claude-code)
   ```

3. Create the MR using MCP:
   ```
   mcp__gitlab__create_merge_request with:
     project_id: "<GITLAB_PROJECT_PATH_ENCODED>"
     source_branch: "<branch-name>"
     target_branch: "<DEFAULT_BRANCH>"
     title: "<mr-title>"
     description: "<mr-description>"
     remove_source_branch: true
     squash: true
   ```

   **Fallback** (only if MCP tool is unavailable):
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

4. Parse the response and extract:
   - `web_url` — the MR URL
   - `iid` — the merge request internal ID
   - `project_id` — the **numeric** project ID

   **Important:** Store the numeric `project_id` and use it for **all** subsequent GitLab API calls in Phase 7. Do NOT use the URL-encoded project path — it may return 404 depending on token permissions.

---

## Phase 7: Monitor Pipeline

**Note:** There are no MCP tools for pipeline monitoring. Use curl for all Phase 7 API calls.

**API base for all Phase 7 calls:** `https://<GITLAB_HOST>/api/v4/projects/<PROJECT_ID>` (using the numeric `project_id` from Phase 6).

### 7a. Wait for Pipeline to Start

1. Query the MR pipelines (no initial sleep — query immediately):
   ```bash
   curl --silent --header "PRIVATE-TOKEN: $GITLAB_TOKEN" \
     "https://<GITLAB_HOST>/api/v4/projects/<PROJECT_ID>/merge_requests/<iid>/pipelines"
   ```

2. Parse the full JSON response. If the response contains an error or is not a JSON array, log the raw response for debugging.

3. Extract the most recent pipeline ID (first item in the array).

4. If the array is empty, wait 10 seconds and retry. If no pipeline is found after 3 attempts, return failure details to caller.

### 7b. Monitor Pipeline Status

Set up a monitoring loop:
- Maximum monitoring time: 30 minutes
- Poll interval: 30 seconds
- Maximum auto-fix attempts: 3

For each poll cycle:

1. Query the pipeline status:
   ```bash
   curl --silent --header "PRIVATE-TOKEN: $GITLAB_TOKEN" \
     "https://<GITLAB_HOST>/api/v4/projects/<PROJECT_ID>/pipelines/<pipeline-id>"
   ```

2. Check the `status` field:
   - `pending` or `running` -> continue monitoring
   - `success` -> transition ticket to **Code Review** (see below), then return success with MR URL
   - `failed` -> proceed to 7c
   - `canceled`, `skipped`, `manual` -> return failure with details

   **On pipeline success**, transition the ticket to Code Review:
   - Get available transitions using `mcp__atlassian__getTransitionsForJiraIssue` with `cloudId` and `issueIdOrKey: TICKET_KEY`.
   - Find the transition whose `name` matches "Code Review" (case-insensitive).
   - Apply it using `mcp__atlassian__transitionJiraIssue` with `transition: { "id": "<transition-id>" }`.
   - If the transition is not available, skip silently and continue.

### 7c. Analyze Pipeline Failure

1. Query all jobs in the pipeline:
   ```bash
   curl --silent --header "PRIVATE-TOKEN: $GITLAB_TOKEN" \
     "https://<GITLAB_HOST>/api/v4/projects/<PROJECT_ID>/pipelines/<pipeline-id>/jobs"
   ```

2. Identify failed jobs (status = `failed`).

3. For each failed job, fetch the job trace:
   ```bash
   curl --silent --header "PRIVATE-TOKEN: $GITLAB_TOKEN" \
     "https://<GITLAB_HOST>/api/v4/projects/<PROJECT_ID>/jobs/<job-id>/trace"
   ```

4. Analyze each failed job trace to determine the type of failure and whether it is auto-fixable.

### 7d. Attempt Auto-Fix

If the failure appears auto-fixable and we haven't exceeded the maximum auto-fix attempts:

1. **Categorize the failure** and apply the appropriate fix strategy:

   **Test Failures (pytest):**
   - Read the test file and implementation, fix, verify with `$RUN_PREFIX pytest`

   **Linting Errors (ruff):**
   - Auto-fix: `$RUN_PREFIX ruff check . --fix`
   - Verify: `$RUN_PREFIX ruff check .`

   **Formatting Issues (ruff):**
   - Auto-fix: `$RUN_PREFIX ruff format .`
   - Verify: `$RUN_PREFIX ruff format . --check`

   **Type Errors (mypy):**
   - Read files with type errors, fix, verify with `$RUN_PREFIX mypy .`

   **Dependency Errors:**
   - Run `$INSTALL_CMD`, check for lock file conflicts

2. Verify the fix locally.

3. Commit and push the fix:
   ```bash
   git add <files>
   git commit -m "$(cat <<'EOF'
   <TICKET_KEY>: Fix pipeline failure - <brief description>

   Co-Authored-By: Claude AI
   EOF
   )"
   git push
   ```

4. Increment the auto-fix attempt counter.

5. Wait 10 seconds, query for the new pipeline ID, and return to 7b.

### 7e. Handle Non-Fixable Failures

If the failure is not auto-fixable or maximum attempts have been reached, return failure details to the caller including:
- Failed job names
- Error messages
- Why it couldn't be auto-fixed
- Pipeline URL: `https://<GITLAB_HOST>/<GITLAB_PROJECT_PATH>/-/pipelines/<pipeline-id>`

---

## Phase 8: Cleanup

1. Return to the project root:
   ```bash
   cd <PROJECT_ROOT>
   ```

2. Return the result to the caller:
   - **Success:** MR URL (`web_url`)
   - **Failure:** error details, what was tried, pipeline URL
