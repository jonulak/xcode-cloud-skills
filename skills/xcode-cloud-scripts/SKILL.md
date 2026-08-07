---
name: xcode-cloud-scripts
description: Write and debug Xcode Cloud custom build scripts in ci_scripts/ (ci_post_clone.sh, ci_pre_xcodebuild.sh, ci_post_xcodebuild.sh) and use the predefined CI_ environment variables. Use when setting up Xcode Cloud dependency installation, code generation, build-number stamping, or post-build notifications.
---

# Xcode Cloud custom build scripts

Author the scripts that run *inside* an Xcode Cloud build. To trigger or monitor
builds, use `xcode-cloud-runs`. To diagnose a failure, use `xcode-cloud-triage`.

## The three hooks

Scripts live in a `ci_scripts/` directory **at the repository root** and must use
these exact names. Xcode Cloud runs whichever ones exist; there is no config file.

```
1. clone repository
2. ci_scripts/ci_post_clone.sh       ← install tools and dependencies
3. resolve Swift package dependencies
4. ci_scripts/ci_pre_xcodebuild.sh   ← code generation, config stamping
5. xcodebuild runs the action
6. ci_scripts/ci_post_xcodebuild.sh  ← notifications, artifact processing
```

`ci_pre_xcodebuild.sh` and `ci_post_xcodebuild.sh` run **once per action**. A
workflow that builds, tests, and archives runs them three times, with
`CI_XCODEBUILD_ACTION` differing each time. Write them to be idempotent and to
branch on the action.

## The executable bit

This is the single most common reason a script silently never runs.

```bash
chmod +x ci_scripts/*.sh
git update-index --chmod=+x ci_scripts/ci_post_clone.sh
git ls-files -s ci_scripts/        # must show 100755, not 100644
```

Setting the local file mode is not enough — the mode has to be committed. Always
verify with `git ls-files -s` after creating scripts.

## Environment variables

Verified against Apple's [environment variable reference](https://developer.apple.com/documentation/xcode/environment-variable-reference).
Availability is **conditional** — check before use, and use absence deliberately as
control flow (`if [ -n "$CI_TAG" ]`).

### Always available

| Variable | Notes |
|---|---|
| `CI` | `TRUE` in Xcode Cloud |
| `CI_XCODE_CLOUD` | `TRUE` in Xcode Cloud — distinguishes it from other CI systems that also set `CI` |
| `CI_XCODEBUILD_ACTION` | `analyze`, `archive`, `build`, `build-for-testing`, or `test-without-building` |
| `CI_XCODEBUILD_EXIT_CODE` | set only *after* the action's `xcodebuild` runs — i.e. in `ci_post_xcodebuild.sh` |
| `CI_START_CONDITION` | `manual`, `manual_rebuild`, `push`, `pr_open`, `pr_update`, or `schedule` |
| `CI_BUILD_ID` / `CI_BUILD_NUMBER` / `CI_BUILD_URL` | unique ID, incrementing number, App Store Connect link |
| `CI_WORKFLOW` / `CI_WORKFLOW_ID` | workflow name and ID |
| `CI_PRODUCT` / `CI_PRODUCT_ID` / `CI_PRODUCT_PLATFORM` | platform is `iOS`, `macOS`, `tvOS`, or `watchOS` |
| `CI_BUNDLE_ID` / `CI_TEAM_ID` | e.g. `com.example.app`, `ABCDE12345` |
| `CI_COMMIT` | Git commit hash |
| `CI_PRIMARY_REPOSITORY_PATH` | cloned source location |
| `CI_WORKSPACE_PATH` / `CI_DERIVED_DATA_PATH` / `CI_PROJECT_FILE_PATH` | |
| `CI_XCODE_PROJECT` / `CI_XCODE_SCHEME` | |

> There is **no** `CI_APP_STORE_ID`. Use `CI_PRODUCT_ID` or `CI_BUNDLE_ID`.

### By start condition

| Variable | Available when |
|---|---|
| `CI_BRANCH` | branch change |
| `CI_TAG` | tag change |
| `CI_GIT_REF` | branch or tag change — canonical ref, e.g. `refs/heads/bug-fix` |
| `CI_PULL_REQUEST_NUMBER`, `CI_PULL_REQUEST_HTML_URL` | pull request change |
| `CI_PULL_REQUEST_SOURCE_BRANCH` / `_TARGET_BRANCH` | pull request change |
| `CI_PULL_REQUEST_SOURCE_COMMIT` / `_TARGET_COMMIT` | source commit equals `CI_COMMIT` |
| `CI_PULL_REQUEST_SOURCE_REPO` / `_TARGET_REPO` | full repo names; equal unless the PR is from a fork |

### Test actions

`CI_RESULT_BUNDLE_PATH` (the `.xcresult`), `CI_TEST_PRODUCTS_PATH`, `CI_TEST_PLAN`
(only when using test plans), and `CI_TEST_DESTINATION_DEVICE_TYPE` /
`_RUNTIME` / `_UDID`.

