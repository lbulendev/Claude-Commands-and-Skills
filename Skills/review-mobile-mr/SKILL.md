---
name: review-mobile-mr
description: Review a GitLab merge request for a mobile project. Takes a Jira ticket key, fetches the ticket, finds the associated MR in GitLab, explores the codebase for context, asks the user about any requirement gaps, then reviews the code and lets the user choose which comments to post.
argument-hint: [TICKET-KEY]
user-invokable: true
---

# Review Mobile MR

Review a GitLab merge request for a mobile project (iOS Swift / Android Kotlin).

**Phases are sequential gates — do not begin Phase N+1 until Phase N succeeds.**

---

## Phase 0: Validate the Ticket Key

1. `$ARGUMENTS` must match the format `ABC-1234` (project prefix, hyphen, number). If invalid or empty, use the AskUserQuestion tool to prompt:

   > What is the Jira ticket key for the MR you'd like me to review? (e.g., MG-10417)

---

## Phase 1: Fetch the Jira Ticket

**IMPORTANT: Always use `mcp__atlassian__*` tools for ALL Jira operations. Never use curl or direct REST API calls for Jira.**

1. Use the known cloudId `364882d6-7651-4070-a7c3-c4590783af13` (corteva-research). Do NOT call `mcp__atlassian__getAccessibleAtlassianResources` — use this cloudId directly.
2. Fetch the ticket using `mcp__atlassian__getJiraIssue` with `cloudId: "364882d6-7651-4070-a7c3-c4590783af13"`, `issueIdOrKey: <TICKET_KEY>`, `expand: "renderedFields"`, `fields: ["summary", "description"]`, and `responseContentFormat: "markdown"`.

3. **If the ticket fetch fails** (any error — MCP error, 404, timeout, user-cancel, etc.):
   - Inform the user: "Could not fetch Jira ticket <TICKET_KEY>: <error message>. Unable to proceed with requirement-aware review without ticket context."
   - **Stop immediately.** Do not continue to Phase 2 or beyond.

4. Extract:
   - **Title**: `fields.summary`
   - **Description**: `fields.description`

5. Build the **ticket context**: combine the title and description into a unified understanding of what the ticket requires.

---

## Phase 2: Find the MR in GitLab

1. Search for the merge request using `mcp__corteva__gitlab-seed__list_merge_requests` with:
   ```
   search: "<TICKET_KEY>"
   state: "opened"
   scope: "all"
   ```

2. If no results are found with `state: "opened"`, retry with `state: "all"` to include merged/closed MRs.

3. If multiple MRs are found, present the list to the user using AskUserQuestion and let them pick which one to review.

4. If no MRs are found at all, inform the user and stop.

5. From the selected MR, extract:
   - `project_id` — the numeric project ID
   - `iid` — the MR IID
   - `title` — MR title
   - `description` — MR description
   - `source_branch` — the branch with changes
   - `target_branch` — the branch being merged into
   - `diff_refs.base_sha`, `diff_refs.head_sha`, `diff_refs.start_sha` — needed for posting inline comments later
   - `web_url` — the MR URL

6. Parse `web_url` to extract:
   - `GITLAB_HOST` — the hostname (e.g., `gitlab.research.corteva.com`)
   - `GITLAB_PROJECT_PATH` — the full project path (e.g., `granular/mobile/insights-app/insights`)
   - `GITLAB_PROJECT_PATH_ENCODED` — URL-encoded project path (slashes → `%2F`)

---

## Phase 3: Fetch MR Diffs

1. List all changed files using `mcp__corteva__gitlab-seed__list_merge_request_changed_files`:
   ```
   project_id: "<GITLAB_PROJECT_PATH_ENCODED>"
   merge_request_iid: "<MR_IID>"
   ```

2. Fetch the full diffs using `mcp__corteva__gitlab-seed__get_merge_request_diffs`:
   ```
   project_id: "<GITLAB_PROJECT_PATH_ENCODED>"
   merge_request_iid: "<MR_IID>"
   ```

   Store the complete diff output — this is the primary material for the review.

---

## Phase 4: Clone the Repo

