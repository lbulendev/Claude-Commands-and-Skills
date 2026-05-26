---
name: implementation-discovery
description: Explore affected repos and ask clarifying questions via Jira before planning. Clones repos, analyzes code, identifies ambiguities, and posts questions or decisions back to the ticket.
argument-hint: ticket-key
user-invokable: true
---

# Implementation Discovery: $ARGUMENTS

Explore all affected repositories for Jira ticket **$ARGUMENTS** and gather any remaining information needed before creating an implementation plan.

---

## Jira Bootstrap

**IMPORTANT: Always use `mcp__atlassian__*` tools for ALL Jira operations. Never use curl or direct REST API calls for Jira.**

1. Validate that `$ARGUMENTS` matches the format `ABC-1234` (project prefix, hyphen, number). If not, stop immediately.
2. Get the cloudId using `mcp__atlassian__getAccessibleAtlassianResources`. Use the first result's `id` as the `cloudId`.
3. Fetch the ticket using `mcp__atlassian__getJiraIssue` with `issueIdOrKey: $ARGUMENTS`, `expand: "renderedFields"`, and `fields: ["summary", "description", "status", "labels", "comment", "reporter", "customfield_15962"]`. This single call returns all fields and comments needed — do not make additional calls for this data.
4. Extract `fields.reporter.accountId` for tagging in comments.

---

## Step 1: Parse Repo Decisions

1. Using the comments already fetched in Jira Bootstrap, scan for the `**REPO DECISIONS**` comment.
2. If not found, stop immediately — repo discovery has not been completed.
3. Extract the list of repos with their GitLab URLs.

---

## Step 2: Parse Previous Q&A

1. Using the comments already fetched in Jira Bootstrap, scan for any `**IMPLEMENTATION QUESTIONS**` comments.
2. For each found, look for user replies posted **after** that comment (by timestamp).
3. Collect all previously asked questions and their answers into a Q&A context.
4. Also scan for any `**IMPLEMENTATION DECISIONS**` comment — if found, incorporate those confirmed decisions.

---

## Step 3: Explore Each Repo

For each repo listed in REPO DECISIONS:

1. Clone (shallow) to a temp directory if not already cloned:
   ```bash
   CLONE_DIR="/tmp/claude-impl-$ARGUMENTS/$(basename <repo-url> .git)"
   if [ ! -d "$CLONE_DIR" ]; then
     git clone --depth=50 <repo-url> "$CLONE_DIR"
   fi
   ```

2. **Detect repo type** before exploring:
   ```bash
   HAS_IOS=false; HAS_ANDROID=false
   if [ -d "$CLONE_DIR/swift" ] || [ -d "$CLONE_DIR/ios" ] || find "$CLONE_DIR" -maxdepth 2 -name "*.xcodeproj" | grep -q .; then HAS_IOS=true; fi
   if [ -d "$CLONE_DIR/android" ] && ([ -f "$CLONE_DIR/android/settings.gradle.kts" ] || [ -f "$CLONE_DIR/android/settings.gradle" ]); then HAS_ANDROID=true; fi
   ```
   Store as `REPO_TYPE`: `mobile` (both), `ios`, `android`, `frontend`, `backend`, or `unknown`.

3. Use the Agent tool with `subagent_type=Explore` to thoroughly understand the repo:
   - **Project structure:** directories, key entry points, architecture
   - **Relevant files:** files related to the ticket requirements
   - **Patterns and conventions:** coding style, naming conventions, component patterns
   - **Test patterns:** test framework, test file locations, how tests are structured
   - **Project config:** check for AGENTS.md, CLAUDE.md, or similar guidance files
   - **Dependencies:** relevant libraries and frameworks in use

   **Additional exploration for mobile repos** (`REPO_TYPE` is `mobile`, `ios`, or `android`):
   - **iOS** (if `HAS_IOS`): Read `skills/pfw-pfw/SKILL.md` first, then relevant `pfw-*` skills. Identify TCA features, reducers, views, and data sync patterns.
   - **Android** (if `HAS_ANDROID`): Read `android/skills/` if present. Identify ViewModels, Composables, Hilt modules, and repository patterns.
   - **Shared:** Check for Mapbox usage, LaunchDarkly feature flags, offline/sync behavior, and localization (Lokalise).

