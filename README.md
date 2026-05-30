# inference-gateway/.github

Org-level repo holding:

- **Org profile** (`profile/README.md`) - what GitHub renders on the org page.
- **Org-default issue templates** (`.github/ISSUE_TEMPLATE/`) - applied to any repo without local templates.
- **Cross-repo orchestrators** - workflows that fan out across downstream repos listed in [`repos.yaml`](./repos.yaml). Three families:
  - **Sync orchestrators** - audit downstream repos and file structured drift issues for human review:
    - `sync-sdks.yml` audits each SDK and the docs site against the canonical OpenAPI spec in [`inference-gateway/schemas`](https://github.com/inference-gateway/schemas).
    - `sync-adks.yml` audits each ADK against the canonical A2A JSON-RPC schema in [`inference-gateway/schemas`](https://github.com/inference-gateway/schemas).
    - `audit-docs-coverage.yml` audits each SDK, ADK, and agent's user-facing surface (README, examples, skills, tools, config) against the [`inference-gateway/docs`](https://github.com/inference-gateway/docs) site and files `[DOCS]` / `[TASK] Refactor` issues on the docs repo.
  - **Agent orchestrators** - mechanical fan-out across `kind: agent` targets:
    - `bump-adl.yml` bumps every agent's ADL CLI pin to a given version, regenerates, and opens one PR per agent.
    - `trigger-cd.yml` dispatches each agent's own `cd.yml` to cut releases (fire-and-forget).
  - **Lifecycle orchestrators** - cross-repo maintenance across every target in `repos.yaml` (no `kind` filter):
    - `stale.yml` marks issues with no activity for 30 days as `stale` and closes them 7 days later.
    - `cleanup-skipped-runs.yml` deletes `conclusion: skipped` workflow runs (the noise `infer-action` / `claude-code-action` leave on every issue event) from each target's Actions tab, daily.

## How the SDK orchestrator works

```
schemas (openapi.yaml)
   │  push to main → repository_dispatch (event_type: spec-updated)
   ▼
.github/workflows/sync-sdks.yml
   │  reads repos.yaml (kind: sdk | docs), fans out matrix
   ▼
claude-code-action (one per target)
   │  reads spec + target tree, detects drift,
   │  files structured [FEATURE] / [TASK] issues
   ▼
issues on python-sdk / typescript-sdk / rust-sdk / sdk / docs
   │
   ▼
maintainer reviews each issue and decides next steps
```

## How the ADK orchestrator works

```
schemas (a2a/a2a-schema.yaml)
   │  push to main → repository_dispatch (event_type: a2a-spec-updated)
   ▼
.github/workflows/sync-adks.yml
   │  reads repos.yaml (kind: adk), fans out matrix
   ▼
claude-code-action (one per target)
   │  reads schema + target tree, detects drift,
   │  files structured [FEATURE] / [TASK] issues
   ▼
issues on adk / rust-adk (typescript-adk added once the repo exists)
   │
   ▼
maintainer reviews each issue and decides next steps
```

## How the docs-coverage orchestrator works

```
maintainer runs: gh workflow run audit-docs-coverage.yml [-f dry_run=true]
   │
   ▼
.github/workflows/audit-docs-coverage.yml
   │  reads repos.yaml (every kind except docs), fans out matrix
   ▼
claude-code-action (one per source repo)
   │  reads source tree + docs tree, detects coverage gaps,
   │  files structured [DOCS] / [TASK] Refactor issues
   ▼
issues on inference-gateway/docs (NOT on the source repos)
   │
   ▼
maintainer reviews each issue and decides next steps
```

Distinct from `sync-sdks.yml` / `sync-adks.yml`:

- **Inverted direction.** Those audit many targets against one shared spec; this orchestrator audits one shared output (`docs`) against many sources. The matrix filter is `select(.kind != "docs")` to exclude the audit's output from the source list.
- **No `repository_dispatch` trigger.** Coverage gaps drift on the source side, not on a spec push, so there is no upstream event to react to. Maintainer-triggered via `workflow_dispatch` only.
- **Two drift classes only.** A = missing docs (`[DOCS]`, type `documentation`), B = stale docs (`[TASK] Refactor`, type `task`). Spec-level operation gaps belong to `sync-sdks.yml` (class E) and `sync-adks.yml`; this workflow audits narrative coverage (README / examples / skills / tools / config) only.
- **Title contains the source name.** Because all 13 matrix jobs write to the same `docs` tracker, titles must be unique per source: `[DOCS] Document missing features for inference-gateway/<source>` and `[TASK] Refactor stale documentation for inference-gateway/<source>`.
- **Always ASCII.** Output target is always `docs`, so the hyphen-minus rule applies to every emitted character (titles, bodies, comments).

Key invariants for the sync orchestrators (do **not** apply to the agent orchestrators below):

- Each sync orchestrator installs the shared `maintainer` skill from [`inference-gateway/skills`](https://github.com/inference-gateway/skills/tree/main/skills/maintainer) before running `claude-code-action`. Prompts keep only workflow-specific audit rules inline and delegate shared issue filing mechanics to the skill's GitHub Issues reference.
- The orchestrator **only files issues**. It never opens PRs, never mentions `@claude`, and never modifies any code on the target repos.
- Issues are notifications. A human reviews each and decides whether to implement, defer, or close.
- One GitHub App (`inference-gateway-maintainer-bot`) provides cross-repo auth - `issues:write` on the target + `contents:read` on the target and `schemas`. No PATs.
- Adding or removing a target is one PR to `repos.yaml`. The `kind` field (`sdk`, `docs`, `adk`, `agent`) routes the row to the right workflow.

## How the agent orchestrators work

### `bump-adl.yml` - bump every agent's ADL CLI pin

```
maintainer runs: gh workflow run bump-adl.yml -f adl_version=vX.Y.Z
   │
   ▼
.github/workflows/bump-adl.yml
   │  reads repos.yaml (kind: agent), fans out matrix
   ▼
per-agent job (one per agent):
   │  mints scoped App token, checks out target into ./target/,
   │  installs Flox + go-task, edits .flox/env/manifest.toml,
   │  runs `flox upgrade adl` + `flox activate -- task generate`,
   │  patches headers in .adl-ignore'd files, asserts agent.yaml is unchanged
   ▼
peter-evans/create-pull-request opens (or updates) a PR titled
`chore(deps): Bump ADL CLI to vX.Y.Z` on branch `bot/bump-adl-cli-vX.Y.Z`
   │
   ▼
maintainer reviews each PR and merges
```

Deliberate deviation from the sync orchestrators: this workflow **opens PRs**, because ADL CLI bumps are mechanical and reviewable, not judgment calls.

### `trigger-cd.yml` - cut releases on every agent

```
maintainer runs: gh workflow run trigger-cd.yml [-f ref=main]
   │
   ▼
.github/workflows/trigger-cd.yml
   │  reads repos.yaml (kind: agent), fans out matrix
   ▼
per-agent job: gh workflow run cd.yml --repo inference-gateway/<agent> --ref <ref>
   │
   ▼
each agent's own cd.yml runs semantic-release independently
```

Fire-and-forget: this orchestrator does **not** wait for the dispatched releases to finish or report their outcomes back. Watch each agent's Actions tab for outcomes.

## How the lifecycle orchestrators work

### `stale.yml` - mark and close inactive issues across `repos.yaml`

```
cron: '15 3 * * *'  (daily, manual via workflow_dispatch with -f dry_run=true)
   │
   ▼
.github/workflows/stale.yml
   │  reads repos.yaml (all kinds, no filter), fans out matrix
   ▼
per-target job:
   │  mints a per-target scoped App token
   │  step 1: gh issue list updated:<30d -label:stale (minus exempt) → add `stale` label + warning comment
   │  step 2: gh issue list label:stale updated:<7d → close with comment
   ▼
issues across every target in repos.yaml
```

Distinct from the sync/agent orchestrators:

- **No `kind` filter.** Every target in `repos.yaml` is swept regardless of `kind`. Adding or removing a target from the sweep = same single PR to `repos.yaml` that already routes the target to its sync or agent workflow.
- **Cannot use `actions/stale` directly** - that action is hard-coded to operate on `github.context.repo` (the runner's own repo) and has no foreign-repo input. The orchestrator does the equivalent with `gh issue` calls against the matrix target.
- **Exempt labels:** `pinned`, `security` (human override), and `sdk-drift`, `adk-drift`, `docs-coverage` (long-lived drift trackers filed by the sync orchestrators - they may legitimately sit open while a maintainer triages).
- **Dry-run support:** `gh workflow run stale.yml -f dry_run=true` prints the planned mark/close set per repo without modifying anything.
- **Issues only.** PRs are intentionally left alone.

Out-of-scope repos (anything not in `repos.yaml` - e.g. `inference-gateway`, `cli`, `operator`, `registry`, `awesome-a2a`) are not swept by this workflow; they should either be added to `repos.yaml` or keep their own per-repo `stale.yml`.

### `cleanup-skipped-runs.yml` - delete skipped workflow runs across `repos.yaml`

```
cron: '0 6 * * *'  (daily, disabled until validated; manual via workflow_dispatch, dry_run previews)
   │
   ▼
.github/workflows/cleanup-skipped-runs.yml
   │  reads repos.yaml (all kinds, no filter), fans out matrix
   ▼
per-target job:
   │  mints a per-target scoped App token (actions: write)
   │  list:   gh api 'repos/inference-gateway/<target>/actions/runs?status=skipped' --paginate
   │  delete: gh api -X DELETE 'repos/inference-gateway/<target>/actions/runs/<id>'  (skipped-only)
   ▼
each target's Actions tab is freed of skipped runs
```

Why these runs exist: `inference-gateway/infer-action` (`@infer`) and `anthropics/claude-code-action` (`@claude`) trigger on every `issues` / `issue_comment` event, then their job hits a job-level `if:` guard and skips when the trigger phrase is absent. GitHub records each as `status: completed, conclusion: skipped` and has no native auto-prune, so they pile up and bury meaningful runs in every Actions tab.

Distinct from `stale.yml` (the other lifecycle orchestrator):

- **Acts on workflow runs, not issues.** Deletes only runs with `conclusion: skipped` - a server-side `?status=skipped` filter plus a per-run re-check immediately before each delete. `cancelled` / `failure` runs are left untouched.
- **Same matrix, same exclusions.** Reads `[.targets[]]` from `repos.yaml` exactly like `stale.yml`, so out-of-`repos.yaml` repos (`inference-gateway`, `cli`, `operator`, `registry`, ...) are intentionally not swept.
- **Destructive, so `dry_run` defaults to `true` on manual dispatch.** `gh workflow run cleanup-skipped-runs.yml -f dry_run=false` actually deletes; the daily cron deletes by design.
- **Schedule disabled on landing.** The `schedule:` block ships commented out (as `stale.yml` did); enable it once validated on one repo.

Testing on one repo (AC#3): the `repository` input narrows the matrix to a single target so the sweep can be validated on one Actions tab first:

```sh
# preview one repo
gh workflow run cleanup-skipped-runs.yml --repo inference-gateway/.github -f dry_run=true -f repository=docs
# delete on that one repo, then check its Actions tab
gh workflow run cleanup-skipped-runs.yml --repo inference-gateway/.github -f dry_run=false -f repository=docs
```

## Layout

```
.github/
  ISSUE_TEMPLATE/             # org-default issue templates (feature, refactor, bug, documentation)
  workflows/
    sync-sdks.yml             # SDK + docs audit against OpenAPI (kind: sdk | docs)
    sync-adks.yml             # ADK audit against A2A schema (kind: adk)
    audit-docs-coverage.yml   # docs site audit against all sources (kind: != docs)
    bump-adl.yml              # ADL CLI version bump fan-out (kind: agent)
    trigger-cd.yml            # release fan-out (kind: agent)
    stale.yml                 # stale-issue sweep (every target in repos.yaml)
    cleanup-skipped-runs.yml  # delete conclusion: skipped runs (every target in repos.yaml)
repos.yaml                    # downstream registry - drives every matrix
profile/                      # GitHub-rendered org profile
```

## Triggering

Once the GitHub App secrets (`BOT_MAINTAINER_APP_ID`, `BOT_MAINTAINER_APP_PRIVATE_KEY`) and the org-standard `CLAUDE_CODE_OAUTH_TOKEN` are provisioned:

```sh
# Sync orchestrators (audit against current schemas main HEAD):
gh workflow run sync-sdks.yml --repo inference-gateway/.github
gh workflow run sync-adks.yml --repo inference-gateway/.github

# Docs-coverage orchestrator (audit every non-docs source against the docs site):
gh workflow run audit-docs-coverage.yml --repo inference-gateway/.github
gh workflow run audit-docs-coverage.yml --repo inference-gateway/.github -f dry_run=true   # preview only

# Agent orchestrators:
gh workflow run bump-adl.yml --repo inference-gateway/.github -f adl_version=vX.Y.Z
gh workflow run bump-adl.yml --repo inference-gateway/.github -f adl_version=vX.Y.Z -f dry_run=true   # preview only
gh workflow run trigger-cd.yml --repo inference-gateway/.github

# Lifecycle orchestrators:
gh workflow run stale.yml --repo inference-gateway/.github
gh workflow run stale.yml --repo inference-gateway/.github -f dry_run=true   # preview only
gh workflow run cleanup-skipped-runs.yml --repo inference-gateway/.github -f dry_run=true                     # preview all targets
gh workflow run cleanup-skipped-runs.yml --repo inference-gateway/.github -f dry_run=true -f repository=docs   # preview one repo
gh workflow run cleanup-skipped-runs.yml --repo inference-gateway/.github -f dry_run=false -f repository=docs  # delete one repo (test)
gh workflow run cleanup-skipped-runs.yml --repo inference-gateway/.github -f dry_run=false                    # delete across all targets
```

For SDK targets the workflow uses each SDK's own `task oas-download` to pull the canonical `openapi.yaml`. For docs it fetches raw from `inference-gateway/schemas`. For ADK targets the workflow uses each ADK's own `task a2a:download-schema` to pull the canonical `a2a/a2a-schema.yaml`. Both sync workflows always audit against current `main` of `schemas`.

## Pre-merge requirements

Before the orchestrators can run end-to-end, the following pieces need to land separately:

1. **GitHub App** `inference-gateway-maintainer-bot` provisioned and installed on every target listed in `repos.yaml`. Its `BOT_MAINTAINER_APP_ID` (client ID) and `BOT_MAINTAINER_APP_PRIVATE_KEY` saved as repo or org secrets, plus `CLAUDE_CODE_OAUTH_TOKEN` for the Claude Code Max subscription auth.
2. **App installation permissions** must cover what every orchestrator needs:
   - Sync workflows: `issues: write` on the target + `contents: read` on the target and `schemas`.
   - `audit-docs-coverage.yml`: `issues: write` on `inference-gateway/docs` + `contents: read` on `inference-gateway/docs` and on every non-docs target in `repos.yaml`. No new scopes beyond what the sync workflows already require.
   - `bump-adl.yml`: `contents: write`, `pull-requests: write`, and `workflows: write` (because regeneration rewrites `.github/workflows/{ci,cd}.yml`) on every `kind: agent` target.
   - `trigger-cd.yml`: `actions: write` on every `kind: agent` target (to call `gh workflow run cd.yml`).
   - `stale.yml`: `issues: write` (label, comment, close) on every target in `repos.yaml`. No new scope beyond what the sync workflows already require.
   - `cleanup-skipped-runs.yml`: `actions: write` on every target in `repos.yaml` (to delete workflow runs). Already granted - `trigger-cd.yml` relies on the same scope on agent targets.
3. **Dispatch workflows in `inference-gateway/schemas`** that fire `repository_dispatch` to this repo:
   - `event_type: spec-updated` whenever `openapi.yaml` changes on `main` (drives `sync-sdks.yml`).
   - `event_type: a2a-spec-updated` whenever `a2a/a2a-schema.yaml` changes on `main` (drives `sync-adks.yml`).
   Until these land, the sync orchestrators only run on manual trigger.
4. **Drift labels** must exist on each sync target before issues file cleanly: `sdk-drift` on every `kind: sdk` / `kind: docs` repo, `adk-drift` on every `kind: adk` repo, and `docs-coverage` on `inference-gateway/docs` (where `audit-docs-coverage.yml` files its issues; `stale.yml` exempts the label to keep these long-lived).
5. **PR labels** `dependencies` and `adl-cli` should exist on every `kind: agent` repo for the bump-adl PRs (the action will create them if the App has permission, but pre-existing is cleaner).

Until these are in place, `workflow_dispatch` lets a maintainer kick any workflow off manually for testing.

## Drift classes detected

### SDK targets (`sync-sdks.yml`, kind: sdk)

- **A. Operation coverage** - `operationId`s without a corresponding public method.
- **B. Generated models / types** - top-level schemas missing from the committed generated-types file.
- **C. README / examples** - operations not demonstrated in `README.md` or `examples/`.
- **D. Vendored spec staleness** - SDK's checked-in `openapi.yaml` diverging from the canonical one.

> Note: the five `proxy*` operations (`proxyGet`/`Post`/`Put`/`Delete`/`Patch`) under `/proxy/{provider}/{path}` are gateway-internal endpoints and are exempt from classes A, C, and E across all targets.

### Docs target (`sync-sdks.yml`, kind: docs)

- **E. Docs coverage** - `operationId`s or schemas not mentioned in `markdown/**` or `app/**`. Filed as `[DOCS] …` with `type: documentation`.

### ADK targets (`sync-adks.yml`, kind: adk)

- **A. JSON-RPC method coverage** - A2A methods (`message/send`, `message/stream`, `tasks/get`, `tasks/list`, `tasks/cancel`, `tasks/pushNotificationConfig/*`, …) without a corresponding public method on the ADK's server / client surface.
- **B. Generated A2A types** - top-level schemas missing from the committed generated-types file (`types/generated_types.go` for the Go ADK, `src/a2a_types.rs` for the Rust ADK).
- **C. README / examples** - JSON-RPC methods not demonstrated in `README.md` or `examples/`.
- **D. Vendored schema staleness** - ADK's checked-in `schema.yaml` diverging from the canonical `a2a/a2a-schema.yaml`.

### Docs-coverage targets (`audit-docs-coverage.yml`, kind: != docs)

Audits each SDK, ADK, and agent's user-facing surface against the `inference-gateway/docs` site. Issues are filed on the docs repo (not on the source) so a docs maintainer triages from a single tracker.

- **A. Missing documentation** - SDK methods / ADK builders / agent skills / tools / config keys present in the source repo's README, examples, or `agent.yaml` but not mentioned in any `docs/*.md` page. Filed as `[DOCS] Document missing features for inference-gateway/<source>` with `type: documentation`.
- **B. Stale documentation** - docs pages describing source-repo behavior that has since changed (signatures, flag names, config keys, etc.). Filed as `[TASK] Refactor stale documentation for inference-gateway/<source>` with `type: task`.

Both classes carry the `docs-coverage` label so `stale.yml` exempts them while maintainers triage. Spec-level coverage (operationIds, JSON-RPC methods, schemas) is intentionally **not** in scope here - that lives in `sync-sdks.yml` class E and `sync-adks.yml`.

Each class maps to one stable issue title and a matching Issue Type (`feature` for sync-sdks/sync-adks A/C, `task` for B/D, `documentation` for sync-sdks E and docs-coverage A, `task` for docs-coverage B); re-runs refresh in place.
