---
name: xcode-cloud-runs
description: Trigger, monitor, and manage Xcode Cloud build runs and workflows using the asc CLI. Use when asked to start an Xcode Cloud build, check build status, rerun a build, list or edit workflows, or download build artifacts.
---

# Xcode Cloud build runs

Drive Xcode Cloud remotely through the [`asc`](https://github.com/rorkai/App-Store-Connect-CLI) CLI.

To diagnose a build that *failed*, use `xcode-cloud-triage` instead. To write the
scripts that run *inside* a build, use `xcode-cloud-scripts`.

## Before anything else

**Verify the command shape with `--help` before you run it.**

```bash
asc xcode-cloud --help
asc xcode-cloud run --help
```

This is not boilerplate caution. The CLI's own published documentation lags the
binary — at time of writing, `--pull-request-id`, `--source-run-id`, `--clean`,
`artifacts download`, `build-runs --sort`, and every `view` subcommand were real
but undocumented. Trust `--help`, not memory and not the website.

If `asc` is missing: `brew install asc`.

## Preflight

```bash
asc auth status          # which credentials are active
```

Credentials resolve in this order: selected profile (keychain or config) first,
then environment variables for any missing fields. A repo-local `./.asc/config.json`
takes precedence over both.

- `asc auth login` — store a key in the keychain
- `asc auth switch --name "<profile>"` — change the active profile
- Env fallback: `ASC_KEY_ID`, `ASC_ISSUER_ID`, `ASC_PRIVATE_KEY_PATH`, `ASC_APP_ID`
- `--strict-auth` (or `ASC_STRICT_AUTH=true`) fails rather than silently mixing sources
- `ASC_BYPASS_KEYCHAIN=1` skips the keychain

Generate keys at <https://appstoreconnect.apple.com/access/integrations/api>.

## Finding what to build

`--app` accepts an App Store Connect **app ID, bundle ID, or exact app name** — you
rarely need to look up a numeric ID first. `ASC_APP_ID` supplies a default.

Bundle-ID resolution fails when one bundle ID is a prefix of another, which is common
with TestFlight-only companion apps:

```
Error: multiple apps found for bundle ID "com.example.app";
use --app with App Store Connect app ID
```

Recover with `asc apps list --output table` and pass the numeric app ID.

```bash
asc xcode-cloud products list --app "com.example.MyApp"
asc xcode-cloud workflows list --app "com.example.MyApp"
asc xcode-cloud workflows view --id "$WORKFLOW_ID"
asc xcode-cloud build-runs list --workflow-id "$WORKFLOW_ID" --sort -number
```

`--sort -number` puts the newest run first — that's how you answer "what happened
on my last build" without paging.

## Triggering a build

Two selectors, each with two forms. Workflow **by name requires `--app`**; by ID it
does not.

```bash
# by name (needs --app) + branch
asc xcode-cloud run --app "$APP" --workflow "CI" --branch main

# by workflow ID + git reference ID
asc xcode-cloud run --workflow-id "$WORKFLOW_ID" --git-reference-id "$REF_ID"

# build a pull request
asc xcode-cloud run --workflow-id "$WORKFLOW_ID" --pull-request-id "$PR_ID"

# rerun an existing run — no workflow/source selectors, add --clean to skip caches
asc xcode-cloud run --source-run-id "$RUN_ID" --clean
```

### Waiting

`--wait` exists on both `run` and `status`. Poll interval defaults to `10s`.

```bash
asc xcode-cloud run --app "$APP" --workflow "CI" --branch main \
  --wait --poll-interval 30s --timeout 1h
```

Xcode Cloud builds routinely outlast the default 30-minute request timeout. When
waiting on a real build, set `--timeout` (or `ASC_TIMEOUT=1h`) deliberately. A
`--poll-interval` under `30s` on a long build just burns API calls.

## Checking status

```bash
asc xcode-cloud status --run-id "$RUN_ID"
asc xcode-cloud status --run-id "$RUN_ID" --wait --poll-interval 30s --timeout 1h
asc xcode-cloud build-runs view --id "$RUN_ID"
asc xcode-cloud build-runs builds --run-id "$RUN_ID"   # resulting App Store Connect builds
```

## Artifacts

```bash
asc xcode-cloud artifacts list --run-id "$RUN_ID"
asc xcode-cloud artifacts view --id "$ARTIFACT_ID"
asc xcode-cloud artifacts download --id "$ARTIFACT_ID" --path ./artifact.zip --overwrite
```

## Managing workflows

`create` and `update` take a JSON payload file, not flags. Inspect an existing
workflow with `view` first and use it as the template — that is far more reliable
than composing the payload from scratch.

```bash
asc xcode-cloud workflows view --id "$WORKFLOW_ID" --pretty > workflow.json
asc xcode-cloud workflows create --file ./workflow.json
asc xcode-cloud workflows update --id "$WORKFLOW_ID" --file ./workflow.json
asc xcode-cloud workflows delete --id "$WORKFLOW_ID" --confirm
```

**Destructive commands require `--confirm`.** Never add `--confirm` on the user's
behalf to a `delete` they did not explicitly ask for.

Workflow *settings* — start conditions, environment, actions, post-actions — are
otherwise configured in Xcode (Product → Xcode Cloud → Manage Workflows) or App
Store Connect. There is no YAML file to edit.

## Source control

```bash
asc xcode-cloud scm providers list
asc xcode-cloud scm repositories list
asc xcode-cloud scm repositories git-references --repo-id "$REPO_ID"
asc xcode-cloud scm pull-requests list
```

## Output conventions

- `--output` is TTY-aware: `table` in a terminal, minified `json` when piped. It
  already defaults to `json` for these commands — don't force the flag just to parse.
- `--pretty` is JSON-only.
- Data goes to stdout, diagnostics to stderr, so `2>/dev/null` is safe when piping.
- List commands take `--limit` (1–200), `--next`, and `--paginate`. Use `--paginate`
  only when the user actually wants every page.

## Scripting pattern

```bash
RUN_ID=$(asc xcode-cloud run --app "$APP" --workflow CI --branch main | jq -r '.buildRunID')
asc xcode-cloud status --run-id "$RUN_ID" --wait --timeout 1h
```

Confirm the JSON key with a real invocation before depending on it; response shapes
are the CLI's, not Apple's, and can change between releases.

## Reference

- [`asc` CLI](https://github.com/rorkai/App-Store-Connect-CLI) — unofficial, MIT
- [Xcode Cloud documentation](https://developer.apple.com/documentation/xcode/xcode-cloud)
- [Xcode Cloud pricing](https://developer.apple.com/xcode-cloud/) — check the current
  tiers here rather than quoting remembered numbers
