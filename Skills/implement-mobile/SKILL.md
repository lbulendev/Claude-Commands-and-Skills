---
name: implement-mobile
description: Implement a Jira ticket for a dual-platform mobile app (iOS + Android). Reads the ticket, creates a plan covering both platforms, implements in a worktree, runs platform-specific quality checks, and creates a GitLab MR.
argument-hint: [ticket-key]
user-invokable: true
---

# Implement Mobile Ticket $ARGUMENTS

Execute the full implementation workflow for Jira ticket **$ARGUMENTS** in a dual-platform mobile repository (iOS Swift + Android Kotlin).

---

## Phase 0: Runtime Detection

Before starting any work, detect the project environment:

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
   GITLAB_HOST=$(echo "$REMOTE_URL" | sed -E 's#^(https?://|git@)([^:/]+)[:/].*#\2#')
   GITLAB_PROJECT_PATH=$(echo "$REMOTE_URL" | sed -E 's#^(https?://|git@)[^:/]+[:/](.+?)(\.git)?$#\2#')
   GITLAB_PROJECT_PATH_ENCODED=$(echo "$GITLAB_PROJECT_PATH" | sed 's/\//%2F/g')
   echo "GITLAB_HOST=$GITLAB_HOST"
   echo "GITLAB_PROJECT_PATH=$GITLAB_PROJECT_PATH"
   echo "GITLAB_PROJECT_PATH_ENCODED=$GITLAB_PROJECT_PATH_ENCODED"
   ```

4. **Detect platforms present:**
   ```bash
   HAS_IOS=false
   HAS_ANDROID=false

   # iOS detection
   if [ -d "$PROJECT_ROOT/swift" ] || [ -d "$PROJECT_ROOT/ios" ]; then
     HAS_IOS=true
     if [ -d "$PROJECT_ROOT/swift" ]; then
       IOS_ROOT="$PROJECT_ROOT/swift"
     else
       IOS_ROOT="$PROJECT_ROOT/ios"
     fi
   fi

   # Android detection
   if [ -d "$PROJECT_ROOT/android" ] && ([ -f "$PROJECT_ROOT/android/settings.gradle.kts" ] || [ -f "$PROJECT_ROOT/android/settings.gradle" ]); then
     HAS_ANDROID=true
     ANDROID_ROOT="$PROJECT_ROOT/android"
   fi

   echo "HAS_IOS=$HAS_IOS IOS_ROOT=$IOS_ROOT"
   echo "HAS_ANDROID=$HAS_ANDROID ANDROID_ROOT=$ANDROID_ROOT"
   ```

5. **Read project conventions:**
   - Read `CLAUDE.md` if it exists
   - Read `AGENTS.md` if it exists
   - List available skills in `skills/` directory

6. Display a summary to the user for confirmation before proceeding.

---

## Phase 1: Read the Jira Ticket

1. Validate that `$ARGUMENTS` matches the format `ABC-1234`. If not, inform the user and **stop immediately**.

2. Get the cloudId first using `mcp__atlassian__getAccessibleAtlassianResources`. Use the first result's `id` as the `cloudId`.

3. Use `mcp__atlassian__getJiraIssue` to fetch ticket `$ARGUMENTS`:
   - Required parameters:
     - `cloudId`: the cloudId from step 2
     - `issueIdOrKey`: `$ARGUMENTS`

4. If the ticket is not found, inform the user and **stop immediately**.

5. Extract from the response:
   - **Title**: `fields.summary`
   - **Description**: `fields.description`
   - **Comments**: fetch with `expand: "renderedFields"` to get all comments

6. Scan comments for:
   - A comment starting with `**IMPLEMENTATION PLAN**` -> store as `existing_plan`
   - A comment starting with `**IMPLEMENTATION DECISIONS**` -> store as `existing_decisions`

---

## Phase 2: Implementation Planning

### If an existing plan was found:
Display the existing plan to the user and confirm they want to proceed with it. Skip to **Phase 3**.

### If no existing plan was found:

#### 2a. Incorporate Decisions
If `existing_decisions` was found, use those Q&A pairs to supplement the description. **Prefer decisions over description** when they conflict.

#### 2b. Determine Affected Platforms

Determine the target platform(s) from the **ticket title**:
- If the title contains **"iOS"** (case-insensitive) but **not** "Android" → **iOS only**
- If the title contains **"Android"** (case-insensitive) but **not** "iOS" → **Android only**
- If the title contains **both** "iOS" and "Android", or **neither** → **Both platforms**

Store the result as `TARGET_PLATFORMS` (one of: `ios`, `android`, `both`).

Only implement, quality-check, and test for the target platform(s). Skip all phases and steps for the other platform.

#### 2c. Explore the Codebase

Use the Agent tool with `subagent_type=Explore` to understand the relevant parts of the codebase. **Only explore the target platform(s).** Read skill files first.

**If `TARGET_PLATFORMS` is `ios` or `both`:**
- Always read `skills/pfw-pfw/SKILL.md` first
- Read relevant skill combos from `CLAUDE.md` (e.g., `pfw-composable-architecture` + `pfw-dependencies` + `pfw-testing` for new features)
- Identify existing TCA features, reducers, and views
- Find related tests and test patterns

**If `TARGET_PLATFORMS` is `android` or `both`:**
- Read `android/skills/` if present (Jetpack Compose patterns, Mapbox patterns)
- Identify existing ViewModels, Composables, and repositories
- Understand Hilt DI setup
- Find related tests and test patterns

#### 2d. Ask Clarifying Questions

Based on the description, decisions (if any), and codebase exploration:
- Identify ambiguities or gaps in the requirements
- Ask about platform-specific behavior differences (if any)
- Ask about LaunchDarkly feature flags if relevant
- Ask about offline behavior and data sync implications
- Use the AskUserQuestion tool to ask the user interactively
- Collect all answers

#### 2e. Post Implementation Decisions to Jira

If any clarifying questions were asked, post a comment using `mcp__atlassian__addCommentToJiraIssue`:

```
**IMPLEMENTATION DECISIONS**

