# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

The org-level `.github` repo for the `inference-gateway` GitHub organization. It holds no runtime code — only:

- `profile/README.md` — what GitHub renders on the org page.
- `.github/ISSUE_TEMPLATE/` — org-default templates inherited by any repo without local ones.
- `.github/workflows/` — cross-repo orchestrators that fan out across downstream repos.
- `repos.yaml` — the registry of downstream targets that drives every orchestrator's matrix.

There is no build, no test suite, no application. Changes here are YAML/Markdown. `README.md` is the authoritative narrative for what each workflow does — read it before editing workflows.

## `repos.yaml` is the single source of truth

Every orchestrator reads `repos.yaml`. Adding or removing a downstream repo = one PR editing this file.

`kind` values and which workflows pick them up:

| `kind`     | Picked up by                                                |
| ---------- | ----------------------------------------------------------- |
| `sdk`      | `sync-sdks.yml`, `audit-docs-coverage.yml`, `stale.yml`     |
| `docs`     | `sync-sdks.yml` (separate drift class E), `stale.yml`       |
| `adk`      | `sync-adks.yml`, `audit-docs-coverage.yml`, `stale.yml`     |
| `agent`    | `bump-adl.yml`, `refresh-agent-manifest.yml`, `trigger-cd.yml`, `audit-docs-coverage.yml`, `stale.yml` |

Three `yq` shapes exist; pick the one that matches what your workflow needs:

- `select(.kind == "<kind>")` - `sync-sdks.yml` (`sdk` or `docs`), `sync-adks.yml` (`adk`), and every `kind: agent` orchestrator filter this way.
- `[.targets[]]` (no filter) - `stale.yml` sweeps every target regardless of `kind`.
- `select(.kind != "docs")` - `audit-docs-coverage.yml` inverts the filter to exclude the audit's *output* (`docs`) from the source list. It is the only workflow that does this.

Keep one of these shapes if you add a new workflow; don't invent a fourth.

## Three workflow families, different rules

### Sync orchestrators (`sync-sdks.yml`, `sync-adks.yml`, `audit-docs-coverage.yml`) — issues only

These run `anthropics/claude-code-action` with an audit prompt and file structured drift issues. Hard invariants when editing their prompts:

- **Never open PRs, never modify target code, never mention `@claude`.** Issues are notifications for a human reviewer.
- **Issue titles are load-bearing for idempotency.** Re-runs find existing issues via byte-exact title match (`gh issue list --search 'in:title "<exact title>"'`). Changing a title in the prompt orphans every open issue using the old title — open issues will pile up as duplicates. The title→class→Issue Type→labels table in each prompt is the contract.
- **`gh issue create/edit` has no `--type` flag.** Setting the Issue Type requires the GraphQL `updateIssueIssueType` mutation after creation. The prompt walks through this — preserve that sequence.
- **Proxy endpoints are exempt** from drift classes A, C, E across every SDK and docs target (`proxyGet/Post/Put/Delete/Patch` under `/proxy/{provider}/{path}`). This exemption is explicitly called out as load-bearing because the tracker was repeatedly polluted by false-positive class A issues about these endpoints. `audit-docs-coverage.yml` carries the same exemption.
- **`docs` target is ASCII-only.** No em dashes (`—` U+2014) or en dashes (`–` U+2013) anywhere in titles, bodies, or comments — plain hyphen-minus (`-`) only. Applies to every character emitted for that target. `audit-docs-coverage.yml` always writes to `docs`, so the rule applies to every issue it files, regardless of source kind.

**`audit-docs-coverage.yml` inverts the direction of the other two.** sync-sdks and sync-adks audit many targets against one shared spec and file on the target repo. audit-docs-coverage audits one shared output (`docs`) against many sources and files on `inference-gateway/docs`. Practical consequences when editing its prompt:

- Matrix filter is `select(.kind != "docs")` (excludes the output from the source list), not `select(.kind == "X")`.
- The App token's `repositories:` field lists **both** the source target and `docs`.
- Issue titles MUST embed the fully-qualified source repo name (`[DOCS] Document missing features for inference-gateway/<source>` and `[TASK] Refactor stale documentation for inference-gateway/<source>`) so the byte-exact match stays unique across the parallel matrix writing to one tracker. Changing the source-name segment orphans every open issue using the prior title - same load-bearing constraint as the title→class→type table in sync-sdks / sync-adks.
- The `docs-coverage` label narrows the broad `documentation` / `refactor` labels (same role `sdk-drift` / `adk-drift` play for sync-sdks / sync-adks) so `stale.yml` exempts only the auto-filed coverage tickets, not unrelated human-filed `documentation` issues.
- Trigger is `workflow_dispatch` only (with a `dry_run: boolean` input) — there is no upstream "docs changed" event to react to; coverage gaps drift on the source side.
- Stays out of `sync-sdks.yml` / `sync-adks.yml` territory: spec-level operation gaps (operationIds, JSON-RPC methods, generated types, vendored spec staleness) belong to those workflows; this one audits README / examples / skills / tools narrative coverage only. Duplicate "regenerate types" tickets would collide with the existing long-lived `sdk-drift` / `adk-drift` issues.

### Agent orchestrators (`bump-adl.yml`, `refresh-agent-manifest.yml`, `trigger-cd.yml`) — PRs or dispatch

These open PRs (mechanical, reviewable) or fire `workflow_dispatch` calls. Distinct contracts:

- **`bump-adl.yml` regenerates code and asserts `agent.yaml` is byte-identical afterwards.** Bumping the ADL CLI pin must not change the manifest — if the generator touches `agent.yaml`, the workflow fails by design. The companion workflow is the only legitimate path to modify `agent.yaml`.
- **`refresh-agent-manifest.yml` is the only workflow that edits `agent.yaml`,** and only additively. It snapshots the paths in `PRESERVE_KEYS` (`spec.tools`, `spec.skills`, `spec.config`, `spec.services`), runs an additive deep merge of the fresh `adl init --defaults` scaffold (existing values win), then restores the preserved paths byte-for-byte. Keep the snapshot→merge→restore→validate order intact.
- **`trigger-cd.yml` is fire-and-forget.** It dispatches each agent's own `cd.yml` and exits without waiting. Outcomes are observed on each target's Actions tab.

### Lifecycle orchestrators (`stale.yml`) — write-through to issues across `repos.yaml`

Daily cron + manual `workflow_dispatch` (with `-f dry_run=true` for preview). Distinct contracts:

- **Cannot use `actions/stale`.** That action is hard-coded to `github.context.repo`; there is no input to target a foreign repo. The orchestrator does the equivalent via `gh issue list --search "updated:<… -label:stale …"` → label + comment, then `gh issue list --label stale --search "updated:<…"` → close.
- **Matrix is every target in `repos.yaml` (no `kind` filter).** Out-of-`repos.yaml` org repos (e.g. `inference-gateway`, `cli`, `operator`, `registry`, `awesome-a2a`) are intentionally not swept by this workflow.
- **Defaults: 30 d → mark `stale` → 7 d → close.** Exempt labels: `pinned`, `security`, `sdk-drift`, `adk-drift`, `docs-coverage`. The three drift labels matter - all three sync orchestrators (sync-sdks, sync-adks, audit-docs-coverage) file `[FEATURE] / [TASK] / [DOCS]` tickets that may legitimately sit open while a maintainer triages, so they must never get auto-closed.
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
gh workflow run audit-docs-coverage.yml --repo inference-gateway/.github
gh workflow run audit-docs-coverage.yml --repo inference-gateway/.github -f dry_run=true
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
