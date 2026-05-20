# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

The org-level `.github` repo for `inference-gateway`. It holds three things only:

1. `profile/README.md` - rendered on the GitHub org page.
2. `.github/ISSUE_TEMPLATE/` - org-default issue templates that apply to any downstream repo without local templates.
3. `.github/workflows/` - cross-repo orchestrators that drive downstream repos. **The substance of this repo is these workflows.** Two families:
   - **Sync orchestrators** (`sync-sdks.yml`, `sync-adks.yml`) - audit downstream repos against canonical specs in `inference-gateway/schemas` and **file structured drift issues**.
   - **Agent orchestrators** (`bump-adl.yml`, `trigger-cd.yml`) - mechanical fan-out across `kind: agent` targets. `bump-adl.yml` regenerates an agent against a new ADL CLI version and **opens a PR** per target; `trigger-cd.yml` dispatches each agent's own `cd.yml` to cut releases.

There is no build, no test suite, no lint step - this is a config + YAML repo. Validate with `yq` / `gh workflow run` against the actual workflows; do not invent commands.

## How the orchestrators are wired

`repos.yaml` is the single registry that drives every orchestrator via a job matrix. Each entry has a `kind` (`sdk`, `docs`, `adk`, or `agent`) that routes the row to the right workflow. Adding or removing a target = one PR to `repos.yaml`.

`kind` → workflow:
- `sdk`, `docs` → `sync-sdks.yml`
- `adk` → `sync-adks.yml`
- `agent` → `bump-adl.yml`, `trigger-cd.yml`

Trigger paths:
- `inference-gateway/schemas` push to `main` → `repository_dispatch` (`spec-updated` for OpenAPI, `a2a-spec-updated` for A2A) → sync orchestrator fans out per matching target.
- `gh workflow run sync-sdks.yml --repo inference-gateway/.github` (or `sync-adks.yml`) - manual sync runs.
- `gh workflow run bump-adl.yml --repo inference-gateway/.github -f adl_version=vX.Y.Z` - manual ADL CLI bump across all agents (one PR per agent).
- `gh workflow run trigger-cd.yml --repo inference-gateway/.github` - manual release fan-out: dispatches each agent's own `cd.yml`.

Every orchestrator follows the same auth pattern: mint a GitHub App token (`inference-gateway-maintainer-bot`) scoped to a single target via `BOT_MAINTAINER_APP_ID` + `BOT_MAINTAINER_APP_PRIVATE_KEY` (org secrets). No PATs.

For each matrix target, the **sync** workflows additionally:
1. Check out the target into `./target/`.
2. Pull the canonical spec - for SDK/ADK targets via the target's own `task oas-download` / `task a2a:download-schema` (so `git -C target diff <spec>` exposes drift class D); for `docs` via raw fetch from `schemas`.
3. Run `anthropics/claude-code-action` with a single audit prompt that detects drift and files structured issues on the target.

For each matrix target, **`bump-adl.yml`** additionally:
1. Checks out the target into `./target/`.
2. Installs Flox and `go-task` in the runner.
3. Edits `.flox/env/manifest.toml` to pin the new ADL CLI flake, runs `flox upgrade adl` to refresh `manifest.lock`, then `flox activate -- task generate`.
4. Patches `.adl-ignore`'d and hand-maintained files with a small `sed` pass for version-string stragglers (excluding `CHANGELOG.md` and `manifest.lock`).
5. Asserts `agent.yaml` is unchanged and opens a PR titled `chore(deps): bump ADL CLI to vX.Y.Z` on branch `bot/bump-adl-cli-vX.Y.Z`.

For each matrix target, **`trigger-cd.yml`** simply calls `gh workflow run cd.yml --repo inference-gateway/<agent> --ref <ref>` and exits. It is fire-and-forget - it does not wait for the dispatched release to finish.

## Hard invariants - sync workflows (`sync-sdks.yml`, `sync-adks.yml`)

These are load-bearing. If you change a sync-workflow prompt, preserve them. They do **not** apply to the agent orchestrators below.

- **Issues only.** Never open PRs, never modify any file under `./target/`, never push.
- **No `@claude` mention** anywhere in issue titles, bodies, or comments - that would re-trigger downstream Claude automation. Issues are notifications for a human.
- **One issue per drift class per run.** Use the **exact** stable titles in the workflow's drift table - idempotency relies on byte-exact title match in `gh issue list --search 'in:title "<title>"'`.
- **Issue Type must be set via GraphQL.** `gh issue create/edit` has no `--type` flag; the template `type:` frontmatter is only honored through the GitHub UI. After every create/edit, call the `updateIssueIssueType` mutation. If the org doesn't have that type defined yet, log a warning and continue - don't fail the run.
- **`docs` target is ASCII-only.** No em dash (`-`, U+2014) or en dash (`–`, U+2013) anywhere - titles, bodies, comments, footer. Use `-` (U+002D). The repo runs CI that forbids these.
- **Proxy operations are gateway-internal - fully exempt from drift detection.** The five `proxyGet/Post/Put/Delete/Patch` ops under `/proxy/{provider}/{path}` are pass-through helpers, not SDK or docs surface. Never flag them in classes A (missing operations), C (missing examples), or E (missing docs), regardless of whether a `proxy_request` helper exists. Class B (types) and D (vendored spec) are unaffected - proxy schemas remain part of the spec.
- **Drift labels must pre-exist on each target.** `sdk-drift` on `kind: sdk|docs` repos, `adk-drift` on `kind: adk` repos. The orchestrator does not create labels.