Q: [question]
A: [answer]
```

If NO questions were asked, skip this step entirely.

#### 2f. Build the Implementation Plan

Create a plan that **only includes sections for the target platform(s)** based on `TARGET_PLATFORMS`:

1. **Summary** — Brief overview of what will be implemented, stating which platform(s) are targeted
2. **iOS Changes** (only if `TARGET_PLATFORMS` is `ios` or `both`):
   - Files to modify/create (under `swift/` or `swift_libraries/`)
   - TCA reducer/view/dependency changes
   - Implementation steps
   - Tests (using Point-Free testing patterns)
3. **Android Changes** (only if `TARGET_PLATFORMS` is `android` or `both`):
   - Files to modify/create (under `android/`)
   - ViewModel/Composable/Repository changes
   - Implementation steps
   - Tests (JUnit 4, MockK, coroutines test)
4. **Shared Considerations** (only if `TARGET_PLATFORMS` is `both`):
   - API contracts / data models that must match across platforms
   - Feature flag coordination
   - Offline/sync behavior
5. **Risks and Considerations**

#### 2g. Get User Approval

Present the plan to the user. Iterate based on their feedback until approved.

#### 2h. Post Plan to Jira

Once approved, post the plan as a comment using `mcp__atlassian__addCommentToJiraIssue`:

```
**IMPLEMENTATION PLAN**

