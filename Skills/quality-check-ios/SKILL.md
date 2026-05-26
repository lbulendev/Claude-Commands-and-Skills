---
name: quality-check-ios
description: Run SwiftLint, xcodebuild tests, and build verification for iOS projects. Use after making iOS code changes to verify quality.
user-invokable: true
---

# Quality Check iOS

Run quality checks on the iOS portion of a mobile project.

## Phase 0: Detect Project Layout

1. **Locate the Xcode project or workspace:**
   ```bash
   PROJECT_ROOT=$(git rev-parse --show-toplevel)

   # Prefer workspace if one exists, otherwise use xcodeproj
   XCWORKSPACE=$(find "$PROJECT_ROOT" -maxdepth 3 -name "*.xcworkspace" ! -path "*/xcuserdata/*" ! -path "*/.build/*" | head -1)
   XCODEPROJ=$(find "$PROJECT_ROOT" -maxdepth 3 -name "*.xcodeproj" ! -path "*/.build/*" | head -1)

   if [ -n "$XCWORKSPACE" ]; then
     BUILD_TARGET="-workspace $XCWORKSPACE"
   elif [ -n "$XCODEPROJ" ]; then
     BUILD_TARGET="-project $XCODEPROJ"
   else
     echo "ERROR: No Xcode project or workspace found"; exit 1
   fi
   echo "BUILD_TARGET=$BUILD_TARGET"
   ```

2. **Detect available schemes:**
   ```bash
   xcodebuild -list $BUILD_TARGET 2>/dev/null | grep -A 50 "Schemes:" | tail -n +2 | sed '/^$/d' | sed 's/^[[:space:]]*//'
   ```
   Prefer a scheme named "CI Test" or "Test" for running tests. Fall back to the main app scheme.

3. **Detect SwiftLint:**
   ```bash
   if [ -f "$PROJECT_ROOT/swiftlint.yml" ] || [ -f "$PROJECT_ROOT/.swiftlint.yml" ] || command -v swiftlint &>/dev/null; then
     HAS_SWIFTLINT=true
   else
     HAS_SWIFTLINT=false
   fi
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

## Step 1: Run SwiftLint

Skip if `HAS_SWIFTLINT` is false.

If Fastlane is available and has a `lint` lane:
```bash
fastlane ios lint
```

Otherwise run SwiftLint directly:
```bash
cd "$PROJECT_ROOT/swift"  # or wherever the Swift source lives
swiftlint lint --reporter emoji
```

If linting errors are found:
- Try auto-fix: `swiftlint lint --fix`
- Re-run to verify
- Fix any remaining issues manually

---

## Step 2: Build the Project

Verify the project compiles before running tests:
```bash
xcodebuild build-for-testing \
  $BUILD_TARGET \
  -scheme "<TEST_SCHEME>" \
  -destination "platform=iOS Simulator,name=iPhone 17" \
  -skipMacroValidation \
  -skipPackagePluginValidation \
  | tail -20
```

If build fails:
- Read the error output
- Fix compilation errors (missing imports, type mismatches, syntax errors)
- Re-run until the build succeeds

---

## Step 3: Run Unit Tests

### 3a. Run Main App Tests

Run the main app test suite from the Xcode project:
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

### 3b. Run Affected Library Tests

**IMPORTANT:** If any changes were made to files under `swift_libraries/`, you MUST also run the tests for each affected library. Library tests cannot be run from the parent Xcode project — they must be run from the library's own package directory.

1. Identify affected libraries by checking which `swift_libraries/` subdirectories have modified files:
   ```bash
   git diff --name-only <DEFAULT_BRANCH>...HEAD | grep "^swift_libraries/" | cut -d/ -f2 | sort -u
   ```

2. For each affected library, run its tests from the package directory:
   ```bash
   cd "$PROJECT_ROOT/swift_libraries/<library-name>"
   xcodebuild test \
     -scheme "<LibraryScheme>" \
     -destination "platform=iOS Simulator,name=iPhone 17" \
     -skipMacroValidation \
     -quiet \
     2>&1 | tail -30
   ```

   The scheme name typically matches the library directory name in PascalCase (e.g., `granular-off` → `GranularOff`, `granular-platform` → `GranularPlatform`).

3. All library test suites must pass before proceeding.

If tests fail:
- Read the test output to understand failures
- Fix the failing tests or the implementation causing failures
- Re-run until all tests pass

---

## Step 4: Report Results

Summarize:
- SwiftLint: pass/fail and issues found
- Build: pass/fail
- Main app tests: pass/fail with count
- Library tests: pass/fail per library with count
- Any issues that were found and fixed
- Final pass/fail status

If all checks pass, confirm the code is ready to commit.
