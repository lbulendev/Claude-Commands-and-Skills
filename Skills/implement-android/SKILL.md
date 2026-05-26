---
name: implement-android
description: Execute Android (Kotlin/Jetpack Compose) implementation for a mobile repo. Creates a worktree, implements changes from the Jira plan, runs ktlint, Gradle tests, and Android lint, commits, and creates a GitLab MR.
argument-hint: ticket-key repo-path
user-invokable: true
---

# Implement Android: $ARGUMENTS

Execute the Android implementation for a mobile repository based on the implementation plan from a Jira ticket.

**Arguments:** `$ARGUMENTS` = `<ticket-key> <repo-path>`

Parse `$ARGUMENTS` to extract:
- `TICKET_KEY` — the first token (e.g., `JAR-1234`)
- `REPO_PATH` — the second token (e.g., `/tmp/repos/insights`)

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

4. **Android project layout:**
   ```bash
   # Find the Android root
   if [ -d "$PROJECT_ROOT/android" ] && ([ -f "$PROJECT_ROOT/android/settings.gradle.kts" ] || [ -f "$PROJECT_ROOT/android/settings.gradle" ]); then
     ANDROID_ROOT="$PROJECT_ROOT/android"
   elif [ -f "$PROJECT_ROOT/settings.gradle.kts" ] || [ -f "$PROJECT_ROOT/settings.gradle" ]; then
     ANDROID_ROOT="$PROJECT_ROOT"
   else
     echo "ERROR: No Android Gradle project found"; exit 1
   fi

   # Detect Gradle wrapper
   if [ -f "$ANDROID_ROOT/gradlew" ]; then
     GRADLE_CMD="$ANDROID_ROOT/gradlew"
   else
     GRADLE_CMD="gradle"
   fi

   echo "ANDROID_ROOT=$ANDROID_ROOT"
   echo "GRADLE_CMD=$GRADLE_CMD"
   ```

5. **Detect modules and test tasks:**
   ```bash
   cd "$ANDROID_ROOT"
   # List all modules
   $GRADLE_CMD projects --quiet 2>/dev/null | grep "---" | sed "s/.*--- Project '\(.*\)'/\1/"

   # Detect build flavors from app/build.gradle.kts
   # Common test tasks for multi-module Android projects:
   # :app:testGtestDebugUnitTest (app with gtest flavor)
   # :app:testDebugUnitTest (app without flavors)
   # :<module>:testDebugUnitTest (library modules)
   ```

6. **Detect Fastlane and version catalog:**
   ```bash
   HAS_FASTLANE=false
   if [ -d "$PROJECT_ROOT/fastlane" ]; then
     HAS_FASTLANE=true
   fi

   HAS_VERSION_CATALOG=false
   if [ -f "$ANDROID_ROOT/gradle/libs.versions.toml" ]; then
     HAS_VERSION_CATALOG=true
   fi

   echo "HAS_FASTLANE=$HAS_FASTLANE"
   echo "HAS_VERSION_CATALOG=$HAS_VERSION_CATALOG"
   ```

7. **Read skill files:** If a `skills/` directory exists under the Android directory (e.g., `android/skills/`), read relevant skill files before writing any code:
   - Jetpack Compose patterns
   - Mapbox Android patterns
   - Architecture guidelines

8. Store all detected values as variables for use throughout the remaining phases.

---

## Phase 1: Read Plan

1. Get the cloudId using `mcp__atlassian__getAccessibleAtlassianResources`.
2. Fetch the Jira ticket using `mcp__atlassian__getJiraIssue` with `issueIdOrKey: TICKET_KEY`.
3. Scan comments for the `**IMPLEMENTATION PLAN**` comment.
4. Extract the section of the plan that pertains to **Android** (match by "Android", "Kotlin", or "android/" path references).
5. This section is the plan for Android — use it for all subsequent phases.
6. Transition the ticket to **In Progress**:
   - Get available transitions using `mcp__atlassian__getTransitionsForJiraIssue` with `cloudId` and `issueIdOrKey: TICKET_KEY`.
   - Find the transition whose `name` matches "In Progress" (case-insensitive).
   - Apply it using `mcp__atlassian__transitionJiraIssue` with `transition: { "id": "<transition-id>" }`.
   - If the ticket is already In Progress or the transition is not available, skip silently and continue.

