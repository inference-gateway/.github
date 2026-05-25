# inference-gateway/.github

Org-level repo holding:

- **Org profile** (`profile/README.md`) - what GitHub renders on the org page.
- **Org-default issue templates** (`.github/ISSUE_TEMPLATE/`) - applied to any repo without local templates.
- **Cross-repo orchestrators** - workflows that fan out across downstream repos. Three families:
  - **Sync orchestrators** - audit downstream repos against canonical specs in [`inference-gateway/schemas`](https://github.com/inference-gateway/schemas) and file structured drift issues for human review:
    - `sync-sdks.yml` audits each SDK and the docs site against the canonical OpenAPI spec.
    - `sync-adks.yml` audits each ADK against the canonical A2A JSON-RPC schema.
  - **Agent orchestrators** - mechanical fan-out across `kind: agent` targets in [`repos.yaml`](./repos.yaml):
    - `bump-adl.yml` bumps every agent's ADL CLI pin to a given version, regenerates, and opens one PR per agent.
    - `trigger-cd.yml` dispatches each agent's own `cd.yml` to cut releases (fire-and-forget).
  - **Lifecycle orchestrators** - cross-org maintenance, scoped to every repo the maintainer App is installed on (not `repos.yaml`):
    - `stale.yml` marks issues with no activity for 30 days as `stale` and closes them 7 days later.

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

### `stale.yml` - mark and close inactive issues across the org

```
cron: '15 3 * * *'  (daily, manual via workflow_dispatch with -f dry_run=true)
   │
   ▼
.github/workflows/stale.yml
   │  mints an org-installation App token, calls
   │  GET /installation/repositories to enumerate every active repo
   │  the maintainer App can reach (excludes archived + .github itself),
   │  fans out matrix
   ▼
per-repo job:
   │  mints a per-target scoped token
   │  step 1: gh issue list updated:<30d -label:stale (minus exempt) → add `stale` label + warning comment
   │  step 2: gh issue list label:stale updated:<7d → close with comment
   ▼
issues across every reachable repo
```

Distinct from the sync/agent orchestrators:

- **Matrix source is the App's installed repos, not `repos.yaml`.** Installing or uninstalling the App on a repo automatically expands or contracts coverage; no edit here is needed.
- **Cannot use `actions/stale` directly** - that action is hard-coded to operate on `github.context.repo` (the runner's own repo) and has no foreign-repo input. The orchestrator does the equivalent with `gh issue` calls against the matrix target.
- **Exempt labels:** `pinned`, `security` (human override), and `sdk-drift`, `adk-drift` (long-lived drift trackers filed by the sync orchestrators - they may legitimately sit open while a maintainer triages).
- **Dry-run support:** `gh workflow run stale.yml -f dry_run=true` prints the planned mark/close set per repo without modifying anything.
- **Issues only.** PRs are intentionally left alone.

## Layout

```
.github/
  ISSUE_TEMPLATE/    # org-default issue templates (feature, refactor, bug, documentation)
  workflows/
    sync-sdks.yml    # SDK + docs audit (kind: sdk | docs)
    sync-adks.yml    # ADK audit (kind: adk)
    bump-adl.yml     # ADL CLI version bump fan-out (kind: agent)
    trigger-cd.yml   # release fan-out (kind: agent)
    stale.yml        # org-wide stale-issue sweep (every App-installed repo)
repos.yaml           # downstream registry - drives the sync + agent matrices
profile/             # GitHub-rendered org profile
```

## Triggering

Once the GitHub App secrets (`BOT_MAINTAINER_APP_ID`, `BOT_MAINTAINER_APP_PRIVATE_KEY`) and the org-standard `CLAUDE_CODE_OAUTH_TOKEN` are provisioned:

```sh
# Sync orchestrators (audit against current schemas main HEAD):
gh workflow run sync-sdks.yml --repo inference-gateway/.github
gh workflow run sync-adks.yml --repo inference-gateway/.github

# Agent orchestrators:
gh workflow run bump-adl.yml --repo inference-gateway/.github -f adl_version=vX.Y.Z
gh workflow run bump-adl.yml --repo inference-gateway/.github -f adl_version=vX.Y.Z -f dry_run=true   # preview only
gh workflow run trigger-cd.yml --repo inference-gateway/.github

# Lifecycle orchestrators:
gh workflow run stale.yml --repo inference-gateway/.github
gh workflow run stale.yml --repo inference-gateway/.github -f dry_run=true   # preview only
```

For SDK targets the workflow uses each SDK's own `task oas-download` to pull the canonical `openapi.yaml`. For docs it fetches raw from `inference-gateway/schemas`. For ADK targets the workflow uses each ADK's own `task a2a:download-schema` to pull the canonical `a2a/a2a-schema.yaml`. Both sync workflows always audit against current `main` of `schemas`.

## Pre-merge requirements

Before the orchestrators can run end-to-end, the following pieces need to land separately:

1. **GitHub App** `inference-gateway-maintainer-bot` provisioned and installed on every target listed in `repos.yaml`. Its `BOT_MAINTAINER_APP_ID` (client ID) and `BOT_MAINTAINER_APP_PRIVATE_KEY` saved as repo or org secrets, plus `CLAUDE_CODE_OAUTH_TOKEN` for the Claude Code Max subscription auth.
2. **App installation permissions** must cover what every orchestrator needs:
   - Sync workflows: `issues: write` on the target + `contents: read` on the target and `schemas`.
   - `bump-adl.yml`: `contents: write`, `pull-requests: write`, and `workflows: write` (because regeneration rewrites `.github/workflows/{ci,cd}.yml`) on every `kind: agent` target.
   - `trigger-cd.yml`: `actions: write` on every `kind: agent` target (to call `gh workflow run cd.yml`).
   - `stale.yml`: `issues: write` (label, comment, close) + `metadata: read` (implicit, used by `GET /installation/repositories`) on every repo where the App is installed. Coverage tracks where the App is installed, not `repos.yaml` - install it on any extra org repo (e.g. `inference-gateway`, `cli`, `operator`, `registry`, `awesome-a2a`) that should be swept.
3. **Dispatch workflows in `inference-gateway/schemas`** that fire `repository_dispatch` to this repo:
   - `event_type: spec-updated` whenever `openapi.yaml` changes on `main` (drives `sync-sdks.yml`).
   - `event_type: a2a-spec-updated` whenever `a2a/a2a-schema.yaml` changes on `main` (drives `sync-adks.yml`).
   Until these land, the sync orchestrators only run on manual trigger.
4. **Drift labels** must exist on each sync target before issues file cleanly: `sdk-drift` on every `kind: sdk` / `kind: docs` repo, `adk-drift` on every `kind: adk` repo.
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

Each class maps to one stable issue title and a matching Issue Type (`feature` for A/C, `task` for B/D, `documentation` for E); re-runs refresh in place.