## Hard invariants - agent orchestrators (`bump-adl.yml`, `trigger-cd.yml`)

The PR-opening deviation from the sync workflows is **deliberate**: ADL CLI bumps are mechanical and reviewable, unlike drift remediation which needs human judgment. Keep these load-bearing rules:

- **Never modify `agent.yaml`** in a bump PR - that file is user-authored. `bump-adl.yml` has an explicit assertion step; if regeneration touches it, fail the run loudly.
- **Never modify `CHANGELOG.md`** - it is a historical record of past releases. The version-string `sed` pass in `bump-adl.yml` explicitly excludes it.
- **`flox upgrade adl`, not bare `flox upgrade`** - scope the lock refresh strictly to the ADL flake; do not let unrelated package constraints drift in the same PR.
- **Idempotent on same version.** If `inputs.adl_version` already matches the pin in `manifest.toml`, the job short-circuits with no PR. Re-running the same dispatch must be a no-op.
- **Branch name is fixed**: `bot/bump-adl-cli-vX.Y.Z`. Force-push with `--force-with-lease` so concurrent dispatches lose cleanly; the `concurrency:` block serializes per-agent.
- **`trigger-cd.yml` is fire-and-forget.** It does not poll for cd.yml outcomes. If you need wait-for-success semantics, add it deliberately - silent retries are worse than visible misses.
- **App permissions on the `inference-gateway-maintainer-bot` installation** must include `contents: write`, `pull-requests: write`, `workflows: write`, and `actions: write` (the last enables `gh workflow run` on downstream repos). `workflows: write` is needed because `task generate` rewrites `.github/workflows/{ci,cd}.yml`.

## Drift class → title → type → labels

This table is duplicated in each workflow's prompt. Keep both copies in sync if you change one:

SDK (`sync-sdks.yml`, `kind: sdk`):
- A: `[FEATURE] Implement missing OpenAPI operations` - `feature` - `enhancement,sdk-drift`
- B: `[TASK] Refactor regenerate models from latest spec` - `task` - `refactor,sdk-drift`
- C: `[FEATURE] Add usage examples for missing operations` - `feature` - `enhancement,sdk-drift`
- D: `[TASK] Refactor sync vendored openapi.yaml with schemas` - `task` - `refactor,sdk-drift`

Docs (`sync-sdks.yml`, `kind: docs`):
- E: `[DOCS] Document missing operations and schemas` - `documentation` - `documentation,sdk-drift`

ADK (`sync-adks.yml`, `kind: adk`):
- A: `[FEATURE] Implement missing A2A JSON-RPC methods` - `feature` - `enhancement,adk-drift`
- B: `[TASK] Refactor regenerate A2A types from latest schema` - `task` - `refactor,adk-drift`
- C: `[FEATURE] Add usage examples for missing A2A methods` - `feature` - `enhancement,adk-drift`
- D: `[TASK] Refactor sync vendored schema.yaml with schemas` - `task` - `refactor,adk-drift`

## Common edits

- **Add/remove a target:** edit `repos.yaml` only. The matrix step (`yq -o=json -I=0 '[.targets[] | select(.kind == "...")]' repos.yaml`) routes by `kind`.
- **Change the audit logic (sync workflows):** edit the `prompt:` block of the relevant workflow. Preserve the invariants above and the exact stable titles. Pinned model is `claude-opus-4-7`.
- **Bump the action version:** `anthropics/claude-code-action@<ver>`, `actions/checkout@<ver>`, `actions/create-github-app-token@<ver>`, `arduino/setup-task@<ver>`, and `flox/install-flox-action@<ver>` are pinned to specific tags - keep them pinned, don't move to `@v1` floats.
- **Add a new drift class:** update the drift table in the workflow prompt **and** the matching summary in `README.md`'s "Drift classes detected" section. Make sure the corresponding label exists on every target before merge.
- **Add a new issue type (e.g. `documentation`):** the org Issue Types must be created in the GitHub UI first; the GraphQL lookup in the prompt resolves IDs at run time and warns-but-continues if missing.
- **Bump ADL CLI across all agents:** run `gh workflow run bump-adl.yml -f adl_version=vX.Y.Z` (add `-f dry_run=true` first to preview). One PR per agent.
- **Cut releases on all agents:** run `gh workflow run trigger-cd.yml`. Fire-and-forget - watch each agent's Actions tab for outcome.

## Profile and templates

- `profile/README.md` is rendered as the org landing page on GitHub. The ecosystem table there is hand-maintained - when a downstream repo is added or renamed, update the table.
- `.github/ISSUE_TEMPLATE/*.md` are the org-default templates. The `type:` frontmatter only applies when issues are filed through the UI (see invariants above).
