---
name: implement-ios
description: Execute iOS (Swift/SwiftUI) implementation for a mobile repo. Creates a worktree, implements changes from the Jira plan, runs SwiftLint and xcodebuild tests, commits, and creates a GitLab MR.
argument-hint: ticket-key repo-path
user-invokable: true
---

# Implement iOS: $ARGUMENTS

Execute the iOS implementation for a mobile repository based on the implementation plan from a Jira ticket.

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

4. **iOS project layout:**
   ```bash
   # Find the Swift/iOS source directory
   if [ -d "$PROJECT_ROOT/swift" ]; then
     IOS_ROOT="$PROJECT_ROOT/swift"
   elif [ -d "$PROJECT_ROOT/ios" ]; then
     IOS_ROOT="$PROJECT_ROOT/ios"
   else
     IOS_ROOT="$PROJECT_ROOT"
   fi

   # Find Xcode project
   XCODEPROJ=$(find "$IOS_ROOT" -maxdepth 2 -name "*.xcodeproj" ! -path "*/.build/*" | head -1)
   XCWORKSPACE=$(find "$IOS_ROOT" -maxdepth 2 -name "*.xcworkspace" ! -path "*/xcuserdata/*" ! -path "*/.build/*" | head -1)

   if [ -n "$XCWORKSPACE" ]; then
     BUILD_TARGET="-workspace $XCWORKSPACE"
   elif [ -n "$XCODEPROJ" ]; then
     BUILD_TARGET="-project $XCODEPROJ"
   fi

   echo "IOS_ROOT=$IOS_ROOT"
   echo "BUILD_TARGET=$BUILD_TARGET"
   ```

5. **Detect test scheme and test target:**
   ```bash
   # List schemes
   xcodebuild -list $BUILD_TARGET 2>/dev/null | grep -A 50 "Schemes:"

   # Prefer "CI Test" scheme for testing, fall back to "Test", then main scheme
   # Detect test target name (e.g., InsightsTests)
   ```

6. **Detect SwiftLint and Fastlane:**
   ```bash
   HAS_SWIFTLINT=false
   if [ -f "$PROJECT_ROOT/swiftlint.yml" ] || [ -f "$PROJECT_ROOT/.swiftlint.yml" ] || [ -f "$IOS_ROOT/swiftlint.yml" ] || command -v swiftlint &>/dev/null; then
     HAS_SWIFTLINT=true
   fi

   HAS_FASTLANE=false
   if [ -d "$PROJECT_ROOT/fastlane" ]; then
     HAS_FASTLANE=true
   fi

   echo "HAS_SWIFTLINT=$HAS_SWIFTLINT"
   echo "HAS_FASTLANE=$HAS_FASTLANE"
   ```

7. **Read skill files:** If a `skills/` directory exists, read relevant skill files before writing any code:
   - Always read `skills/pfw-pfw/SKILL.md` first (if it exists)
   - Read architecture-specific skills (TCA, data sync, Mapbox, etc.)
   - Read `CLAUDE.md` for project conventions

8. Store all detected values as variables for use throughout the remaining phases.

---

## Phase 1: Read Plan

1. Get the cloudId using `mcp__atlassian__getAccessibleAtlassianResources`.
2. Fetch the Jira ticket using `mcp__atlassian__getJiraIssue` with `issueIdOrKey: TICKET_KEY`.
3. Scan comments for the `**IMPLEMENTATION PLAN**` comment.
4. Extract the section of the plan that pertains to **iOS** (match by "iOS", "Swift", or "swift/" path references).
5. This section is the plan for iOS — use it for all subsequent phases.
6. Transition the ticket to **In Progress**:
   - Get available transitions using `mcp__atlassian__getTransitionsForJiraIssue` with `cloudId` and `issueIdOrKey: TICKET_KEY`.
   - Find the transition whose `name` matches "In Progress" (case-insensitive).
   - Apply it using `mcp__atlassian__transitionJiraIssue` with `transition: { "id": "<transition-id>" }`.
   - If the ticket is already In Progress or the transition is not available, skip silently and continue.

---

## Phase 2: Create Worktree and Branch

