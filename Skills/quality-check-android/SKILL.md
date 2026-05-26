---
name: quality-check-android
description: Run ktlint, Android lint, Gradle unit tests, and build verification for Android projects. Use after making Android code changes to verify quality.
user-invokable: true
---

# Quality Check Android

Run quality checks on the Android portion of a mobile project.

## Phase 0: Detect Project Layout

1. **Locate the Android project:**
   ```bash
   PROJECT_ROOT=$(git rev-parse --show-toplevel)

   # Find the Android root (contains settings.gradle.kts or settings.gradle)
   if [ -f "$PROJECT_ROOT/android/settings.gradle.kts" ] || [ -f "$PROJECT_ROOT/android/settings.gradle" ]; then
     ANDROID_ROOT="$PROJECT_ROOT/android"
   elif [ -f "$PROJECT_ROOT/settings.gradle.kts" ] || [ -f "$PROJECT_ROOT/settings.gradle" ]; then
     ANDROID_ROOT="$PROJECT_ROOT"
   else
     echo "ERROR: No Android Gradle project found"; exit 1
   fi
   echo "ANDROID_ROOT=$ANDROID_ROOT"
   ```

2. **Detect Gradle wrapper:**
   ```bash
   if [ -f "$ANDROID_ROOT/gradlew" ]; then
     GRADLE_CMD="$ANDROID_ROOT/gradlew"
   else
     GRADLE_CMD="gradle"
   fi
   echo "GRADLE_CMD=$GRADLE_CMD"
   ```

3. **Detect modules** (for multi-module projects):
   ```bash
   cd "$ANDROID_ROOT"
   $GRADLE_CMD projects --quiet 2>/dev/null | grep "---" | sed "s/.*--- Project '\(.*\)'/\1/"
   ```

4. **Detect Fastlane:**
   ```bash
   if [ -d "$PROJECT_ROOT/fastlane" ]; then
     HAS_FASTLANE=true
   else
     HAS_FASTLANE=false
   fi
   ```

---

## Step 1: Run ktlint

```bash
cd "$ANDROID_ROOT"
$GRADLE_CMD ktlintCheck --build-cache
```

If ktlint errors are found:
- Try auto-fix: `$GRADLE_CMD ktlintFormat --build-cache`
- Re-run `ktlintCheck` to verify
- Fix any remaining issues manually

---

## Step 2: Compile the Project

Verify the project compiles:
```bash
cd "$ANDROID_ROOT"
$GRADLE_CMD compileAll --build-cache --stacktrace
```

If build fails:
- Read the error output
- Fix compilation errors (missing imports, type mismatches, Kotlin syntax)
- Re-run until the build succeeds

---

## Step 3: Run Android Lint

```bash
cd "$ANDROID_ROOT"
$GRADLE_CMD :app:lint --build-cache --stacktrace
```

If lint errors are found:
- Read the HTML report at `app/build/reports/lint-results-*.html`
- Fix lint issues
- Re-run until clean or only expected baseline warnings remain

**Note:** Android lint may have a baseline file (`lint-baseline.xml`). New violations beyond the baseline should be fixed.

---

## Step 4: Run Unit Tests

Run tests for each module. For the insights project, these are the standard test tasks:

```bash
cd "$ANDROID_ROOT"
$GRADLE_CMD testDebugUnitTest --stacktrace --build-cache --continue
```

For multi-module projects, run all module tests:
```bash
$GRADLE_CMD :app:testGtestDebugUnitTest --build-cache --continue
$GRADLE_CMD :granular-platform:testDebugUnitTest --build-cache --continue
$GRADLE_CMD :granular-off:testDebugUnitTest --build-cache --continue
$GRADLE_CMD :granular-fielddata:testDebugUnitTest --build-cache --continue
$GRADLE_CMD :granular-mop:testDebugUnitTest --build-cache --continue
$GRADLE_CMD :granular-notes:testDebugUnitTest --build-cache --continue
```

If tests fail:
- Read the test reports at `**/build/reports/tests/`
- Fix the failing tests or the implementation causing failures
- Re-run until all tests pass

---

## Step 5: Report Results

Summarize:
- ktlint: pass/fail and issues found
- Build compilation: pass/fail
- Android lint: pass/fail and issues found
- Unit tests: pass/fail with count per module
- Any issues that were found and fixed
- Final pass/fail status

If all checks pass, confirm the code is ready to commit.
