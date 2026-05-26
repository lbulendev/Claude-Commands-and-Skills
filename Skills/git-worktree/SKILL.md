---
name: git-worktree
description: Create a new git worktree with a feature branch for isolated development. Use when starting work on a new feature or ticket.
argument-hint: [branch-name]
user-invocable: true
---

# Create Git Worktree

Create a new git worktree with branch **$ARGUMENTS**.

## Steps

### 1. Validate Branch Name

The branch name should be in kebab-case. If it contains spaces or invalid characters, convert it to a valid kebab-case branch name.

### 2. Determine Worktree Path

The worktree will be created as a sibling directory to the current repository:
- Current repo: `/Users/adam.johnson/repos/fabric3`
- Worktree path: `/Users/adam.johnson/repos/fabric3-$ARGUMENTS`

### 3. Check for Existing Worktree

Run `git worktree list` to check if a worktree already exists for this branch. If it does, inform the user and ask if they want to use the existing one or remove it and create a fresh one.

### 4. Ensure Clean State

Run `git status` on the main repository. If there are uncommitted changes, warn the user and ask how to proceed.

### 5. Update Main Branch

```bash
git fetch origin main
git checkout main
git pull origin main
```

### 6. Create the Worktree

```bash
git worktree add ../fabric3-$ARGUMENTS -b $ARGUMENTS
```

If the branch already exists on the remote, use:
```bash
git worktree add ../fabric3-$ARGUMENTS $ARGUMENTS
```

### 7. Install Dependencies

```bash
cd ../fabric3-$ARGUMENTS
pnpm install
```

### 8. Confirm

Report to the user:
- Worktree path
- Branch name
- That dependencies are installed and the worktree is ready for development

**Important**: All subsequent commands should be run from the worktree directory, not the main repository.