1. Derive the branch name from the ticket: `TICKET_KEY` + abbreviated title in kebab-case.
   - Example: ticket `JAR-1234` with title "Add field boundary editing" -> branch `JAR-1234-add-field-boundary-editing`

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

4. Resolve SPM dependencies in the worktree (if Xcode project):
   ```bash
   cd ../<REPO_NAME>-<TICKET_KEY>
   xcodebuild -resolvePackageDependencies $BUILD_TARGET -skipMacroValidation 2>&1 | tail -5
   ```

5. All subsequent commands in Phases 3-8 must run **inside the worktree directory**.

---

## Phase 3: Implement Changes

Working from the worktree directory:

1. **Read skill files first** — before writing any code, read the relevant skill files from the `skills/` directory. For TCA code, always read `pfw` + `pfw-composable-architecture`. Check `CLAUDE.md` for the correct skill combos.

2. Follow the implementation plan step by step.

3. For each step:
   - Read existing files before modifying them
   - Follow existing code patterns and conventions
   - Use TCA patterns for iOS (if the project uses TCA): `@Reducer`, `@ObservableState`, `@Dependency`
   - Write tests as specified in the plan
   - Never improvise when a skill file exists — read and follow it

4. Use the Agent tool with sub-agents for complex implementation steps to keep context focused.

---

## Phase 4: Quality Checks

Working from the worktree directory. Run each check and fix failures before proceeding to the next.

### 4a. Run SwiftLint

Skip if `HAS_SWIFTLINT` is false.

If Fastlane is available:
```bash
fastlane ios lint
```

Otherwise:
```bash
cd <IOS_ROOT>
swiftlint lint
```

If errors are found, try auto-fix first (`swiftlint lint --fix`), then fix remaining issues manually.

### 4b. Build the Project

```bash
xcodebuild build-for-testing \
  $BUILD_TARGET \
  -scheme "<TEST_SCHEME>" \
  -destination "platform=iOS Simulator,name=iPhone 17" \
  -skipMacroValidation \
  -skipPackagePluginValidation \
  | tail -20
```

If build fails, fix compilation errors and re-run until the build succeeds.

### 4c. Run Main App Unit Tests

```bash
xcodebuild test \
  $BUILD_TARGET \
  -scheme "<TEST_SCHEME>" \
  -destination "platform=iOS Simulator,name=iPhone 17" \
  -skipMacroValidation \
  -skipPackagePluginValidation \
  -only-testing:<TestTarget> \
  | tail -50
```

If tests fail, fix the implementation and re-run until all pass.

### 4d. Run Affected Library Tests

**IMPORTANT:** If any changes were made to files under `swift_libraries/`, you MUST also run tests for each affected library from the library's own package directory. Library tests cannot be run from the parent Xcode project.

1. Identify affected libraries:
   ```bash
   git diff --name-only <DEFAULT_BRANCH>...HEAD | grep "^swift_libraries/" | cut -d/ -f2 | sort -u
   ```

2. For each affected library, run its tests:
   ```bash
   cd <WORKTREE>/swift_libraries/<library-name>
   xcodebuild test \
     -scheme "<LibraryScheme>" \
     -destination "platform=iOS Simulator,name=iPhone 17" \
     -skipMacroValidation \
     -quiet \
     2>&1 | tail -30
   ```

   The scheme name typically matches the directory name in PascalCase (e.g., `granular-off` → `GranularOff`).

3. All library test suites must pass before proceeding.

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

   **Important:**
   - Only stage files under `swift/`, `swift_libraries/`, or other iOS-relevant directories. Do not stage Android files.
   - **Never stage or commit `.xcconfig` files** (e.g., `Local-Prod.xcconfig`, `Local.xcconfig`). These are local environment configuration. If any are pre-staged, unstage them with `git restore --staged <file>`.

3. Commit with the ticket number prefix:
   ```bash
   git commit -m "$(cat <<'EOF'
   <TICKET_KEY>: Brief imperative description of iOS changes

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

   - Key iOS change 1
   - Key iOS change 2
   - Key iOS change 3

   ## Platform
   iOS (Swift/SwiftUI)

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