1. Clone the repo if not already cloned:
   ```bash
   CLONE_DIR="/tmp/claude-review-<MR_IID>/$(basename <repo-url> .git)"
   if [ ! -d "$CLONE_DIR" ]; then
     git clone --depth=100 <repo-ssh-url> "$CLONE_DIR"
   fi
   ```

   Derive the SSH URL from `GITLAB_HOST` and `GITLAB_PROJECT_PATH`:
   ```
   git@<GITLAB_HOST>:<GITLAB_PROJECT_PATH>.git
   ```

2. Fetch and checkout the MR source branch:
   ```bash
   cd "$CLONE_DIR"
   git fetch origin <source_branch>
   git checkout <source_branch>
   ```

---

## Phase 5: Explore the Codebase

1. Detect the project type:
   ```bash
   HAS_IOS=false; HAS_ANDROID=false
   if [ -d "$CLONE_DIR/swift" ] || [ -d "$CLONE_DIR/ios" ]; then HAS_IOS=true; fi
   if [ -d "$CLONE_DIR/android" ] && ([ -f "$CLONE_DIR/android/settings.gradle.kts" ] || [ -f "$CLONE_DIR/android/settings.gradle" ]); then HAS_ANDROID=true; fi
   ```

2. Read project conventions:
   - Read `CLAUDE.md` if it exists at the repo root
   - Read `AGENTS.md` if it exists at the repo root

3. Determine which platform(s) the MR changes target by examining the changed files:
   - Files under `swift/` or `swift_libraries/` → iOS
   - Files under `android/` → Android
   - Both → both

4. Read relevant skill files for the target platform(s):

   **iOS:**
   - Read `skills/pfw-pfw/SKILL.md` first (if it exists)
   - Read relevant `pfw-*` skill combos referenced in `CLAUDE.md`

   **Android:**
   - Read `android/skills/` directory if present

5. Use the Agent tool with `subagent_type=Explore` to understand the areas of the codebase touched by the MR:
   - For each changed file, understand its role in the architecture
   - Identify the patterns and conventions used in surrounding code
   - Read related test files
   - Understand the feature area being modified

---

## Phase 6: Identify Requirement Gaps

Compare the **ticket context** (from Phase 1) against the **MR diff** (from Phase 3):

1. Check whether the MR addresses all acceptance criteria in the ticket description.
2. Check whether the MR matches the implementation plan (if one exists).
3. Identify any gaps:
   - Requirements from the ticket that are not addressed in the MR
   - Changes in the MR that go beyond the ticket scope (unexpected additions)
   - Implementation decisions that were made differently than planned
   - Edge cases from the ticket that may not be handled

4. If there are gaps or questions, present them to the user using AskUserQuestion:
   - Frame each gap clearly: what the ticket says vs. what the MR does
   - Ask whether the gap is intentional, a known limitation, or needs to be addressed

5. Collect the user's responses. These inform the review — gaps the user marks as intentional should not be flagged as issues in the review.

---

## Phase 7: Code Review

Perform a thorough code review of the MR diffs, informed by:
- The ticket requirements and implementation plan
- The user's responses about requirement gaps
- The project conventions from CLAUDE.md, AGENTS.md, and skill files
- The codebase patterns observed during exploration

### Review Checklist

**For all platforms:**
- **Correctness**: Does the code do what the ticket requires?
- **Completeness**: Are all acceptance criteria covered?
- **Naming**: Do names follow project conventions? Are they descriptive?
- **Error handling**: Are failure cases handled appropriately?
- **Edge cases**: Are boundary conditions considered?
- **Code duplication**: Is there unnecessary duplication?
- **Unnecessary changes**: Are there unrelated modifications mixed in?
- **Tests**: Are there tests? Do they cover the important cases? Are negative/edge cases tested?

**iOS-specific (if applicable):**
- **TCA patterns**: Correct use of `@Reducer`, `@ObservableState`, `@Dependency`, `Effect`, `State`, `Action`
- **SwiftUI best practices**: Proper use of property wrappers, view composition, performance
- **Point-Free conventions**: TestStore usage, exhaustive testing, dependency management
- **Memory management**: No retain cycles, proper use of `[weak self]` where needed
- **Thread safety**: Main actor annotations, proper async/await usage

