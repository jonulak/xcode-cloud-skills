# Xcode Cloud Skills

Claude Code skills for driving [Xcode Cloud](https://developer.apple.com/xcode-cloud/)
— triggering builds, triaging failures, and writing `ci_scripts/`.

## Why

Xcode Cloud has no YAML file to edit and no CLI of its own. Its automation surface is
the App Store Connect API plus the `ci_scripts/` hooks, which makes it awkward for an
agent to work with unless it knows both halves.

The [`asc` CLI](https://github.com/rorkai/App-Store-Connect-CLI) covers the API half
very well. Its [companion skills repo](https://github.com/rorkai/app-store-connect-cli-skills)
covers 22 App Store Connect workflows — but none of them Xcode Cloud. This fills that gap.

## Install

```
/plugin marketplace add jonulak/xcode-cloud-skills
/plugin install xcode-cloud@jonulak
```

Requires the `asc` CLI for the two API-facing skills:

```bash
brew install asc
asc auth login     # App Store Connect API key
```

`xcode-cloud-scripts` has no dependencies.

## Skills

| Skill | Use when |
|---|---|
| `xcode-cloud-runs` | Start a build, check status, rerun, manage workflows, download artifacts |
| `xcode-cloud-triage` | A build failed and you need to know why |
| `xcode-cloud-scripts` | Writing or debugging `ci_post_clone.sh` / `ci_pre_xcodebuild.sh` / `ci_post_xcodebuild.sh` |

## Notes on accuracy

Two deliberate choices, both learned from getting it wrong elsewhere:

**Commands are verified against `--help`, not the website.** The `asc` published docs
lag the binary. As of `asc` 3.5.1, `run --pull-request-id`, `run --source-run-id`,
`--clean`, `artifacts download`, `build-runs --sort`, and the `view` subcommands were
all real but undocumented. The skills say so and tell the agent to check `--help` first.

**Environment variables come from Apple's own documentation data**, not from
secondary sources. This matters — widely-copied community tables contain a
`CI_APP_STORE_ID` variable that does not exist, and list `CI_XCODEBUILD_ACTION`
values as `build, test, archive` when the real values are `analyze`, `archive`,
`build`, `build-for-testing`, and `test-without-building`. A script matching on
`"test"` never fires.

Compute-hour pricing is deliberately **not** tabulated here — the skills link Apple's
pricing page instead, because stale pricing tables are how the same community skills
ended up understating the tiers threefold.

## License

MIT
