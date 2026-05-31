# inference-gateway/.github

Org-level repo holding:

- **Org profile** (`profile/README.md`) - what GitHub renders on the org page.
- **Org-default issue templates** (`.github/ISSUE_TEMPLATE/`) - applied to any repo without local templates.
- **Cross-repo orchestrators** - workflows that fan out across downstream repos listed in [`repos.yaml`](./repos.yaml). Every orchestrator resolves its target set through the shared [`.github/actions/resolve-targets`](./.github/actions/resolve-targets) composite action (one `yq` read + a jq `select` + an optional single-target `repository` input), and every fan-out workflow defaults to `dry_run: true` on manual dispatch. The families:
  - **Sync orchestrators** - audit downstream repos and file structured drift issues for human review:
    - `sync-sdks.yml` audits each SDK and the docs site against the canonical OpenAPI spec in [`inference-gateway/schemas`](https://github.com/inference-gateway/schemas).
    - `sync-adks.yml` audits each ADK against the canonical A2A JSON-RPC schema in [`inference-gateway/schemas`](https://github.com/inference-gateway/schemas).
  - **Agent orchestrators** - mechanical fan-out across `kind: agent` targets:
    - `bump-adl.yml` bumps every agent's ADL CLI pin to a given version, regenerates, and opens one PR per agent.
    - `refresh-agent-manifest.yml` additively merges new `adl init` defaults into each agent's `agent.yaml` (one PR per agent).
    - `trigger-cd.yml` dispatches each agent's own `cd.yml` to cut releases (fire-and-forget).
  - **Lifecycle orchestrators** - cross-repo maintenance across `repos.yaml` targets (`stale.yml` sweeps every target except the `kind: none` infra repos, `select(.kind != "none")`; `cleanup-runs.yml` sweeps every registered target, `select(true)`, including `kind: none`):
    - `stale.yml` marks issues with no activity for 30 days as `stale` and closes them 7 days later.
    - `cleanup-runs.yml` prunes completed workflow runs by conclusion (default `skipped` - the noise `infer-action` / `claude-code-action` leave on every issue event; `skipped,failure` also drops failed-run logs), with an optional `keep_last` per-workflow retention floor, from each target's Actions tab, daily.
  - **Migration (one-shot)** - `migrate-claude.yml` rewrites each repo's `.github/workflows/claude.yml` into a thin caller of the org [reusable `claude.yml`](./.github/workflows/claude.yml), reading its inputs from that entry's `orchestrators.claude` block (`select(.orchestrators.claude != null)`); `migrate-infer.yml` does the same for `.github/workflows/infer.yml` against the org [reusable `infer.yml`](./.github/workflows/infer.yml), reading that entry's `orchestrators.infer` block (`select(.orchestrators.infer != null)`). Both operate on every repo that defines the matching block.

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

Key invariants for the sync orchestrators (do **not** apply to the agent orchestrators below):

- Each sync orchestrator installs the shared `maintainer` skill from [`inference-gateway/skills`](https://github.com/inference-gateway/skills/tree/main/skills/maintainer) before running `claude-code-action`. Prompts keep only workflow-specific audit rules inline and delegate shared issue filing mechanics to the skill's GitHub Issues reference.
- The orchestrator **only files issues**. It never opens PRs, never mentions `@claude`, and never modifies any code on the target repos.
- Issues are notifications. A human reviews each and decides whether to implement, defer, or close.
- One GitHub App (`inference-gateway-maintainer-bot`) provides cross-repo auth - `issues:write` on the target + `contents:read` on the target and `schemas`. No PATs.
- Adding or removing a target is one PR to `repos.yaml` (a single `targets` list). The `kind` field (`sdk`, `docs`, `adk`, `agent`, `none`) routes the row to the right workflow; an optional nested `orchestrators:` block carries the reusable-workflow inputs - `orchestrators.claude` marks the repo for `migrate-claude.yml`, `orchestrators.infer` for `migrate-infer.yml`.

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
`chore(deps): bump ADL CLI to vX.Y.Z` on branch `bot/bump-adl-cli-vX.Y.Z`
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
   │  reads repos.yaml (select(.kind != "none")), fans out matrix
   ▼
per-target job:
   │  mints a per-target scoped App token
   │  step 1: gh issue list updated:<30d -label:stale (minus exempt) → add `stale` label + warning comment
   │  step 2: gh issue list label:stale updated:<7d → close with comment
   ▼
issues across every target in repos.yaml
```

Distinct from the sync/agent orchestrators:

- **Filter is `select(.kind != "none")`.** Every product target (`sdk`/`docs`/`adk`/`agent`) is swept; the infra/tooling repos (`kind: none` - `cli`, `operator`, `registry`, `inference-gateway`, `adl`, ...) are intentionally excluded so their issues are not auto-staled. Adding or removing a product target from the sweep = the same single PR to `repos.yaml` that already routes it to its sync or agent workflow.
- **Cannot use `actions/stale` directly** - that action is hard-coded to operate on `github.context.repo` (the runner's own repo) and has no foreign-repo input. The orchestrator does the equivalent with `gh issue` calls against the matrix target.
- **Exempt labels:** `pinned`, `security` (human override), and `sdk-drift`, `adk-drift`, `docs-coverage` (long-lived drift trackers filed by the sync orchestrators - they may legitimately sit open while a maintainer triages).
- **Dry-run support:** defaults to `dry_run: true` on manual dispatch (prints the planned mark/close set per repo without modifying anything); the daily cron sweeps for real. `-f repository=<name>` narrows to one target.
- **Issues only.** PRs are intentionally left alone.

Out-of-scope repos (`kind: none` entries, plus anything not in `repos.yaml` such as `awesome-a2a`) are not swept by this workflow.

### `cleanup-runs.yml` - prune completed workflow runs across `repos.yaml`

```
cron: '0 6 * * *'  (daily, disabled until validated; manual via workflow_dispatch, dry_run previews)
   │
   ▼
.github/workflows/cleanup-runs.yml
   │  reads repos.yaml (select(true) - all registered targets, incl. kind: none), fans out matrix
   ▼
per-target job:
   │  mints a per-target scoped App token (actions: write)
   │  list:   gh api 'repos/inference-gateway/<target>/actions/runs?status=completed' --paginate
   │  rank:   jq group_by(workflow_id) -> drop newest keep_last -> filter conclusions
   │  delete: gh api -X DELETE 'repos/inference-gateway/<target>/actions/runs/<id>'  (re-checked)
   ▼
each target's Actions tab is pruned per the active filters
```

Why these runs exist: `inference-gateway/infer-action` (`@infer`) and `anthropics/claude-code-action` (`@claude`) trigger on every `issues` / `issue_comment` event, then their job hits a job-level `if:` guard and skips when the trigger phrase is absent. GitHub records each as `status: completed, conclusion: skipped` and has no native auto-prune, so they pile up and bury meaningful runs in every Actions tab.

Distinct from `stale.yml` (the other lifecycle orchestrator):

- **Acts on workflow runs, not issues.** Deletes completed runs whose `conclusion` is in the `conclusions` input (default `skipped`; e.g. `skipped,failure` to also drop failed-run logs and shrink the secret-leak surface; `all` for any conclusion). `keep_last` protects the newest N runs **per workflow** (by `workflow_id`, not per repo), so a recent tail of every pipeline survives while the rest is pruned. Lists `?status=completed` so `keep_last` can rank every run, then filters in jq; a per-run re-check re-confirms `status: completed` and `conclusion ∈ set` before each delete.
- **`keep_last` only protects *live* workflows.** The step first lists `actions/workflows` (the IDs whose file still exists) and applies the per-workflow floor only to those. Runs of a **deleted or renamed workflow** (a `workflow_id` no longer backed by a file) are pruned in **full** - otherwise a retired pipeline's history would sit behind `keep_last` forever.
- **Fails loud, resumes idempotently.** All matrix jobs mint from **one** App installation (a single 5,000-req/hr bucket), so a large `conclusions=all` purge across many repos can exhaust it mid-sweep. A rate-limit 403 on the run/workflow listing used to surface as a false `found 0` and a green check (the `mapfile < <(...)` process substitution hid the non-zero exit from `set -e`); it now **fails the job** with an `::error::` and the leftover count. Because deletes are idempotent, just rerun (or wait for the daily cron) to continue - and prefer `-f repository=<name>` to purge one repo at a time within the shared budget.
- **Broader matrix than `stale.yml`.** Resolves `select(true)` - every registered `repos.yaml` target, including the `kind: none` infra repos (`inference-gateway`, `cli`, `operator`, `registry`, ...), which accumulate the same skipped `@claude`/`@infer` runs. This intentionally diverges from `stale.yml` (which exempts `kind: none` so infra issue trackers are not auto-staled). Private repos are never registered in `repos.yaml`, so they are excluded by construction; repos absent from `repos.yaml` (`.github`, `agents`, `awesome-a2a`, `tools`) are untouched.
- **Destructive, so `dry_run` defaults to `true` on manual dispatch.** `gh workflow run cleanup-runs.yml -f dry_run=false` actually deletes; the daily cron deletes by design. A per-target `MAX_DELETES` cap bounds one run and logs a `::warning::` with the leftover count when reached (leftovers handled on the next run).
- **Schedule disabled on landing.** The `schedule:` block ships commented out (as `stale.yml` did); enable it once validated on one repo.

Testing on one repo (AC#3): the `repository` input narrows the matrix to a single target so the sweep can be validated on one Actions tab first:

```sh
# preview one repo (default: skipped-only, no retention)
gh workflow run cleanup-runs.yml --repo inference-gateway/.github -f dry_run=true -f repository=docs
# preview pruning skipped + failed runs while keeping the newest 5 of each workflow
gh workflow run cleanup-runs.yml --repo inference-gateway/.github -f dry_run=true -f repository=docs -f conclusions=skipped,failure -f keep_last=5
# delete on that one repo, then check its Actions tab
gh workflow run cleanup-runs.yml --repo inference-gateway/.github -f dry_run=false -f repository=docs
```

## Layout

```
.github/
  actions/
    resolve-targets/          # composite action: repos.yaml + jq select -> matrix
  ISSUE_TEMPLATE/             # org-default issue templates (feature, refactor, bug, documentation)
  workflows/
    sync-sdks.yml             # SDK + docs audit against OpenAPI (kind: sdk | docs)
    sync-adks.yml             # ADK audit against A2A schema (kind: adk)
    bump-adl.yml              # ADL CLI version bump fan-out (kind: agent)
    refresh-agent-manifest.yml # additive agent.yaml defaults refresh (kind: agent)
    trigger-cd.yml            # release fan-out (kind: agent)
    migrate-claude.yml        # rewrite claude.yml into a thin caller (select(.orchestrators.claude != null))
    migrate-infer.yml         # write infer.yml thin caller (select(.orchestrators.infer != null))
    stale.yml                 # stale-issue sweep (select(.kind != "none"))
    cleanup-runs.yml          # prune completed runs by conclusion/retention (select(true) - all registered targets, incl. kind: none)
    claude.yml                # reusable @claude workflow (workflow_call)
    infer.yml                 # reusable @infer workflow (workflow_call)
repos.yaml                    # single downstream registry - drives every matrix
profile/                      # GitHub-rendered org profile
```

## Triggering

Once the GitHub App secrets (`BOT_MAINTAINER_APP_ID`, `BOT_MAINTAINER_APP_PRIVATE_KEY`) and the org-standard `CLAUDE_CODE_OAUTH_TOKEN` are provisioned:

Every fan-out workflow below defaults to `dry_run: true` on manual dispatch (safe preview). Add `-f dry_run=false` to act for real and `-f repository=<name>` to run on a single target first.

```sh
# Sync orchestrators (audit against current schemas main HEAD; spec-updated events file for real):
gh workflow run sync-sdks.yml --repo inference-gateway/.github                                            # dry, all sdk/docs
gh workflow run sync-sdks.yml --repo inference-gateway/.github -f dry_run=false -f repository=python-sdk  # file for real, one repo
gh workflow run sync-adks.yml --repo inference-gateway/.github

# Agent orchestrators:
gh workflow run bump-adl.yml --repo inference-gateway/.github -f adl_version=vX.Y.Z                       # dry preview
gh workflow run bump-adl.yml --repo inference-gateway/.github -f adl_version=vX.Y.Z -f dry_run=false      # open PRs
gh workflow run refresh-agent-manifest.yml --repo inference-gateway/.github -f adl_version=vX.Y.Z
gh workflow run trigger-cd.yml --repo inference-gateway/.github -f dry_run=false                          # dispatch cd.yml on all agents

# Migration (one-shot):
gh workflow run migrate-claude.yml --repo inference-gateway/.github -f repository=cli                     # dry, one repo
gh workflow run migrate-claude.yml --repo inference-gateway/.github -f dry_run=false                      # open PRs for all
gh workflow run migrate-infer.yml --repo inference-gateway/.github -f repository=cli                      # dry, one repo
gh workflow run migrate-infer.yml --repo inference-gateway/.github -f dry_run=false                       # open PRs for all

# Lifecycle orchestrators (cron runs for real; manual previews by default):
gh workflow run stale.yml --repo inference-gateway/.github -f dry_run=false                               # sweep for real
gh workflow run cleanup-runs.yml --repo inference-gateway/.github                                               # dry, all targets, skipped-only
gh workflow run cleanup-runs.yml --repo inference-gateway/.github -f dry_run=false -f repository=docs           # delete skipped on one target
gh workflow run cleanup-runs.yml --repo inference-gateway/.github -f conclusions=skipped,failure -f keep_last=5 # dry: prune skipped+failed, keep newest 5/workflow
gh workflow run cleanup-runs.yml --repo inference-gateway/.github -f conclusions=all -f keep_last=5             # dry: keep only newest 5/workflow, prune the rest
```

For SDK targets the workflow uses each SDK's own `task oas-download` to pull the canonical `openapi.yaml`. For docs it fetches raw from `inference-gateway/schemas`. For ADK targets the workflow uses each ADK's own `task a2a:download-schema` to pull the canonical `a2a/a2a-schema.yaml`. Both sync workflows always audit against current `main` of `schemas`.

## Pre-merge requirements

Before the orchestrators can run end-to-end, the following pieces need to land separately:

1. **GitHub App** `inference-gateway-maintainer-bot` provisioned and installed on every target listed in `repos.yaml`. Its `BOT_MAINTAINER_APP_ID` (client ID) and `BOT_MAINTAINER_APP_PRIVATE_KEY` saved as repo or org secrets, plus `CLAUDE_CODE_OAUTH_TOKEN` for the Claude Code Max subscription auth. The reusable `@infer` workflow (`infer.yml`) needs its own dedicated GitHub App installed on every `orchestrators.infer` target (`INFER_APP_ID` / `INFER_APP_PRIVATE_KEY`) with `issues: write`, `contents: write`, and `pull-requests: write`, plus the provider API-key secrets it forwards to `infer-action` (`DEEPSEEK_API_KEY` covers the default `deepseek/deepseek-v4-flash` model; `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, `GOOGLE_API_KEY`, `GROQ_API_KEY`, `MISTRAL_API_KEY`, `CLOUDFLARE_API_KEY`, `COHERE_API_KEY`, `OLLAMA_API_KEY`, `OLLAMA_CLOUD_API_KEY`, `MOONSHOT_API_KEY` enable the others). All reach the reusable workflow via `secrets: inherit`.
2. **App installation permissions** must cover what every orchestrator needs:
   - Sync workflows: `issues: write` on the target + `contents: read` on the target and `schemas`.
   - `bump-adl.yml` / `refresh-agent-manifest.yml`: `contents: write`, `pull-requests: write`, and `workflows: write` (because regeneration rewrites `.github/workflows/{ci,cd}.yml`) on every `kind: agent` target.
   - `trigger-cd.yml`: `actions: write` on every `kind: agent` target (to call `gh workflow run cd.yml`).
   - `migrate-claude.yml`: `contents: write`, `pull-requests: write`, and `workflows: write` (it writes `.github/workflows/claude.yml`) on every target with an `orchestrators.claude` block.
   - `migrate-infer.yml`: `contents: write`, `pull-requests: write`, and `workflows: write` (it writes `.github/workflows/infer.yml`) on every target with an `orchestrators.infer` block.
   - `stale.yml`: `issues: write` (label, comment, close) on every swept target (`select(.kind != "none")`). No new scope beyond what the sync workflows already require.
   - `cleanup-runs.yml`: `actions: write` on every swept target (`select(true)` - every registered target, **including the `kind: none` infra repos** `cli`, `operator`, `inference-gateway`, `registry`, `schemas`, `skills`, `adl`, `adl-cli`, `a2a-debugger`, `infer-action`) to delete workflow runs (deleting skipped, failed, or any conclusion needs no broader scope). The maintainer App is already installed on these repos (the `migrate-*` workflows mint tokens for them) and its `actions: write` permission is installation-wide; `trigger-cd.yml` relies on the same scope on agent targets.
3. **Dispatch workflows in `inference-gateway/schemas`** that fire `repository_dispatch` to this repo:
   - `event_type: spec-updated` whenever `openapi.yaml` changes on `main` (drives `sync-sdks.yml`).
   - `event_type: a2a-spec-updated` whenever `a2a/a2a-schema.yaml` changes on `main` (drives `sync-adks.yml`).
   Until these land, the sync orchestrators only run on manual trigger.
4. **Drift labels** must exist on each sync target before issues file cleanly: `sdk-drift` on every `kind: sdk` / `kind: docs` repo and `adk-drift` on every `kind: adk` repo. `stale.yml` also exempts a `docs-coverage` label (legacy, kept so any historical coverage tickets stay long-lived).
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

Each class maps to one stable issue title and a matching Issue Type (`feature` for sync-sdks/sync-adks A/C, `task` for B/D, `documentation` for sync-sdks class E); re-runs refresh in place.
