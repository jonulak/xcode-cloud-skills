---
name: xcode-cloud-triage
description: Diagnose why an Xcode Cloud build failed by walking build actions, issues, and test results with the asc CLI. Use when an Xcode Cloud build failed, errored, timed out, or has failing tests, or when asked why a build broke.
---

# Triaging a failed Xcode Cloud build

Find out *why* a build run failed. To start or monitor builds, use
`xcode-cloud-runs`. To fix something that belongs in a build script, use
`xcode-cloud-scripts`.

Verify command shapes with `asc xcode-cloud <subcommand> --help` before running —
the CLI's published docs lag the binary.

## The action chain

A build run is not a single unit. It contains **build actions** — discrete steps
like Resolve Dependencies, Build, Test, Archive, Upload — and failures belong to a
specific action. Issues and test results hang off actions, not off the run.

This matters mechanically. On `issues list` and `test-results list`, `--action-id`
is the real selector; `--run-id` is a convenience that, per the CLI's own help text,
resolves only **a single** action from the run. On a run that built *and* tested
*and* archived, a lone `--run-id` call will quietly under-report.

So: fan out over actions.

```bash
# 1. What steps ran, and which failed?
asc xcode-cloud actions list --run-id "$RUN_ID" --output table

# 2. For each failed action, pull its issues
asc xcode-cloud issues list --action-id "$ACTION_ID"

# 3. Expand anything unclear
asc xcode-cloud issues view --id "$ISSUE_ID"
```

Scripted fan-out over every action in a run:

```bash
for a in $(asc xcode-cloud actions list --run-id "$RUN_ID" | jq -r '.data[].id'); do
  echo "── action $a"
  asc xcode-cloud issues list --action-id "$a"
done
```

## Failing tests

```bash
asc xcode-cloud test-results list --action-id "$ACTION_ID"
asc xcode-cloud test-results view --id "$TEST_RESULT_ID"
```

For the full `.xcresult` bundle, fetch it from the action's artifacts:

```bash
asc xcode-cloud artifacts list --run-id "$RUN_ID"
asc xcode-cloud artifacts download --id "$ARTIFACT_ID" --path ./result.zip
```

Logs are artifacts too — when `issues` is thin, the downloaded log is usually where
the real error text lives.

## Reading the failure

Match the failed action to the likely cause before guessing:

| Failed action | Look first at |
|---|---|
| Resolve Dependencies | SPM resolution, private repo auth, `Package.resolved` drift |
| Build / Test | compile errors, missing files not committed, simulator/runtime availability |
| Archive | code signing, provisioning, entitlements, bundle ID mismatch |
| Upload | App Store Connect validation, build number collision, missing export compliance |
| Any, with no issues at all | a custom build script — see below |

### Common causes

**Custom script never ran.** The most common silent failure. Scripts must live in
`ci_scripts/` at the repo root, be named exactly `ci_post_clone.sh`,
`ci_pre_xcodebuild.sh`, or `ci_post_xcodebuild.sh`, and be committed with the
executable bit:

```bash
git ls-files -s ci_scripts/     # must show mode 100755, not 100644
```

**Script failed and took the build with it.** A script under `set -e` fails the
build on any non-zero command. Non-essential steps (linting, notifications) should
not be fatal. See `xcode-cloud-scripts`.

**Private SPM dependencies.** Xcode Cloud's source-control credentials cover the
primary repo; additional private packages need explicit auth configured in
`ci_post_clone.sh`, or the repo registered as an additional repository on the
product.

**Signing.** Prefer Xcode Cloud's managed signing. If the project pins
`PROVISIONING_PROFILE_SPECIFIER` or uses manual signing, the profile has to exist
and match the team — check `CODE_SIGN_STYLE` and `DEVELOPMENT_TEAM` in the project.

**Timeout.** Distinguish two different things: the *build* hitting Xcode Cloud's
own limit, versus your local `asc` request timing out while waiting. The latter is
a client-side `--timeout`/`ASC_TIMEOUT` problem and the build may still be running.

**Works locally, fails in CI.** Usually something uncommitted, something in
`.gitignore`, a tool present on your Mac but not the build environment, or a
case-sensitivity difference in file paths.

## Rerunning

Once you have a hypothesis, rerun the same source without re-triggering from
scratch. `--clean` bypasses caches, which is the right call when you suspect stale
derived data or dependency resolution:

```bash
asc xcode-cloud run --source-run-id "$RUN_ID" --clean --wait --timeout 1h
```

## Reporting back

State the failed action, the concrete error, and the specific fix. If the evidence
is thin — no issues recorded, empty logs — say that plainly and name what you'd need
next (the downloaded log artifact, the workflow's environment settings) rather than
presenting a guess as a diagnosis.

## Reference

- [Xcode Cloud troubleshooting](https://developer.apple.com/documentation/xcode/troubleshooting-xcode-cloud-issues)
- [`asc` CLI](https://github.com/rorkai/App-Store-Connect-CLI)