Xcode Cloud exposes environment variables to test runner processes with a
`TEST_RUNNER_` prefix — that prefix is required by `xcodebuild`, and the test reads
the variable under its original name.

### Archive actions

`CI_ARCHIVE_PATH`, plus the exported signed app paths, which differ by distribution
method:

- `CI_APP_STORE_SIGNED_APP_PATH` — TestFlight / App Store
- `CI_AD_HOC_SIGNED_APP_PATH` — Ad Hoc
- `CI_DEVELOPMENT_SIGNED_APP_PATH` — development
- `CI_DEVELOPER_ID_SIGNED_APP_PATH` — Mac apps distributed outside the App Store

Custom variables set in the workflow's Environment section (including secrets) are
available to scripts by their own names.

Xcode Cloud builds behind an HTTP proxy and sets `HTTP_PROXY` / `HTTPS_PROXY`. Most
tools honour these; a tool that ignores them may fail to reach the network.

## Templates

### ci_post_clone.sh

```bash
#!/bin/sh
set -e

# Install only what the build actually needs. Every tool installed here is
# billed compute time on every single build.
brew install swiftlint

# Private SPM dependencies: the primary repo is authenticated by Xcode Cloud,
# additional private packages are not. GITHUB_TOKEN is a workflow secret.
if [ -n "$GITHUB_TOKEN" ]; then
  git config --global url."https://x-access-token:${GITHUB_TOKEN}@github.com/".insteadOf "https://github.com/"
fi
```

### ci_pre_xcodebuild.sh

```bash
#!/bin/sh
set -e

# Runs once per action — branch on which one.
case "$CI_XCODEBUILD_ACTION" in
  archive)
    API_BASE="https://api.example.com" ;;
  build-for-testing|test-without-building)
    API_BASE="https://staging.example.com" ;;
  *)
    API_BASE="https://dev.example.com" ;;
esac

cat > "$CI_PRIMARY_REPOSITORY_PATH/Sources/BuildConfig.swift" <<EOF
// Generated by ci_pre_xcodebuild.sh — do not edit.
enum BuildConfig {
    static let buildNumber = "$CI_BUILD_NUMBER"
    static let commit = "$CI_COMMIT"
    static let apiBase = "$API_BASE"
}
EOF

# Non-essential checks must not fail the build.
if command -v swiftlint > /dev/null; then
  swiftlint lint --quiet || echo "swiftlint reported issues (non-blocking)"
fi
```

Note the action values: there is no plain `test`. Matching on `"test"` alone never
fires — the real values are `build-for-testing` and `test-without-building`.

### ci_post_xcodebuild.sh

```bash
#!/bin/sh
set -e

if [ "$CI_XCODEBUILD_EXIT_CODE" != "0" ]; then
  STATUS="failed"
else
  STATUS="succeeded"
fi

if [ -n "$SLACK_WEBHOOK_URL" ]; then
  curl -sS -X POST "$SLACK_WEBHOOK_URL" \
    -H "Content-Type: application/json" \
    -d "{\"text\":\"$CI_PRODUCT $CI_WORKFLOW build $CI_BUILD_NUMBER $STATUS — $CI_BUILD_URL\"}"
fi

if [ "$CI_XCODEBUILD_ACTION" = "archive" ] && [ -n "$CI_APP_STORE_SIGNED_APP_PATH" ]; then
  echo "Signed app at $CI_APP_STORE_SIGNED_APP_PATH"
fi
```

## Stamping the build number

```bash
# ci_pre_xcodebuild.sh
PLIST="$CI_PRIMARY_REPOSITORY_PATH/MyApp/Info.plist"
/usr/libexec/PlistBuddy -c "Set :CFBundleVersion $CI_BUILD_NUMBER" "$PLIST"
```

Prefer `CI_BUILD_NUMBER` over inventing your own counter — it already increments
per product and won't collide on App Store Connect upload.

## Pitfalls

- **Not executable / not committed as 100755.** Script never runs, no error shown.
- **Wrong directory or filename.** Must be `ci_scripts/` at the repo root, exact names.
- **`set -e` plus an optional step.** One lint warning fails the whole build. Guard
  optional commands with `|| true` or an explicit message.
- **Assuming a variable exists.** `CI_BRANCH` is empty on a tag build;
  `CI_XCODEBUILD_EXIT_CODE` is unset before the action runs. Test with `-n`.
- **Heavy `ci_post_clone.sh`.** It runs on every build, billed as compute time.
  Install the minimum.
- **Editing tracked files.** Generated files written into the repo should be
  gitignored; the build environment is discarded, so nothing is committed back.
- **`$CI_PULL_REQUEST_NUMBER` on non-PR builds.** Unset. Use it to detect PR builds:
  `if [ -n "$CI_PULL_REQUEST_NUMBER" ]; then ...`

## Reference

- [Writing custom build scripts](https://developer.apple.com/documentation/xcode/writing-custom-build-scripts)
- [Environment variable reference](https://developer.apple.com/documentation/xcode/environment-variable-reference)
- [Xcode Cloud pricing](https://developer.apple.com/xcode-cloud/) — compute-hour
  tiers change; read them there rather than quoting remembered figures
