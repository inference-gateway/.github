# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

The org-level `.github` repo for `inference-gateway`. It holds three things only:

1. `profile/README.md` — rendered on the GitHub org page.
2. `.github/ISSUE_TEMPLATE/` — org-default issue templates that apply to any downstream repo without local templates.
3. `.github/workflows/sync-sdks.yml` and `sync-adks.yml` — cross-repo orchestrators that audit downstream repos against canonical specs in `inference-gateway/schemas` and file structured drift issues. **The substance of this repo is these two workflows.**

There is no build, no test suite, no lint step — this is a config + YAML repo. Validate with `yq` / `gh workflow run` against the actual workflows; do not invent commands.

## How the orchestrators are wired

`repos.yaml` is the single registry that drives both workflows via a job matrix. Each entry has a `kind` (`sdk`, `docs`, or `adk`) that routes the row to the right workflow. Adding or removing a target = one PR to `repos.yaml`.

Trigger paths:
- `inference-gateway/schemas` push to `main` → `repository_dispatch` (`spec-updated` for OpenAPI, `a2a-spec-updated` for A2A) → orchestrator fans out per matching target.
- `gh workflow run sync-sdks.yml --repo inference-gateway/.github` (or `sync-adks.yml`) for manual runs.

For each matrix target, the workflow:
1. Mints a GitHub App token (`inference-gateway-maintainer-bot`) scoped to that single target. `BOT_MAINTAINER_APP_ID` + `BOT_MAINTAINER_APP_PRIVATE_KEY` are org/repo secrets. No PATs.
2. Checks out the target into `./target/`.
3. Pulls the canonical spec — for SDK/ADK targets via the target's own `task oas-download` / `task a2a:download-schema` (so `git -C target diff <spec>` exposes drift class D); for `docs` via raw fetch from `schemas`.
4. Runs `anthropics/claude-code-action` with a single audit prompt that detects drift and files structured issues on the target.

## Hard invariants — both workflows

These are load-bearing. If you change a workflow prompt, preserve them:

- **Issues only.** Never open PRs, never modify any file under `./target/`, never push.
- **No `@claude` mention** anywhere in issue titles, bodies, or comments — that would re-trigger downstream Claude automation. Issues are notifications for a human.
- **One issue per drift class per run.** Use the **exact** stable titles in the workflow's drift table — idempotency relies on byte-exact title match in `gh issue list --search 'in:title "<title>"'`.
- **Issue Type must be set via GraphQL.** `gh issue create/edit` has no `--type` flag; the template `type:` frontmatter is only honored through the GitHub UI. After every create/edit, call the `updateIssueIssueType` mutation. If the org doesn't have that type defined yet, log a warning and continue — don't fail the run.
- **`docs` target is ASCII-only.** No em dash (`—`, U+2014) or en dash (`–`, U+2013) anywhere — titles, bodies, comments, footer. Use `-` (U+002D). The repo runs CI that forbids these.
- **Proxy operations exempt from class C.** The five `proxyGet/Post/Put/Delete/Patch` ops are covered by the verb-collapsed `proxy_request` helper and never need per-verb examples.
- **Drift labels must pre-exist on each target.** `sdk-drift` on `kind: sdk|docs` repos, `adk-drift` on `kind: adk` repos. The orchestrator does not create labels.

## Drift class → title → type → labels

This table is duplicated in each workflow's prompt. Keep both copies in sync if you change one:

SDK (`sync-sdks.yml`, `kind: sdk`):
- A: `[FEATURE] Implement missing OpenAPI operations` — `feature` — `enhancement,sdk-drift`
- B: `[TASK] Refactor regenerate models from latest spec` — `task` — `refactor,sdk-drift`
- C: `[FEATURE] Add usage examples for missing operations` — `feature` — `enhancement,sdk-drift`
- D: `[TASK] Refactor sync vendored openapi.yaml with schemas` — `task` — `refactor,sdk-drift`

Docs (`sync-sdks.yml`, `kind: docs`):
- E: `[DOCS] Document missing operations and schemas` — `documentation` — `documentation,sdk-drift`

ADK (`sync-adks.yml`, `kind: adk`):
- A: `[FEATURE] Implement missing A2A JSON-RPC methods` — `feature` — `enhancement,adk-drift`
- B: `[TASK] Refactor regenerate A2A types from latest schema` — `task` — `refactor,adk-drift`
- C: `[FEATURE] Add usage examples for missing A2A methods` — `feature` — `enhancement,adk-drift`
- D: `[TASK] Refactor sync vendored schema.yaml with schemas` — `task` — `refactor,adk-drift`

## Common edits

- **Add/remove a target:** edit `repos.yaml` only. The matrix step (`yq -o=json -I=0 '[.targets[] | select(.kind == "...")]' repos.yaml`) routes by `kind`.
- **Change the audit logic:** edit the `prompt:` block of the relevant workflow. Preserve the invariants above and the exact stable titles. Pinned model is `claude-opus-4-7`.
- **Bump the action version:** `anthropics/claude-code-action@<ver>` and `actions/checkout@<ver>` / `actions/create-github-app-token@<ver>` are pinned to specific tags — keep them pinned, don't move to `@v1` floats.
- **Add a new drift class:** update the drift table in the workflow prompt **and** the matching summary in `README.md`'s "Drift classes detected" section. Make sure the corresponding label exists on every target before merge.
- **Add a new issue type (e.g. `documentation`):** the org Issue Types must be created in the GitHub UI first; the GraphQL lookup in the prompt resolves IDs at run time and warns-but-continues if missing.

## Profile and templates

- `profile/README.md` is rendered as the org landing page on GitHub. The ecosystem table there is hand-maintained — when a downstream repo is added or renamed, update the table.
- `.github/ISSUE_TEMPLATE/*.md` are the org-default templates. The `type:` frontmatter only applies when issues are filed through the UI (see invariants above).
