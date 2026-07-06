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
  - **Lifecycle orchestrators** - cross-repo maintenance across `repos.yaml` targets (`stale.yml` sweeps every target except the `kind: none` infra repos, `select(.kind != "none")`; `cleanup-runs.yml` and `backfill-roadmap.yml` sweep every registered target, `select(true)`, including `kind: none`):
    - `stale.yml` marks issues with no activity for 30 days as `stale` and closes them 7 days later.
    - `cleanup-runs.yml` prunes completed workflow runs by conclusion (default `skipped` - the noise `infer-action` / `claude-code-action` leave on every issue event; `skipped,failure` also drops failed-run logs), with an optional `keep_last` per-repo retention floor, from each target's Actions tab, daily.
    - `backfill-roadmap.yml` adds open issues **and open PRs** that are on no project board to the org Roadmap 2026 board (project #7) at Status `Todo`, so work filed outside the `@claude` / `@infer` flow is not lost off the roadmap. PRs are added from any author (humans and bots); only draft PRs are skipped.
  - **Migration (one-shot)** - `migrate-claude.yml` rewrites each repo's `.github/workflows/claude.yml` into a thin caller of the org [reusable `claude.yml`](./.github/workflows/claude.yml), reading its inputs from that entry's `orchestrators.claude` block (`select(.orchestrators.claude != null)`), and in the same PR also bumps the target's own Flox `claude-code` pin (`.flox/env/manifest.toml`) to the latest flox catalog version, isolates `claude-code` in its own `pkg-group = "claude-code"` (so the bumped floor resolves independently of `codex` - a shared `ai` group made the constraints unsatisfiable), and refreshes the lock - note `claude-code` is a flox catalog package (`claude-code.version = "^X.Y.Z"`), not a GitHub release, so the latest comes from `flox show claude-code` (never npm, which can be ahead of the catalog), and there is no config-regen step - so `flox activate -- true` runs explicitly after `flox upgrade claude-code` purely to write `manifest.lock` in its canonical form. Its PR title names the moves - `chore(deps): bump claude-code X -> Y, claude-code-action vA -> vB` (claude-code-action read from the reusable `claude.yml` at the caller's old vs. new ref) - falling back to `ci: centralize claude.yml via reusable workflow`. The generated caller also exposes a `browser` boolean dispatch checkbox (forwarded as `browser: ${{ inputs.browser || <static> }}`); that, a per-repo `repos.yaml` `browser: true`, and a case-insensitive `[browser]` marker in an issue/comment/review all enable the reusable workflow's browser MCP (headless Chromium + `@playwright/mcp`) for that run. `migrate-infer.yml` does the same for `.github/workflows/infer.yml` against the org [reusable `infer.yml`](./.github/workflows/infer.yml), reading that entry's `orchestrators.infer` block (`select(.orchestrators.infer != null)`), and in the same PR also bumps the target's own Flox `infer` pin (`.flox/env/manifest.toml`) to the latest `inference-gateway/cli` release and refreshes the lock - mirroring how `migrate-claude.yml` bumps each repo's `claude-code` pin (`flox upgrade infer` then a bare `flox activate -- true` to canonicalize `manifest.lock`). The migrate-infer PR title names the actual version moves - `chore(deps): bump infer CLI vX -> vY, infer-action vA -> vB` - listing only the component(s) that changed (infer-action is read from the reusable `infer.yml` at the caller's old vs. new ref), and falls back to `ci(infer): centralize infer.yml via reusable workflow` when no version moved. `migrate-codex.yml` does the same for `.github/workflows/codex.yml` against the org [reusable `codex.yml`](./.github/workflows/codex.yml), reading that entry's `orchestrators.codex` block (`select(.orchestrators.codex != null)`), and in the same PR also bumps the target's own Flox `codex` caret pin (`.flox/env/manifest.toml`) to the latest flox catalog version and isolates `codex` in its own `pkg-group = "codex"` - converted from the former standalone `bump-codex.yml` now that `@codex` is a bot. Its PR title is `chore(deps): bump codex X -> Y[, codex-action vA -> vB]`, falling back to `ci(codex): centralize codex.yml via reusable workflow`. All three migrations operate on every repo that defines the matching block. If a target has **no** Flox env (`.flox/env/manifest.toml` absent, e.g. a freshly registered repo), the bump step first `flox init`s one and pins the bot's own CLI (`infer` / `claude-code` / `codex`) at the latest version, so the migration still succeeds (the PR adds a bootstrapped `.flox/`) instead of aborting on the missing manifest.

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

- Each sync orchestrator inlines the org's issue-filing mechanics directly in its prompt - the stable drift titles and labels, the Bug/Feature/Task Issue Type id table, and the `updateIssueIssueType` mutation. No skill is installed; the prompt is self-contained.
- The orchestrator **only files issues**. It never opens PRs, never mentions `@claude`, and never modifies any code on the target repos.
- Issues are notifications. A human reviews each and decides whether to implement, defer, or close.
- One GitHub App (`inference-gateway-maintainer-bot`) provides cross-repo auth - `issues:write` on the target + `contents:read` on the target and `schemas`. No PATs.
- Adding or removing a target is one PR to `repos.yaml` (a single `targets` list). The `kind` field (`sdk`, `docs`, `adk`, `agent`, `none`) routes the row to the right workflow; an optional nested `orchestrators:` block carries the reusable-workflow inputs - `orchestrators.claude` marks the repo for `migrate-claude.yml`, `orchestrators.infer` for `migrate-infer.yml`, `orchestrators.codex` for `migrate-codex.yml`.

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
   │  rank:   jq protect newest keep_last repo-wide (live wfs) -> prune dead wfs in full -> filter conclusions
   │  delete: gh api -X DELETE 'repos/inference-gateway/<target>/actions/runs/<id>'  (re-checked)
   ▼
each target's Actions tab is pruned per the active filters
```

Why these runs exist: `inference-gateway/infer-action` (`@infer`) and `anthropics/claude-code-action` (`@claude`) trigger on every `issues` / `issue_comment` event, then their job hits a job-level `if:` guard and skips when the trigger phrase is absent. GitHub records each as `status: completed, conclusion: skipped` and has no native auto-prune, so they pile up and bury meaningful runs in every Actions tab.

Distinct from `stale.yml` (the other lifecycle orchestrator):

- **Acts on workflow runs, not issues.** Deletes completed runs whose `conclusion` is in the `conclusions` input (default `skipped`; e.g. `skipped,failure` to also drop failed-run logs and shrink the secret-leak surface; `all` for any conclusion). `keep_last` protects the newest N runs **across the repo** (regardless of workflow), so a recent tail survives while the rest is pruned. Lists `?status=completed` so `keep_last` can rank every run, then filters in jq; a per-run re-check re-confirms `status: completed` and `conclusion ∈ set` before each delete.
- **`keep_last` only protects *live* workflows.** The step first lists `actions/workflows` (the IDs whose file still exists) and applies the retention floor only to those. Runs of a **deleted or renamed workflow** (a `workflow_id` no longer backed by a file) are pruned in **full** - otherwise a retired pipeline's history would sit behind `keep_last` forever.
- **Fails loud, resumes idempotently.** All matrix jobs mint from **one** App installation (a single 5,000-req/hr bucket), so a large `conclusions=all` purge across many repos can exhaust it mid-sweep. A rate-limit 403 on the run/workflow listing used to surface as a false `found 0` and a green check (the `mapfile < <(...)` process substitution hid the non-zero exit from `set -e`); it now **fails the job** with an `::error::` and the leftover count. Because deletes are idempotent, just rerun (or wait for the daily cron) to continue - and prefer `-f repository=<name>` to purge one repo at a time within the shared budget.
- **Broader matrix than `stale.yml`.** Resolves `select(true)` - every registered `repos.yaml` target, including the `kind: none` infra repos (`inference-gateway`, `cli`, `operator`, `registry`, ...), which accumulate the same skipped `@claude`/`@infer` runs. This intentionally diverges from `stale.yml` (which exempts `kind: none` so infra issue trackers are not auto-staled). Private repos are never registered in `repos.yaml`, so they are excluded by construction; repos absent from `repos.yaml` (`.github`, `agents`, `awesome-a2a`, `tools`) are untouched.
- **Destructive, so `dry_run` defaults to `true` on manual dispatch.** `gh workflow run cleanup-runs.yml -f dry_run=false` actually deletes; the daily cron deletes by design. A per-target `MAX_DELETES` cap bounds one run and logs a `::warning::` with the leftover count when reached (leftovers handled on the next run).
- **Schedule disabled on landing.** The `schedule:` block ships commented out (as `stale.yml` did); enable it once validated on one repo.

Testing on one repo (AC#3): the `repository` input narrows the matrix to a single target so the sweep can be validated on one Actions tab first:

```sh
# preview one repo (default: skipped-only, no retention)
gh workflow run cleanup-runs.yml --repo inference-gateway/.github -f dry_run=true -f repository=docs
# preview pruning skipped + failed runs while keeping the newest 5 in the repo
gh workflow run cleanup-runs.yml --repo inference-gateway/.github -f dry_run=true -f repository=docs -f conclusions=skipped,failure -f keep_last=5
# delete on that one repo, then check its Actions tab
gh workflow run cleanup-runs.yml --repo inference-gateway/.github -f dry_run=false -f repository=docs
```

### `backfill-roadmap.yml` - add orphan issues and PRs to the Roadmap 2026 board across `repos.yaml`

```
cron: '30 4 * * *'  (daily, disabled until validated; manual via workflow_dispatch, dry_run previews)
   │
   ▼
.github/workflows/backfill-roadmap.yml
   │  reads repos.yaml (select(true) - all registered targets, incl. kind: none and kind: agent), fans out matrix
   ▼
per-target job:
   │  mints a per-target scoped App token (org Projects: read & write rides along)
   │  list:   gh issue list --state open --json projectItems  -> keep issues whose projectItems == [] (on no board)
   │          gh pr list    --state open --json projectItems,isDraft
   │                        -> keep PRs whose projectItems == [] AND not draft (any author, incl. bots)
   │  add:    gh project item-add 7 --owner inference-gateway --url <issue|pr>   (idempotent; --url takes either)
   │  status: gh project item-edit ... --single-select-option-id <Todo>
   ▼
every open issue + non-draft PR that was on no project board is now on Roadmap 2026 (project #7) at Status Todo
```

Why this exists: an issue (or PR) lands on the org Roadmap 2026 board (project #7) only when an `@claude` / `@infer` run touches it (`claude.yml` / `infer.yml` add the worked issue and advance its Status). Issues or PRs filed directly by a human, or before board tracking existed, never reach the board and are invisible to anyone planning off it. This sweep reconciles that gap so planned work spread across repos is never lost off the roadmap.

Distinct from the other lifecycle orchestrators:

- **Adds issues and PRs to a board; never edits or closes them.** For each target it lists **open** issues and **open** PRs on **no** project board and adds them to project #7 at Status `Todo` for a human to triage. It does not touch items already tracked anywhere and never sets `In progress` / `QA` / `Done` (those stay owned by the `@claude` / `@infer` flow and by merges).
- **PRs from any author; only drafts are skipped.** Open PRs are added regardless of author (humans **and** bots - dependabot, the maintainer App, etc.); the only PR filter is `isDraft == false`. Issues are added unfiltered. The board distinguishes issues from PRs natively via the Projects v2 `is:issue` / `is:pr` view filter, so no per-item Type field is written.
- **"On no board" = empty `projectItems`, not "not on #7".** The filter is the item's ProjectsV2 membership (`gh issue list --json projectItems` and `gh pr list --json projectItems,isDraft`, keep length 0 - both `gh` subcommands expose the field). The maintainer App's org-level Projects: read grant lets `projectItems` see every org board, so an item already on #7 *or any other org board* is left alone. There is deliberately no prefetch of #7's items (that would answer the narrower "not on #7" and re-add items tracked elsewhere). Classic Projects (`projectCards`, shut down 2024-2025) is always empty now and is not checked.
- **Broad matrix, like `cleanup-runs.yml`.** Resolves `select(true)` - every registered `repos.yaml` target, including the `kind: none` infra and `kind: agent` repos (orphan issues and PRs can live anywhere). Private repos and repos absent from `repos.yaml` (`.github`, `agents`, `awesome-a2a`, `tools`) are untouched. Open issues and open PRs only.
- **Idempotent, so safe to rerun.** `gh project item-add 7` on an item already on #7 returns the existing item id (exit 0); re-runs are no-ops and the `projectItems` filter shrinks the candidate set each run. A per-target `MAX_ADDS` cap bounds one run and logs a `::warning::` with the leftover count (next run continues). Both the issue and PR listings are rate-limit aware and **fail the job** on a 403 rather than reporting a false `found 0`.
- **`dry_run` defaults to `true` on manual dispatch.** `-f dry_run=false` adds for real; the `schedule:` block (cron `30 4 * * *`, staggered off `stale.yml` and `cleanup-runs.yml` to share the 5,000-req/hr bucket) ships commented out until validated on one repo.

Testing on one repo: the `repository` input narrows the matrix to a single target so the backfill can be validated before it runs fleet-wide:

```sh
# preview one repo (lists the orphan open issues and PRs it would add)
gh workflow run backfill-roadmap.yml --repo inference-gateway/.github -f dry_run=true -f repository=cli
# add that one repo's orphans to project #7 at Status Todo, then check the board
gh workflow run backfill-roadmap.yml --repo inference-gateway/.github -f dry_run=false -f repository=cli
```

## How the `@codex` bot + migration work

`@codex` is the org's third mention-driven bot, alongside `@claude` and `@infer`. Because `openai/codex-action` is a bare `codex exec` runner (it edits the working tree and returns a `final-message`, but never commits, opens PRs, comments, or gets a `gh` token), the reusable `codex.yml` performs every GitHub side-effect itself with the maintainer App token.

### `@codex` at runtime - mention-driven task bot

```
maintainer mentions @codex on an issue (or runs the manual Codex workflow_dispatch form)
   │  thin .github/workflows/codex.yml caller (loop-filtered: no bot actors, non-PR issues)
   ▼
reusable inference-gateway/.github/.github/workflows/codex.yml@<ref>
   │  mints App token, sets git identity, checks out, sets up the language toolchain
   │  board: add issue to Roadmap 2026 (project #7), Status -> In progress   (deterministic gh)
   │  assembles task + bot-instructions `style` into a prompt-file
   │  openai/codex-action (sandbox: workspace-write) edits the working tree
   ▼
if codex changed files:
   │  peter-evans/create-pull-request opens a signed PR (App token), Closes #N
   │  posts codex's final-message as a PR comment
   │  board: Status -> QA
else (question answered / no change):
   │  posts final-message as an issue comment, opens no PR, board stays In progress
```

### `migrate-codex.yml` - deploy the caller + bump the Flox pin

```
maintainer runs: gh workflow run migrate-codex.yml [-f codex_version=X.Y.Z] -f dry_run=false
   │
   ▼
.github/workflows/migrate-codex.yml
   │  matrix job: `flox show codex` -> latest catalog version, resolves the caller ref
   │  (latest .github release tag), reads repos.yaml (select(.orchestrators.codex != null))
   ▼
per-target job (one per repo with an orchestrators.codex block):
   │  mints scoped App token, checks out target into ./target/, installs Flox,
   │  bumps codex.version floor to ^X.Y.Z + isolates codex in its own pkg-group "codex"
   │  (bootstraps a fresh .flox env when absent), `flox upgrade codex` + `flox activate -- true`,
   │  renders .github/workflows/codex.yml as a thin caller (language + orchestrators.codex)
   ▼
peter-evans/create-pull-request opens (or updates) ONE PR carrying the caller + .flox bump,
titled `chore(deps): bump codex X -> Y[, codex-action vA -> vB]`
(fallback `ci(codex): centralize codex.yml via reusable workflow`) on branch `bot/centralize-codex-workflow`
   │
   ▼
maintainer reviews each PR and merges
```

Converted from the former standalone `bump-codex.yml`: codex used to be a local-authoring CLI only, so its bump touched `.flox/env/*` alone. Now that it is also a bot, `migrate-codex.yml` folds the Flox catalog-pin bump into the caller migration, exactly as `migrate-claude.yml` folds in the `claude-code` bump - catalog package, latest from `flox show codex`, `pkg-group` isolation so the raised floor resolves independently of co-resident packages, bare `flox activate -- true` to canonicalize `manifest.lock`. It needs `workflows: write` (it now writes `.github/workflows/codex.yml`), the same scope `migrate-claude` / `migrate-infer` already hold. codex-action ships no semver release (only tags `v1`..`v1.8`), so the codex-action half of the PR title is best-effort. Like the other migrations it opens PRs (mechanical, reviewable) and defaults `dry_run: true`.

## Layout

```
.github/
  actions/
    resolve-targets/          # composite action: repos.yaml + jq select -> matrix
    bot-instructions/         # composite action: shared @claude/@infer/@codex board-tracking + escalation prompt
  ISSUE_TEMPLATE/             # org-default issue templates (feature, refactor, bug, documentation)
  workflows/
    sync-sdks.yml             # SDK + docs audit against OpenAPI (kind: sdk | docs)
    sync-adks.yml             # ADK audit against A2A schema (kind: adk)
    bump-adl.yml              # ADL CLI version bump fan-out (kind: agent)
    refresh-agent-manifest.yml # additive agent.yaml defaults refresh (kind: agent)
    trigger-cd.yml            # release fan-out (kind: agent)
    migrate-claude.yml        # write claude.yml thin caller + bump claude-code Flox pin (select(.orchestrators.claude != null))
    migrate-infer.yml         # write infer.yml thin caller + bump infer Flox pin (select(.orchestrators.infer != null))
    migrate-codex.yml         # write codex.yml thin caller + bump codex Flox pin (select(.orchestrators.codex != null))
    stale.yml                 # stale-issue sweep (select(.kind != "none"))
    cleanup-runs.yml          # prune completed runs by conclusion/retention (select(true) - all registered targets, incl. kind: none)
    backfill-roadmap.yml      # add orphan open issues + non-draft PRs (any author) to Roadmap 2026 / project #7 at Status Todo (select(true))
    claude.yml                # reusable @claude workflow (workflow_call)
    infer.yml                 # reusable @infer workflow (workflow_call)
    codex.yml                 # reusable @codex workflow (workflow_call)
repos.yaml                    # single downstream registry - drives every matrix
profile/                      # GitHub-rendered org profile
```

## Triggering

Once the GitHub App secrets (`INFERENCE_GATEWAY_MAINTAINER_APP_ID`, `INFERENCE_GATEWAY_MAINTAINER_APP_PRIVATE_KEY`), the org-standard `CLAUDE_CODE_OAUTH_TOKEN`, and `OPENAI_API_KEY` (for `@codex`) are provisioned:

Every fan-out workflow below defaults to `dry_run: true` on manual dispatch (safe preview). Add `-f dry_run=false` to act for real and `-f repository=<name>` to run on a single target first (or `-f repository=<name1>,<name2>` for a comma-separated subset).

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
# (also bumps each target's own .flox/env/manifest.toml claude-code pin to the latest flox catalog version)
gh workflow run migrate-infer.yml --repo inference-gateway/.github -f repository=cli                      # dry, one repo
gh workflow run migrate-infer.yml --repo inference-gateway/.github -f dry_run=false                       # open PRs for all
# (bumps each target's own .flox/env/manifest.toml infer pin to the latest inference-gateway/cli release)

# Migration (one-shot) - codex.yml thin caller + Flox codex pin bump:
gh workflow run migrate-codex.yml --repo inference-gateway/.github -f repository=cli                      # dry, one repo
gh workflow run migrate-codex.yml --repo inference-gateway/.github -f dry_run=false                       # open PRs for all (latest codex)

# Lifecycle orchestrators (cron runs for real; manual previews by default):
gh workflow run stale.yml --repo inference-gateway/.github -f dry_run=false                               # sweep for real
gh workflow run cleanup-runs.yml --repo inference-gateway/.github                                               # dry, all targets, skipped-only
gh workflow run cleanup-runs.yml --repo inference-gateway/.github -f dry_run=false -f repository=docs           # delete skipped on one target
gh workflow run cleanup-runs.yml --repo inference-gateway/.github -f conclusions=skipped,failure -f keep_last=5 # dry: prune skipped+failed, keep newest 5 repo-wide
gh workflow run cleanup-runs.yml --repo inference-gateway/.github -f conclusions=all -f keep_last=5             # dry: keep only newest 5 in the repo, prune the rest
gh workflow run backfill-roadmap.yml --repo inference-gateway/.github -f repository=cli                         # dry, one repo: list orphan open issues + PRs
gh workflow run backfill-roadmap.yml --repo inference-gateway/.github                                           # dry, all targets
gh workflow run backfill-roadmap.yml --repo inference-gateway/.github -f dry_run=false -f repository=cli        # add one repo's orphans to project #7 (Status=Todo)
gh workflow run backfill-roadmap.yml --repo inference-gateway/.github -f dry_run=false                          # backfill all targets for real
```

For SDK targets the workflow uses each SDK's own `task oas-download` to pull the canonical `openapi.yaml`. For docs it fetches raw from `inference-gateway/schemas`. For ADK targets the workflow uses each ADK's own `task a2a:download-schema` to pull the canonical `a2a/a2a-schema.yaml`. Both sync workflows always audit against current `main` of `schemas`.

## Pre-merge requirements

Before the orchestrators can run end-to-end, the following pieces need to land separately:

1. **GitHub App** `inference-gateway-maintainer-bot` provisioned and installed on every target listed in `repos.yaml`. Its `INFERENCE_GATEWAY_MAINTAINER_APP_ID` (client ID) and `INFERENCE_GATEWAY_MAINTAINER_APP_PRIVATE_KEY` saved as repo or org secrets, plus `CLAUDE_CODE_OAUTH_TOKEN` for the Claude Code Max subscription auth. The reusable `@claude` and `@infer` workflows (`claude.yml`, `infer.yml`) both mint a maintainer-App token from the `INFERENCE_GATEWAY_MAINTAINER_APP_ID` / `INFERENCE_GATEWAY_MAINTAINER_APP_PRIVATE_KEY` secret pair (the identity the agent's `gh` runs as; the App needs `issues: write`, `contents: write`, `pull-requests: write` on every target it runs on, plus org-level Projects: Read and write - see the per-workflow bullets below). `infer.yml` additionally forwards the provider API-key secrets to `infer-action` (`DEEPSEEK_API_KEY` covers the default `deepseek/deepseek-v4-flash` model; `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, `GOOGLE_API_KEY`, `GROQ_API_KEY`, `MISTRAL_API_KEY`, `CLOUDFLARE_API_KEY`, `COHERE_API_KEY`, `OLLAMA_API_KEY`, `OLLAMA_CLOUD_API_KEY`, `MOONSHOT_API_KEY` enable the others). `infer.yml` also accepts the optional `CLAUDE_CODE_OAUTH_TOKEN` secret (forwarded via `secrets: inherit`); when a maintainer writes `/subscription anthropic/<model>` in the `@infer` prompt or issue/comment body, that run forces the model to the specified Claude model and authenticates via the Claude Code Max subscription (OAuth) instead of `ANTHROPIC_API_KEY` — the directive is case-insensitive and the `anthropic/` prefix is stripped so infer-action receives a bare Claude id (e.g. `claude-sonnet-4-6`) as its subscription mode requires. All reach the reusable workflow via `secrets: inherit`.
2. **App installation permissions** must cover what every orchestrator needs:
   - Sync workflows: `issues: write` on the target + `contents: read` on the target and `schemas`.
   - Reusable `claude.yml` (the `@claude` bot): `contents: write`, `pull-requests: write`, `issues: write` on every target it runs on, **plus Organization permissions -> Projects: Read and write** (org-level, not per-repo) so the bot can add the issue it works on to the Roadmap 2026 board (org project #7) and advance its Status (In progress at start, QA at PR open). The agent's `gh` already runs as the maintainer App (claude-code-action exports the `github_token` input to it as `GH_TOKEN`/`GITHUB_TOKEN`), so only the App's org-Projects grant is needed - no extra token wiring. On a permission failure it files a best-effort, idempotent tracking issue in `inference-gateway/.github` (title `[bot] Missing GitHub permission: <capability>`, label `bot`, deduped org-wide) and continues without aborting the task - so a `bot` label must exist on `inference-gateway/.github`. It also persists Claude Code auto-memory across runs to the private `.memory` repo's `claude` branch (default-on `memory` input; per-repo opt-out `orchestrators.claude.memory: false`; `@infer` uses `main`, so the two agents' differing memory layouts stay isolated on separate branches), so `claude.yml`'s App-token `repositories:` scope now also lists `.memory` - `contents: write` there, already granted since `@infer` writes it.
   - Reusable `infer.yml` (the `@infer` bot): the same `contents: write`, `pull-requests: write`, `issues: write` on every target **plus the org-level Projects: Read and write** grant - and because both bots now mint from the same `INFERENCE_GATEWAY_MAINTAINER_APP_*` maintainer App, that single grant already covers `@infer`. Board tracking + permission-escalation match `claude.yml`, injected via infer-action's `custom-instructions`. One difference: infer's App token is repo-scoped, so its `repositories:` list is widened to the current repo **+ `.github`** so the escalation issue can reach the control repo (board writes ride the org-level grant and need no repo in the list).
   - Reusable `codex.yml` (the `@codex` bot): the same `contents: write`, `pull-requests: write`, `issues: write` on every target **plus the org-level Projects: Read and write** grant - all already covered by the shared maintainer App. Unlike `@claude` / `@infer`, the GitHub side-effects (board, PR, comment) run as **explicit workflow steps** with the App token, not via the action (`openai/codex-action` is a bare `codex exec` runner with no `gh`); it additionally needs the `OPENAI_API_KEY` secret, forwarded via `secrets: inherit`.
   - `bump-adl.yml` / `refresh-agent-manifest.yml`: `contents: write`, `pull-requests: write`, and `workflows: write` (because regeneration rewrites `.github/workflows/{ci,cd}.yml`) on every `kind: agent` target.
   - `trigger-cd.yml`: `actions: write` on every `kind: agent` target (to call `gh workflow run cd.yml`).
   - `migrate-claude.yml`: `contents: write`, `pull-requests: write`, and `workflows: write` (it writes `.github/workflows/claude.yml`) on every target with an `orchestrators.claude` block. Bumping the Flox `claude-code` pin needs only `contents: write` - already covered, no new scope.
   - `migrate-infer.yml`: `contents: write`, `pull-requests: write`, and `workflows: write` (it writes `.github/workflows/infer.yml`) on every target with an `orchestrators.infer` block. Regenerating `.infer/` needs only `contents: write` - already covered, no new scope.
   - `migrate-codex.yml`: `contents: write`, `pull-requests: write`, and `workflows: write` (it writes `.github/workflows/codex.yml`) on every target with an `orchestrators.codex` block. Bumping the Flox `codex` pin needs only `contents: write` - already covered. Same scope `migrate-claude` / `migrate-infer` use, so no new grant.
   - `stale.yml`: `issues: write` (label, comment, close) on every swept target (`select(.kind != "none")`). No new scope beyond what the sync workflows already require.
   - `cleanup-runs.yml`: `actions: write` on every swept target (`select(true)` - every registered target, **including the `kind: none` infra repos** `cli`, `operator`, `inference-gateway`, `registry`, `schemas`, `skills`, `adl`, `adl-cli`, `a2a-debugger`, `infer-action`) to delete workflow runs (deleting skipped, failed, or any conclusion needs no broader scope). The maintainer App is already installed on these repos (the `migrate-*` workflows mint tokens for them) and its `actions: write` permission is installation-wide; `trigger-cd.yml` relies on the same scope on agent targets.
   - `backfill-roadmap.yml`: `issues: read` **and `pull-requests: read`** on every swept target (`select(true)` - every registered target, incl. `kind: none` and `kind: agent`) to list issues and PRs and read their `projectItems`, **plus the org-level Projects: Read and write** grant to add them to project #7 and set Status. All are already provisioned - `issues` access is installation-wide (the sync/stale workflows use it), `pull-requests` is the same grant `claude.yml` / `infer.yml` use to open PRs (write subsumes read), and the org-Projects grant is the same one the bots use - so no new scope and no new secret.
3. **Dispatch workflows in `inference-gateway/schemas`** that fire `repository_dispatch` to this repo:
   - `event_type: spec-updated` whenever `openapi.yaml` changes on `main` (drives `sync-sdks.yml`).
   - `event_type: a2a-spec-updated` whenever `a2a/a2a-schema.yaml` changes on `main` (drives `sync-adks.yml`).
   Until these land, the sync orchestrators only run on manual trigger.
4. **Drift labels** must exist on each sync target before issues file cleanly: `sdk-drift` on every `kind: sdk` / `kind: docs` repo and `adk-drift` on every `kind: adk` repo. `stale.yml` also exempts a `docs-coverage` label (legacy, kept so any historical coverage tickets stay long-lived).
5. **PR labels** `dependencies` and `adl-cli` should exist on every `kind: agent` repo for the bump-adl PRs, and `dependencies`, `codex`, `ci`, `bot` on every `select(.orchestrators.codex != null)` repo for the migrate-codex PRs (the action will create them if the App has permission, but pre-existing is cleaner).

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
