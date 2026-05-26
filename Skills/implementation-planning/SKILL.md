---
name: implementation-planning
description: Create a detailed implementation plan from all gathered Jira context (repo decisions, implementation decisions, ticket description) and post it back to the ticket.
argument-hint: ticket-key
user-invokable: true
---

# Implementation Planning: $ARGUMENTS

Create a detailed implementation plan for Jira ticket **$ARGUMENTS** using all previously gathered context.

---

## Jira Bootstrap

**IMPORTANT: Always use `mcp__atlassian__*` tools for ALL Jira operations. Never use curl or direct REST API calls for Jira.**

1. Validate that `$ARGUMENTS` matches the format `ABC-1234` (project prefix, hyphen, number). If not, stop immediately.
2. Get the cloudId using `mcp__atlassian__getAccessibleAtlassianResources`. Use the first result's `id` as the `cloudId`.
3. Fetch the ticket using `mcp__atlassian__getJiraIssue` with `issueIdOrKey: $ARGUMENTS`, `expand: "renderedFields"`, and `fields: ["summary", "description", "status", "labels", "comment", "reporter", "customfield_15962"]`. This single call returns all fields and comments needed — do not make additional calls for this data.
4. Extract `fields.reporter.accountId` for tagging in comments.

---

## Step 1: Gather All Context

Using the comments already fetched in Jira Bootstrap, collect:

1. **REPO DECISIONS** comment -> extract the list of repos with their GitLab URLs
2. **IMPLEMENTATION DECISIONS** comment -> extract all Q&A pairs (confirmed decisions)
3. **Ticket description** -> `fields.description`
4. **Ticket title** -> `fields.summary`

If `**REPO DECISIONS**` is not found, stop immediately — repo discovery has not been completed.
If `**IMPLEMENTATION DECISIONS**` is not found, that is acceptable — it means no clarifying questions were needed.

---

## Step 2: Explore Each Repo

For each repo listed in REPO DECISIONS:

1. Clone (shallow) to a temp directory if not already cloned:
   ```bash
   CLONE_DIR="/tmp/claude-impl-$ARGUMENTS/$(basename <repo-url> .git)"
   if [ ! -d "$CLONE_DIR" ]; then
     git clone --depth=50 <repo-url> "$CLONE_DIR"
   fi
   ```

2. **Detect repo type:**
   ```bash
   HAS_IOS=false; HAS_ANDROID=false
   if [ -d "$CLONE_DIR/swift" ] || [ -d "$CLONE_DIR/ios" ] || find "$CLONE_DIR" -maxdepth 2 -name "*.xcodeproj" | grep -q .; then HAS_IOS=true; fi
   if [ -d "$CLONE_DIR/android" ] && ([ -f "$CLONE_DIR/android/settings.gradle.kts" ] || [ -f "$CLONE_DIR/android/settings.gradle" ]); then HAS_ANDROID=true; fi
   ```
   Store as `REPO_TYPE`: `mobile` (both), `ios`, `android`, `frontend`, `backend`, or `unknown`.

3. Use the Agent tool with `subagent_type=Explore` to understand the repo:
   - Project structure and architecture
   - Files related to the ticket requirements
   - Existing patterns and conventions
   - Test patterns and frameworks
   - Check for AGENTS.md/CLAUDE.md

   **For mobile repos** (`REPO_TYPE` is `mobile`, `ios`, or `android`):
   - **iOS** (if `HAS_IOS`): Read `skills/pfw-pfw/SKILL.md` first, then relevant `pfw-*` skills. Understand TCA architecture, data sync patterns.
   - **Android** (if `HAS_ANDROID`): Read `android/skills/` if present. Understand MVVM/Hilt architecture, Compose patterns.

4. **For mobile repos, determine `TARGET_PLATFORMS`** from the ticket title:
   - Title contains "iOS" but not "Android" → `ios`
   - Title contains "Android" but not "iOS" → `android`
   - Title contains both or neither → `both`

---

## Step 3: Create the Plan

For each repo, create a plan section. The format depends on the repo type.

**For web repos** (frontend/backend):

1. **Repo Name and URL** — header identifying the repo
2. **Summary of Changes** — brief overview of what will be implemented in this repo
3. **Files to Modify/Create** — list of files with a description of changes for each
4. **Implementation Steps** — ordered list of steps with enough detail to execute
5. **Tests to Write** — covering all acceptance criteria:
   - Positive tests (happy path)
   - Negative tests (error cases, edge cases, boundary conditions)
6. **Risks and Considerations** — potential risks or trade-offs for this repo

**For mobile repos** (`REPO_TYPE` is `mobile`, `ios`, or `android`):

Use platform-specific sections so the mobile executor skills (`implement-ios`, `implement-android`, `implement-mobile`) can parse the plan:

1. **Repo Name and URL** — header identifying the repo
2. **Summary** — brief overview, stating which platform(s) are targeted (`TARGET_PLATFORMS`)
3. **iOS Changes** (only if `TARGET_PLATFORMS` is `ios` or `both`):
   - Files to modify/create (under `swift/` or `swift_libraries/`)
   - TCA reducer/view/dependency changes
   - Implementation steps
   - Tests (using Point-Free testing patterns: `TestStore`, `@Dependency`)
4. **Android Changes** (only if `TARGET_PLATFORMS` is `android` or `both`):
   - Files to modify/create (under `android/`)
   - ViewModel/Composable/Repository changes
   - Implementation steps
   - Tests (JUnit 4, MockK, coroutines test)
5. **Shared Considerations** (only if `TARGET_PLATFORMS` is `both`):
   - API contracts / data models that must match across platforms
   - Feature flag coordination
   - Offline/sync behavior
6. **Risks and Considerations**

When information conflicts between the description and IMPLEMENTATION DECISIONS, **prefer the decisions**.

---

## Step 4: Post Plan and Advance

1. Update the AI Status field:
   ```
   mcp__atlassian__editJiraIssue with fields: { "customfield_15962": "implementing" }
   ```

2. Post the plan as a comment using `mcp__atlassian__addCommentToJiraIssue`:
   ```
   **IMPLEMENTATION PLAN**

   ## <Repo 1 Name> (<repo-url>)

   ### Summary
   <summary of changes>

   ### Files to Modify
   - `path/to/file.ts` — <description>
   - `path/to/new-file.ts` — <description>

   ### Implementation Steps
   1. <step>
   2. <step>

   ### Tests
   - <test description>
   - <test description>

   ### Risks
   - <risk>

   ---

   ## <Repo 2 Name> (<repo-url>)
   ...
   ```

3. **Stop.** The comment will trigger the webhook, which invokes `/ticket-discovery`. The router will read the updated AI Status (`implementing`) and route to the next skill.