4. Summarize findings per repo, noting the detected `REPO_TYPE`.

---

## Step 4: Identify Remaining Ambiguities

**Do not make assumptions — ask about anything that is unclear or underspecified.** It is always better to ask a question than to guess wrong. A good implementation plan requires clear answers upfront; skipping questions leads to rework later.

Based on:
- The ticket description (`fields.description`)
- The ticket title (`fields.summary`)
- Previous Q&A context (from Step 2)
- Codebase exploration results (from Step 3)

Actively look for ambiguities and gaps in **each** of the following categories:
- **Requirements clarity:** Can each acceptance criterion be implemented without interpretation? If a requirement could be read two ways, ask.
- **Implementation approach:** Are there multiple valid approaches in the codebase? Ask which to follow.
- **Feature flags:** Should changes be behind a LaunchDarkly feature flag? If the ticket doesn't mention one but the feature seems like it should be gated, ask.
- **Cross-repo interactions:** If multiple repos are involved, how do they coordinate? Are there API contract changes, shared types, or deployment ordering concerns?
- **Edge cases and error handling:** What should happen on failure? Are there boundary conditions or empty-state behaviors not specified?
- **UI/UX details (if applicable):** Are there designs or mockups? What about loading states, error states, empty states, responsive behavior?
- **Data and schema:** Are there database migrations, new fields, or data format changes implied but not specified?
- **Testing expectations:** Are there specific test scenarios expected beyond the obvious happy path?

**Additional questions for mobile repos** (`REPO_TYPE` is `mobile`, `ios`, or `android`):
- **Platform scope:** If the ticket title doesn't specify iOS or Android, ask which platform(s) should be implemented.
- **Platform parity:** Should the behavior be identical on both platforms, or are there platform-specific differences?
- **Offline/sync:** Does this feature need to work offline? Are there data sync implications (DownloadSyncEngine, UploadSyncEngine)?
- **Localization:** Does this feature include user-facing strings that need Lokalise translation?

**Do not ask questions that have already been answered** in previous Q&A. But for everything else, err on the side of asking.

---

## Step 5: Route Based on Remaining Questions

**Most tickets should result in at least 1-2 questions.** Only skip questions if the ticket is exceptionally well-specified with clear acceptance criteria, explicit implementation details, and no ambiguity after codebase exploration.

### If there ARE clarifying questions (expected path):

1. Add the `waiting_for_user` label:
   - Fetch current labels from the ticket
   - Add `waiting_for_user` to the array
   - Update via `mcp__atlassian__editJiraIssue` with `fields: { "labels": [...existing, "waiting_for_user"] }`

2. Post a comment using `mcp__atlassian__addCommentToJiraIssue`, tagging the reporter:
   ```
   **IMPLEMENTATION QUESTIONS**

   [~accountId:<reporter-accountId>] I've explored the affected repositories and have the following questions before I can create an implementation plan:

   1. <question 1>
   2. <question 2>
   3. <question 3>
   ```

3. Stop. The workflow will resume when the user replies and the webhook re-triggers `/ticket-discovery`.

### If there are truly NO remaining questions (rare):

1. Update the AI Status field:
   ```
   mcp__atlassian__editJiraIssue with fields: { "customfield_15962": "implementation_planning" }
   ```

2. Post a comment using `mcp__atlassian__addCommentToJiraIssue`:
   ```
   **IMPLEMENTATION DECISIONS**

   All information gathered. Summary of confirmed decisions:

   Q: <question from earlier>
   A: <answer from user>

   Q: <question from earlier>
   A: <answer from user>

   Additional context from code exploration:
   - <finding 1>
   - <finding 2>
   ```

   If no questions were ever asked, post:
   ```
   **IMPLEMENTATION DECISIONS**

   No clarifying questions needed. All requirements are clear from the ticket description and code exploration.

   Key findings from code exploration:
   - <finding 1>
   - <finding 2>
   ```

3. **Stop.** The comment will trigger the webhook, which invokes `/ticket-discovery`. The router will read the updated AI Status (`implementation_planning`) and route to the next skill.