---

## Phase 2: Create Worktree and Branch

1. Derive the branch name from the ticket: `TICKET_KEY` + abbreviated title in kebab-case.
   - Example: ticket `JAR-1234` with title "Add offline sync indicator" -> branch `JAR-1234-add-offline-sync-indicator`

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

4. Sync Gradle in the worktree:
   ```bash
   cd ../<REPO_NAME>-<TICKET_KEY>/android
   $GRADLE_CMD dependencies --quiet 2>&1 | tail -5
   ```

5. All subsequent commands in Phases 3-8 must run **inside the worktree directory**.

---

## Phase 3: Implement Changes

Working from the worktree directory:

1. **Read skill files first** — before writing any code, read any relevant skill files from `android/skills/` (e.g., Jetpack Compose expert, Mapbox Android patterns).

2. Follow the implementation plan step by step.

3. For each step:
   - Read existing files before modifying them
   - Follow existing code patterns and conventions
   - Use MVVM/MVI patterns: `@HiltViewModel`, `ViewModel`, Compose State
   - Use Hilt for dependency injection: `@Module`, `@InstallIn`, `@Provides`
   - Use the version catalog (`libs.versions.toml`) for adding dependencies
   - Write tests as specified in the plan (JUnit 4, MockK, coroutines test)

4. Use the Agent tool with sub-agents for complex implementation steps to keep context focused.

---

## Phase 4: Quality Checks

Working from the worktree directory. Run each check and fix failures before proceeding to the next.

### 4a. Run ktlint

```bash
cd <ANDROID_ROOT>
$GRADLE_CMD ktlintCheck --build-cache
```

If errors are found, try auto-fix first:
```bash
$GRADLE_CMD ktlintFormat --build-cache
```
If issues remain, fix manually. Re-run until clean.

### 4b. Compile the Project

```bash
cd <ANDROID_ROOT>
$GRADLE_CMD compileAll --build-cache --stacktrace
```

If build fails, fix compilation errors and re-run until the build succeeds.

### 4c. Run Android Lint

```bash
cd <ANDROID_ROOT>
$GRADLE_CMD :app:lint --build-cache --stacktrace
```

Fix any new lint violations beyond the baseline. Re-run until clean.

### 4d. Run Unit Tests

Run tests per module:
```bash
cd <ANDROID_ROOT>
$GRADLE_CMD testDebugUnitTest --stacktrace --build-cache --continue
```

For multi-module projects with flavors (like insights), run each module's tests:
```bash
$GRADLE_CMD :app:testGtestDebugUnitTest --build-cache --continue
$GRADLE_CMD :granular-platform:testDebugUnitTest --build-cache --continue
$GRADLE_CMD :granular-off:testDebugUnitTest --build-cache --continue
$GRADLE_CMD :granular-fielddata:testDebugUnitTest --build-cache --continue
$GRADLE_CMD :granular-mop:testDebugUnitTest --build-cache --continue
$GRADLE_CMD :granular-notes:testDebugUnitTest --build-cache --continue
```

If tests fail, fix the implementation and re-run until all pass.

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

   **Important:** Only stage files under `android/`, `ktlint/`, or other Android-relevant directories. Do not stage iOS files.

3. Commit with the ticket number prefix:
   ```bash
   git commit -m "$(cat <<'EOF'
   <TICKET_KEY>: Brief imperative description of Android changes

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

   - Key Android change 1
   - Key Android change 2
   - Key Android change 3

   ## Platform
   Android (Kotlin/Jetpack Compose)

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

---

## Phase 7: Transition Ticket and Cleanup

1. Transition the ticket to **Code Review**:
   - Get available transitions using `mcp__atlassian__getTransitionsForJiraIssue` with `cloudId` and `issueIdOrKey: TICKET_KEY`.
   - Find the transition whose `name` matches "Code Review" (case-insensitive).
   - Apply it using `mcp__atlassian__transitionJiraIssue`. If the transition is not available, skip silently.

2. Return to the project root:
   ```bash
   cd <PROJECT_ROOT>
   ```

3. Return the result to the caller:
   - **Success:** MR URL (`web_url`)
