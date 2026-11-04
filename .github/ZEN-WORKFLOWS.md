# Zen-Technology fork: CI/CD guide

This fork tracks [openai/codex-security](https://github.com/openai/codex-security).
Upstream workflow files are left untouched so syncing never conflicts.
Zen-specific automation lives in `zen-*.yml` files only.

## One-time setup

1. **Enable Actions**: Actions tab, then *I understand my workflows, go ahead and enable them*.
2. **Sync token**: create a fine-grained PAT (resource owner `Zen-Technology`, this repo, **Contents** and **Workflows** read/write) and save it as the secret **`ZEN_FORK_SYNC_TOKEN`**.
3. Run **Actions > Zen fork housekeeping > Run workflow** once.
4. Optional, to scan other Zen repositories: add an OpenAI API key as the **organization** secret **`CODEX_SECURITY_API_KEY`**, scoped to the repositories you want scanned.

## Zen workflows

| Workflow | Trigger | Purpose |
| --- | --- | --- |
| `zen-codex-security-scan.yml` | `workflow_call` | **Reusable** Codex Security scan for any Zen repository (see below) |
| `zen-codex-security-selftest.yml` | PRs touching the scan workflow or the Action, manual | Runs the reusable scan in `dry-run` mode: checks the Action pin, sandbox setup and config, with no API key and no cost |
| `zen-upstream-sync.yml` | daily 05:23 UTC, manual | Syncs `main` with upstream through GitHub's *Sync fork* API |
| `zen-fork-housekeeping.yml` | daily 05:53 UTC, manual | Disables upstream-only release/bot workflows (list below) |

## Scanning Zen repositories with Codex Security

Add this file to any Zen-Technology repository as
`.github/workflows/codex-security.yml`:

```yaml
name: Codex Security

on:
  pull_request:
    types: [opened, synchronize, reopened, ready_for_review]
  schedule:
    - cron: "23 7 * * 1"   # weekly full scan
  workflow_dispatch:

permissions:
  contents: read

concurrency:
  group: codex-security-${{ github.event.pull_request.number || github.ref }}
  cancel-in-progress: true

jobs:
  codex-security:
    # pin to a reviewed commit of the fork, never a branch
    uses: Zen-Technology/codex-security/.github/workflows/zen-codex-security-scan.yml@<40-char-sha>
    permissions:
      contents: read
      pull-requests: read
      security-events: write   # SARIF upload to code scanning
      actions: read
    with:
      fail-on-severity: high   # PRs fail on high/critical findings
      max-cost: "5"            # USD guardrail per scan
    secrets:
      CODEX_SECURITY_API_KEY: ${{ secrets.CODEX_SECURITY_API_KEY }}
```

Behaviour:

* **Pull requests** get a `diff` scan of only the changed code. **Schedule, push and manual** runs get a full `repository` scan.
* Fork PRs, Dependabot PRs and drafts are skipped, so the API key is never exposed to untrusted code.
* Completed findings go to **Security > Code scanning**, in the `codex-security-diff` and `codex-security-repository` categories. Private repositories need GitHub Code Security for SARIF upload. Otherwise set `upload-sarif: false` and `upload-artifacts: true`.
* Outputs `scan-status`, `critical-count`, `high-count` and `estimated-cost` are available to later jobs.

| Input | Default | Notes |
| --- | --- | --- |
| `scope` | auto | `repository` or `diff` |
| `paths` | | newline-separated folders/files (repository scope) |
| `mode` | `standard` | `deep` for long repository reviews |
| `model` | CLI default | |
| `effort` | `high` | |
| `cyber-access-program` | CLI default | `standard`, `daybreak_blue` or `daybreak_red` if your API project has access |
| `fail-on-severity` | `none` | `low` / `medium` / `high` / `critical` |
| `max-cost` | `10` | estimated USD stop threshold, empty for no limit |
| `upload-sarif` | `true` | |
| `upload-artifacts` | `false` | reports include source excerpts |
| `dry-run` | `false` | validate setup without a key |
| `timeout-minutes` | `60` | |

**Pinning:** the reusable workflow pins the Action to a reviewed commit of this
fork (`ACTION_PIN` in `zen-codex-security-scan.yml`). After `zen-upstream-sync`
brings in new upstream commits and you have reviewed them, bump that SHA. Then
bump the `@<sha>` that callers use.

## What upstream workflows do in this fork

| Workflow | In the fork |
| --- | --- |
| `node-ci.yml` (+ `native-*`, `container-validate`, `workflow-quality`, `action-ci`, `test-quality`) | **Runs** on PRs, pushes to `main` and daily: native builds (Linux glibc/musl, macOS, Windows), SDK tests, plugin and MCP tests, container validation, workflow lint |
| `published-install-smoke.yml` | **Runs** daily: installs the published npm package |
| `validate-pr-title.yml`, `invoice-desk.yml`, `wolfi-build.yml` | **Run** on PRs |
| `clean-merged-pr-caches.yml` | **Runs** when PRs merge |
| `node-release*.yml`, `node-github-release.yml`, `release-progress.yml`, `container-release.yml` | Disabled by housekeeping (npm/GHCR publishing, needs OpenAI's tokens) |
| `invoice-desk-scan.yml` | Disabled by housekeeping (uses OpenAI's inference environment). Use `zen-codex-security-scan.yml` instead. |
| `container-ci.yml`, `native-artifacts.yml`, `publish-wolfi.yml` | Manual or called by other workflows only |

## Verified locally

- `pnpm --dir sdk/typescript run types`: generated schemas/models in sync, mcp-app typecheck, `tsc --noEmit`
- `pnpm --dir sdk/typescript run format`: Prettier clean
- `pnpm --dir sdk/typescript run build:ci`: OK
- `pnpm --dir sdk/typescript run audit:prod`: no known vulnerabilities
- Rust native primitives: `cargo fmt --check`, `cargo clippy -D warnings`, `node native/build.mjs` (linux-x64-gnu) and `native/proof.mjs`: OK
- The bun test suite needs the **universal** native payload (all OS/arch builds), which only `native-artifacts.yml` produces. CI runs it.