**Android-specific (if applicable):**
- **MVVM/MVI patterns**: Correct ViewModel state management, Compose state hoisting
- **Hilt DI**: Proper module setup, scoping, `@Provides` vs `@Binds`
- **Jetpack Compose**: Recomposition safety, stable types, proper key usage
- **Coroutines**: Proper scope management, structured concurrency, cancellation handling
- **Kotlin idioms**: Data classes, sealed classes, extension functions used appropriately

### Categorize Findings

For each finding, assign a severity:
- **Critical**: Bugs, crashes, data loss, security issues — must fix before merge
- **Major**: Logic errors, missing error handling, missing tests for important paths — should fix
- **Minor**: Style issues, naming suggestions, small improvements — nice to have
- **Nitpick**: Purely cosmetic or subjective — take it or leave it
- **Positive**: Something done well worth calling out

For each finding that references a specific line, record:
- `file_path` — the file
- `old_line` / `new_line` — line number (old for deletions, new for additions)
- `severity` — one of the above
- `comment` — the review comment text

---

## Phase 8: Present Review to User

Present the review findings to the user, organized by severity:

```
## MR Review: <MR_TITLE>
**Ticket**: <TICKET_KEY> — <TICKET_TITLE>
**Branch**: <source_branch> → <target_branch>
**Changed files**: <count>

### Critical Issues
1. [file:line] <description>

### Major Issues
1. [file:line] <description>

### Minor Issues
1. [file:line] <description>

### Nitpicks
1. [file:line] <description>

### Positives
1. [file:line] <description>

### Overall Assessment
<1-2 sentence summary: is this MR ready to merge, or does it need changes?>
```

Then ask the user which comments to post using AskUserQuestion:

> Which review comments would you like me to post to the MR?

Options:
- **All comments** — Post every finding as a comment
- **Critical and Major only** — Skip minor/nitpick/positive
- **Let me choose** — I'll present each one for approval
- **None** — Don't post any comments, just keep the review local

---

## Phase 9: Post Comments to MR

Based on the user's selection, post comments to the merge request.

### For inline comments (tied to specific lines):

Use `mcp__corteva__gitlab-seed__create_merge_request_thread` with a `position` object:
```
project_id: "<GITLAB_PROJECT_PATH_ENCODED>"
merge_request_iid: "<MR_IID>"
body: "<severity-prefix> <comment-text>"
position:
  base_sha: "<diff_refs.base_sha>"
  head_sha: "<diff_refs.head_sha>"
  start_sha: "<diff_refs.start_sha>"
  position_type: "text"
  old_path: "<file_path>"
  new_path: "<file_path>"
  new_line: <line_number>  (for added/modified lines)
  old_line: <line_number>  (for removed lines)
```

Prefix each comment body with the severity:
- Critical: `**Critical:**`
- Major: `**Major:**`
- Minor: `**Minor:**`
- Nitpick: `**Nitpick:**`
- Positive: `**Positive:**`

### For general comments (not tied to a specific line):

Use `mcp__corteva__gitlab-seed__create_merge_request_note`:
```
project_id: "<GITLAB_PROJECT_PATH_ENCODED>"
merge_request_iid: "<MR_IID>"
body: "<comment-text>"
```

### Post a summary comment

After posting individual comments, post a summary note using `mcp__corteva__gitlab-seed__create_merge_request_note`:

```
**Code Review Summary**

Reviewed by Claude Code.

- Critical: <count>
- Major: <count>
- Minor: <count>
- Nitpick: <count>
- Positive: <count>

**Overall**: <1-2 sentence assessment>
```

### If user selected "Let me choose":

For each comment, present it to the user and ask whether to post it. Use the AskUserQuestion tool, batching up to 4 comments per question. For each, show the file, line, severity, and comment text. Options: **Post** or **Skip**.

---

## Phase 10: Cleanup

1. Inform the user that the review is complete with:
   - MR URL
   - Ticket key
   - Number of comments posted (by severity)
   - Overall assessment

2. If there were critical issues, suggest the MR author address them before merging.
