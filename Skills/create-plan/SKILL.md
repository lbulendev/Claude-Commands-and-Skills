---
name: create-plan
description: Create an implementation plan for a Jira ticket. Reads the ticket, checks for existing plans and decisions, asks clarifying questions, and posts the plan back to Jira.
argument-hint: [ticket-key]
user-invocable: true
---

# Create Implementation Plan

Create an implementation plan for Jira ticket **$ARGUMENTS**.

## Steps

### 1. Fetch the Ticket

Use `mcp_atlassian_getJiraIssue` to fetch ticket `$ARGUMENTS`. If the ticket cannot be found, inform the user and stop immediately.

### 2. Extract Ticket Details

Read the title, description, and all comments from the ticket.

### 3. Check for Existing Plan

Look through the comments for one that starts with `**IMPLEMENTATION PLAN**`.

- If found, display the existing plan to the user and ask if they want to use it as-is, modify it, or create a new one.
- If the user wants to use it as-is, stop here and return the plan.

### 4. Check for Implementation Decisions

Look through the comments for one that starts with `**IMPLEMENTATION DECISIONS**`. If found, incorporate these decisions into your understanding. When there is conflicting information between the description and decisions, **prefer the decisions**.

### 5. Explore the Codebase

First, detect the project type:
```bash
PROJECT_ROOT=$(git rev-parse --show-toplevel)
HAS_IOS=false; HAS_ANDROID=false
if [ -d "$PROJECT_ROOT/swift" ] || [ -d "$PROJECT_ROOT/ios" ] || find "$PROJECT_ROOT" -maxdepth 2 -name "*.xcodeproj" | grep -q .; then HAS_IOS=true; fi
if [ -d "$PROJECT_ROOT/android" ] && ([ -f "$PROJECT_ROOT/android/settings.gradle.kts" ] || [ -f "$PROJECT_ROOT/android/settings.gradle" ]); then HAS_ANDROID=true; fi
```
Store as `REPO_TYPE`: `mobile` (both), `ios`, `android`, `frontend`, `backend`, or `unknown`.

**For mobile repos, determine `TARGET_PLATFORMS`** from the ticket title:
- Title contains "iOS" but not "Android" → `ios`
- Title contains "Android" but not "iOS" → `android`
- Title contains both or neither → `both`

Then use the Task tool with `subagent_type=Explore` to understand the relevant parts of the codebase:
- Identify files and directories related to the ticket's requirements
- Understand existing patterns, conventions, and architecture
- Find related tests and test patterns

**For mobile repos**, also:
- **iOS** (if `TARGET_PLATFORMS` is `ios` or `both`): Read `skills/pfw-pfw/SKILL.md` first, then relevant `pfw-*` skills. Understand TCA architecture.
- **Android** (if `TARGET_PLATFORMS` is `android` or `both`): Read `android/skills/` if present. Understand MVVM/Hilt architecture.

### 6. Build Clarifying Questions

Based on the description, decisions (if any), and codebase exploration:
- Identify any ambiguities or gaps in the requirements
- **Do not make assumptions** - if something is unclear, ask
- Build a list of clarifying questions

Use the AskUserQuestion tool to ask these questions interactively. Collect all answers.

### 7. Post Implementation Decisions

If any clarifying questions were asked and answered, post a comment to the Jira ticket using `mcp_atlassian_addCommentToJiraIssue`:

```
**IMPLEMENTATION DECISIONS**

Q: [question 1]
A: [answer 1]

Q: [question 2]
A: [answer 2]
```

### 8. Create the Implementation Plan

The plan format depends on the repo type.

**For web repos** (frontend/backend), build a plan that includes:

1. **Summary** - Brief overview of what will be implemented
2. **Files to Modify** - List of files that will be created or modified, with a description of changes for each
3. **Implementation Steps** - Ordered list of implementation steps, each with enough detail to execute
4. **Tests** - Tests that cover all acceptance criteria from the ticket:
   - Positive tests (happy path)
   - Negative tests (error cases, edge cases, boundary conditions)
5. **Risks and Considerations** - Any potential risks or trade-offs

**For mobile repos** (`REPO_TYPE` is `mobile`, `ios`, or `android`), use platform-specific sections:

1. **Summary** - Brief overview, stating which platform(s) are targeted (`TARGET_PLATFORMS`)
2. **iOS Changes** (only if `TARGET_PLATFORMS` is `ios` or `both`):
   - Files to modify/create (under `swift/` or `swift_libraries/`)
   - TCA reducer/view/dependency changes
   - Implementation steps
   - Tests (using Point-Free testing patterns: `TestStore`, `@Dependency`)
3. **Android Changes** (only if `TARGET_PLATFORMS` is `android` or `both`):
   - Files to modify/create (under `android/`)
   - ViewModel/Composable/Repository changes
   - Implementation steps
   - Tests (JUnit 4, MockK, coroutines test)
4. **Shared Considerations** (only if `TARGET_PLATFORMS` is `both`):
   - API contracts / data models that must match across platforms
   - Feature flag coordination
   - Offline/sync behavior
5. **Risks and Considerations** - Any potential risks or trade-offs

### 9. Solicit User Feedback

Present the plan to the user and ask for feedback. Iterate on the plan based on their feedback until they approve it.

### 10. Post the Plan to Jira

Once approved, post the plan as a comment to the Jira ticket using `mcp_atlassian_addCommentToJiraIssue`:

```
**IMPLEMENTATION PLAN**

[full plan content]
```

Confirm to the user that the plan has been posted.