[full plan content with iOS and Android sections]
```

---

## Phase 3: Create Worktree and Branch

1. Derive the branch name from the ticket: `$ARGUMENTS` key + abbreviated title in kebab-case.
   - Example: ticket `JAR-1234` with title "Add field boundary editing" -> branch `JAR-1234-add-field-boundary-editing`

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

4. Resolve dependencies in the worktree (only for target platform(s)):

   **iOS** (only if `TARGET_PLATFORMS` is `ios` or `both`):
   ```bash
   cd ../<REPO_NAME>-$ARGUMENTS/swift
   xcodebuild -resolvePackageDependencies -project *.xcodeproj -skipMacroValidation 2>&1 | tail -5
   ```

   **Android** (only if `TARGET_PLATFORMS` is `android` or `both`):
   ```bash
   cd ../<REPO_NAME>-$ARGUMENTS/android
   ./gradlew dependencies --quiet 2>&1 | tail -5
   ```

5. All subsequent commands in Phases 4-8 must run **inside the worktree directory**.

---

## Phase 4: Implement the Changes

Working from the worktree directory (`../<REPO_NAME>-$ARGUMENTS`):

### 4a. iOS Implementation (only if `TARGET_PLATFORMS` is `ios` or `both`)

Skip this sub-phase entirely if `TARGET_PLATFORMS` is `android`.

1. **Read skill files first** — always read `pfw` + relevant skills before writing code.

2. Follow the iOS section of the implementation plan step by step.

3. For each step:
   - Read existing files before modifying them
   - Follow TCA patterns: `@Reducer`, `@ObservableState`, `@Dependency`
   - Use Point-Free libraries — never improvise when a skill exists
   - Write tests using `TestStore` and Point-Free testing patterns

4. Use the Agent tool with sub-agents for complex implementation steps.

### 4b. Android Implementation (only if `TARGET_PLATFORMS` is `android` or `both`)

Skip this sub-phase entirely if `TARGET_PLATFORMS` is `ios`.

1. **Read skill files first** — read `android/skills/` if present.

2. Follow the Android section of the implementation plan step by step.

3. For each step:
   - Read existing files before modifying them
   - Follow MVVM patterns: `@HiltViewModel`, `ViewModel`, Compose State
   - Use Hilt for DI, version catalog for dependencies
   - Write tests using JUnit 4, MockK, coroutines test

4. Use the Agent tool with sub-agents for complex implementation steps.

---

## Phase 5: Quality Checks

Working from the worktree directory. **Only run checks for the target platform(s).**

### 5a. iOS Quality Checks (only if `TARGET_PLATFORMS` is `ios` or `both`)

Skip this sub-phase entirely if `TARGET_PLATFORMS` is `android`.

**SwiftLint:**
```bash
fastlane ios lint
```
Or: `swiftlint lint` from the Swift source directory. Fix errors with `swiftlint lint --fix`.

**Build:**
```bash
cd swift
xcodebuild build-for-testing \
  -project *.xcodeproj \
  -scheme "CI Test" \
  -destination "platform=iOS Simulator,name=iPhone 17" \
  -skipMacroValidation -skipPackagePluginValidation \
  | tail -20
```

**Main App Tests:**
```bash
xcodebuild test \
  -project *.xcodeproj \
  -scheme "CI Test" \
  -destination "platform=iOS Simulator,name=iPhone 17" \
  -skipMacroValidation -skipPackagePluginValidation \
  -only-testing:InsightsTests \
  | tail -50
```

**Affected Library Tests:**

**IMPORTANT:** If any changes were made to files under `swift_libraries/`, you MUST also run tests for each affected library from the library's own package directory. Library tests cannot be run from the parent Xcode project.

1. Identify affected libraries:
   ```bash
   git diff --name-only <DEFAULT_BRANCH>...HEAD | grep "^swift_libraries/" | cut -d/ -f2 | sort -u
   ```

2. For each affected library:
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

### 5b. Android Quality Checks (only if `TARGET_PLATFORMS` is `android` or `both`)

Skip this sub-phase entirely if `TARGET_PLATFORMS` is `ios`.

**ktlint:**
```bash
cd android
./gradlew ktlintCheck --build-cache
```
Fix errors with `./gradlew ktlintFormat --build-cache`.

**Compile:**
```bash
./gradlew compileAll --build-cache --stacktrace
```

**Android Lint:**
```bash
./gradlew :app:lint --build-cache --stacktrace
```

**Tests:**
```bash
./gradlew testDebugUnitTest --stacktrace --build-cache --continue
```

Fix all failures before proceeding.

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

   Stage files from both platforms as appropriate. Do **not** stage unrelated files, secrets, generated artifacts, or **`.xcconfig` files** (local environment configuration). If any `.xcconfig` files are pre-staged, unstage them with `git restore --staged <file>`.

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

2. Compose the MR description:
   ```markdown
   ## Summary

   - Key change 1
   - Key change 2
   - Key change 3

   ## Platforms
   - [x] iOS (Swift/SwiftUI)
   - [x] Android (Kotlin/Jetpack Compose)

   Generated with [Claude Code](https://claude.com/claude-code)
   ```

   Adjust the platform checkboxes based on what was actually changed.

3. Create the MR:
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
   - `web_url` — the MR URL to display

---

## Phase 8: Cleanup

1. Return to the main repository directory:
   ```bash
   cd <PROJECT_ROOT>
   ```

2. Inform the user that the workflow is complete with:
   - Jira ticket key
   - Branch name
   - MR URL
   - Pipeline status (success/failed/monitoring stopped)
   - Platforms implemented (iOS/Android/both)
   - Number of auto-fix attempts made (if any)
   - Summary of what was implemented
   - Worktree location (in case they need to make manual adjustments)
