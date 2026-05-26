---
name: create-mr
description: Create a GitLab merge request for the current branch. Use after committing and pushing changes.
argument-hint: [ticket-key]
user-invocable: true
---

# Create GitLab Merge Request

Create a merge request in GitLab for the current branch, linked to Jira ticket **$ARGUMENTS**.

## Steps

### 1. Gather Branch Info

```bash
git branch --show-current
git log main..HEAD --oneline
git diff main...HEAD --stat
```

Collect:
- Current branch name (this becomes the source branch)
- List of commits since diverging from main
- Summary of changed files

### 2. Verify Push Status

Check if the branch has been pushed to the remote:

```bash
git status -sb
```

If the branch has not been pushed, push it:

```bash
git push -u origin $(git branch --show-current)
```

### 3. Compose MR Title

Format: `$ARGUMENTS: Brief imperative description of the change`

The title should be under 70 characters. Use the Jira ticket key as a prefix.

### 4. Compose MR Description

Write a clear, concise description in imperative mood. Use this format:

```markdown
## Summary

- Key change 1
- Key change 2
- Key change 3

Generated with [Claude Code](https://claude.com/claude-code)
```

If there were complex decisions made during the implementation, include a brief paragraph explaining them before the summary list.

Do not use emojis. Keep it succinct.

### 5. Create the Merge Request

```bash
curl --request POST \
  --header "PRIVATE-TOKEN: $GITLAB_TOKEN" \
  --header "Content-Type: application/json" \
  --data '{
    "source_branch": "<current-branch>",
    "target_branch": "main",
    "title": "<mr-title>",
    "description": "<mr-description>",
    "remove_source_branch": true,
    "squash": true
  }' \
  "https://gitlab.research.corteva.com/api/v4/projects/granular%2Ffabric%2Ffabric3/merge_requests"
```

**Required settings**:
- `remove_source_branch`: always `true`
- `squash`: always `true`

### 6. Extract and Display MR URL

Parse the JSON response and extract the `web_url` field. Display the MR URL to the user.

If the API returns an error, display the error message and suggest corrective action.
