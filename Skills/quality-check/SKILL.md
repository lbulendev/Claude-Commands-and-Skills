---
name: quality-check
description: Run tests, linting, type checking, and formatting checks using Nx affected. Use after making code changes to verify quality.
user-invocable: true
---

# Quality Check

Run quality checks on affected projects using Nx.

## Steps

### 1. Determine Affected Projects

The checks use `nx affected` which compares the current branch against `main` to determine which projects are affected by the changes.

### 2. Run Tests

```bash
npx nx affected -t test --base=main
```

If any tests fail:
- Read the test output to understand failures
- Fix the failing tests or the implementation causing failures
- Re-run until all tests pass

### 3. Run Linting

```bash
npx nx affected -t lint --base=main
```

If any linting errors are found:
- Fix the linting issues
- Re-run until clean

### 4. Run Type Checking

```bash
npx nx affected -t typecheck --base=main
```

If any type errors are found:
- Fix the type errors
- Re-run until clean

### 5. Run Formatting

```bash
npx nx affected -t format --base=main
```

If any formatting issues are found:
- Run `npx nx affected -t format:fix --base=main` to auto-fix
- If auto-fix is not available, fix manually
- Re-run until clean

### 6. Report Results

Summarize the results:
- Number of projects tested
- Number of projects linted
- Number of projects type-checked
- Number of projects formatted
- Any issues that were found and fixed
- Final pass/fail status

If all checks pass, confirm the code is ready to commit.
