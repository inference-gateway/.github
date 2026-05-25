# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

The org-level `.github` repo for the `inference-gateway` GitHub organization. It holds no runtime code — only:

- `profile/README.md` — what GitHub renders on the org page.
- `.github/ISSUE_TEMPLATE/` — org-default templates inherited by any repo without local ones.
- `.github/workflows/` — cross-repo orchestrators that fan out across downstream repos.
- `repos.yaml` — the registry of downstream targets that drives every orchestrator's matrix.

There is no build, no test suite, no application. Changes here are YAML/Markdown. `README.md` is the authoritative narrative for what each workflow does — read it before editing workflows.

## `repos.yaml` is the single source of truth for sync + agent matrices

The sync and agent orchestrators read `repos.yaml` and filter by the `kind` field. Adding or removing a downstream repo from those workflows = one PR editing this file.

`kind` values and which workflows pick them up:

| `kind`  | Picked up by                                                |
| ------- | ----------------------------------------------------------- |
| `sdk`   | `sync-sdks.yml`                                             |
| `docs`  | `sync-sdks.yml` (separate drift class E)                    |
| `adk`   | `sync-adks.yml`                                             |
| `agent` | `bump-adl.yml`, `refresh-agent-manifest.yml`, `trigger-cd.yml` |

The matrix step is always `yq -o=json -I=0 '[.targets[] | select(.kind == "<kind>")]' repos.yaml`. Keep that filter shape if you add new workflows that source from `repos.yaml`.

**Lifecycle orchestrators (`stale.yml`) do not read `repos.yaml`.** Their matrix is `GET /installation/repositories` against the maintainer App, so coverage tracks where the App is installed (every reachable org repo except archived and `.github`). To add or remove a repo from the stale sweep, install or uninstall the App on it - do not edit `repos.yaml`.

## Three workflow families, different rules

### Sync orchestrators (`sync-sdks.yml`, `sync-adks.yml`) — issues only

These run `anthropics/claude-code-action` with an audit prompt and file structured drift issues on the target repo. Hard invariants when editing their prompts:

- **Never open PRs, never modify target code, never mention `@claude`.** Issues are notifications for a human reviewer.
- **Issue titles are load-bearing for idempotency.** Re-runs find existing issues via byte-exact title match (`gh issue list --search 'in:title "<exact title>"'`). Changing a title in the prompt orphans every open issue using the old title — open issues will pile up as duplicates. The title→class→Issue Type→labels table in each prompt is the contract.
- **`gh issue create/edit` has no `--type` flag.** Setting the Issue Type requires the GraphQL `updateIssueIssueType` mutation after creation. The prompt walks through this — preserve that sequence.
- **Proxy endpoints are exempt** from drift classes A, C, E across every SDK and docs target (`proxyGet/Post/Put/Delete/Patch` under `/proxy/{provider}/{path}`). This exemption is explicitly called out as load-bearing because the tracker was repeatedly polluted by false-positive class A issues about these endpoints.
- **`docs` target is ASCII-only.** No em dashes (`—` U+2014) or en dashes (`–` U+2013) anywhere in titles, bodies, or comments — plain hyphen-minus (`-`) only. Applies to every character emitted for that target.

### Agent orchestrators (`bump-adl.yml`, `refresh-agent-manifest.yml`, `trigger-cd.yml`) — PRs or dispatch

These open PRs (mechanical, reviewable) or fire `workflow_dispatch` calls. Distinct contracts:

- **`bump-adl.yml` regenerates code and asserts `agent.yaml` is byte-identical afterwards.** Bumping the ADL CLI pin must not change the manifest — if the generator touches `agent.yaml`, the workflow fails by design. The companion workflow is the only legitimate path to modify `agent.yaml`.
- **`refresh-agent-manifest.yml` is the only workflow that edits `agent.yaml`,** and only additively. It snapshots the paths in `PRESERVE_KEYS` (`spec.tools`, `spec.skills`, `spec.config`, `spec.services`), runs an additive deep merge of the fresh `adl init --defaults` scaffold (existing values win), then restores the preserved paths byte-for-byte. Keep the snapshot→merge→restore→validate order intact.
- **`trigger-cd.yml` is fire-and-forget.** It dispatches each agent's own `cd.yml` and exits without waiting. Outcomes are observed on each target's Actions tab.

### Lifecycle orchestrators (`stale.yml`) — write-through to issues across the whole org

Daily cron + manual `workflow_dispatch` (with `-f dry_run=true` for preview). Distinct contracts:

- **Cannot use `actions/stale`.** That action is hard-coded to `github.context.repo`; there is no input to target a foreign repo. The orchestrator does the equivalent via `gh issue list --search "updated:<… -label:stale …"` → label + comment, then `gh issue list --label stale --search "updated:<…"` → close.
- **Matrix is sourced from the App, not `repos.yaml`.** Coverage = every reachable org repo where the App is installed (minus archived and `.github`). The right way to scope sweeps is via App installations.
- **Defaults: 30 d → mark `stale` → 7 d → close.** Exempt labels: `pinned`, `security`, `sdk-drift`, `adk-drift`. The two drift labels matter - the sync orchestrators file `[FEATURE] / [TASK] / [DOCS]` tickets that may legitimately sit open while a maintainer triages, so they must never get auto-closed.
- **Search-query semantics rely on `updatedAt`.** Adding a label or comment bumps `updatedAt`, so the close step's `updated:<7d ago` filter correctly excludes any issue that received activity (including ours) within the grace window. This is why removal of the `stale` label on re-activity is *not* implemented - the close query already does the right thing.
- **Issues only.** PRs are intentionally untouched.

## Auth model

A single GitHub App, `inference-gateway-maintainer-bot`, mints per-target scoped tokens via `actions/create-github-app-token` in every job. No PATs anywhere. Required secrets:

- `BOT_MAINTAINER_APP_ID` — App client ID
- `BOT_MAINTAINER_APP_PRIVATE_KEY` — App private key
- `CLAUDE_CODE_OAUTH_TOKEN` — for `claude-code-action` (sync workflows only)

Per-workflow App permission requirements are documented in `README.md` under "Pre-merge requirements" — consult it before adding a new orchestrator that needs new scopes.

## Common operations

```sh
# Manual runs (any workflow supports workflow_dispatch):
gh workflow run sync-sdks.yml --repo inference-gateway/.github
gh workflow run sync-adks.yml --repo inference-gateway/.github
gh workflow run bump-adl.yml --repo inference-gateway/.github -f adl_version=vX.Y.Z
gh workflow run bump-adl.yml --repo inference-gateway/.github -f adl_version=vX.Y.Z -f dry_run=true
gh workflow run refresh-agent-manifest.yml --repo inference-gateway/.github -f adl_version=vX.Y.Z
gh workflow run trigger-cd.yml --repo inference-gateway/.github
gh workflow run stale.yml --repo inference-gateway/.github
gh workflow run stale.yml --repo inference-gateway/.github -f dry_run=true

# Validate repos.yaml shape after edits:
yq '.targets[] | .kind' repos.yaml | sort -u   # should only print: adk agent docs sdk
```

The sync workflows are normally driven by `repository_dispatch` from `inference-gateway/schemas` (`event_type: spec-updated` for OpenAPI, `a2a-spec-updated` for the A2A schema). `workflow_dispatch` is the manual path for testing.

## Local dev environment

A Flox env (`.flox/env/manifest.toml`) provides AI CLIs used during local authoring — `infer` (from `inference-gateway/cli`), `claude-code`, `codex`. These are tools for editing this repo, not dependencies of any workflow. The workflows themselves install their tools (`yq`, `gh`, `task`, `flox`) inside the runner.
