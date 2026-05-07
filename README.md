# inference-gateway/.github

Org-level repo holding:

- **Org profile** (`profile/README.md`) - what GitHub renders on the org page.
- **Org-default issue templates** (`.github/ISSUE_TEMPLATE/`) - applied to any repo without local templates.
- **Cross-repo orchestrators** - two workflows that audit downstream repos against canonical specs in [`inference-gateway/schemas`](https://github.com/inference-gateway/schemas) and file structured drift issues for human review:
  - `sync-sdks.yml` audits each SDK and the docs site against the canonical OpenAPI spec.
  - `sync-adks.yml` audits each ADK against the canonical A2A JSON-RPC schema.

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

Key invariants (apply to both orchestrators):

- The orchestrator **only files issues**. It never opens PRs, never mentions `@claude`, and never modifies any code on the target repos.
- Issues are notifications. A human reviews each and decides whether to implement, defer, or close.
- One GitHub App (`inference-gateway-maintainer-bot`) provides cross-repo auth - `issues:write` on the target + `contents:read` on the target and `schemas`. No PATs.
- Adding or removing a target is one PR to `repos.yaml`. The `kind` field (`sdk`, `docs`, `adk`) routes the row to the right workflow.

## Layout

```
.github/
  ISSUE_TEMPLATE/   # org-default issue templates (feature, refactor, bug, documentation)
  workflows/
    sync-sdks.yml   # SDK + docs audit (kind: sdk | docs)
    sync-adks.yml   # ADK audit (kind: adk)
repos.yaml          # downstream registry - drives both matrices
profile/            # GitHub-rendered org profile
```

## Triggering

Once the GitHub App secrets (`BOT_MAINTAINER_APP_ID`, `BOT_MAINTAINER_APP_PRIVATE_KEY`) and the org-standard `CLAUDE_CODE_OAUTH_TOKEN` are provisioned:

```sh
# Manual run against current schemas main HEAD:
gh workflow run sync-sdks.yml --repo inference-gateway/.github
gh workflow run sync-adks.yml --repo inference-gateway/.github
```

For SDK targets the workflow uses each SDK's own `task oas-download` to pull the canonical `openapi.yaml`. For docs it fetches raw from `inference-gateway/schemas`. For ADK targets the workflow uses each ADK's own `task a2a:download-schema` to pull the canonical `a2a/a2a-schema.yaml`. Both workflows always audit against current `main` of `schemas`.

## Pre-merge requirements

Before either workflow can run end-to-end, the following pieces need to land separately:

1. **GitHub App** `inference-gateway-maintainer-bot` provisioned and installed on every target listed in `repos.yaml`. Its `BOT_MAINTAINER_APP_ID` (client ID) and `BOT_MAINTAINER_APP_PRIVATE_KEY` saved as repo or org secrets, plus `CLAUDE_CODE_OAUTH_TOKEN` for the Claude Code Max subscription auth.
2. **Dispatch workflows in `inference-gateway/schemas`** that fire `repository_dispatch` to this repo:
   - `event_type: spec-updated` whenever `openapi.yaml` changes on `main` (drives `sync-sdks.yml`).
   - `event_type: a2a-spec-updated` whenever `a2a/a2a-schema.yaml` changes on `main` (drives `sync-adks.yml`).
   Until these land, the orchestrators only run on manual trigger.
3. **Drift labels** must exist on each target before issues file cleanly: `sdk-drift` on every `kind: sdk` / `kind: docs` repo, `adk-drift` on every `kind: adk` repo.

Until then, `workflow_dispatch` lets a maintainer kick either workflow off manually for testing.

## Drift classes detected

### SDK targets (`sync-sdks.yml`, kind: sdk)

- **A. Operation coverage** - `operationId`s without a corresponding public method.
- **B. Generated models / types** - top-level schemas missing from the committed generated-types file.
- **C. README / examples** - operations not demonstrated in `README.md` or `examples/`. The five `proxy*` operations (`proxyGet`/`Post`/`Put`/`Delete`/`Patch`) are exempt - they're covered by the verb-collapsed `proxy_request` helper.
- **D. Vendored spec staleness** - SDK's checked-in `openapi.yaml` diverging from the canonical one.

### Docs target (`sync-sdks.yml`, kind: docs)

- **E. Docs coverage** - `operationId`s or schemas not mentioned in `markdown/**` or `app/**`. Filed as `[DOCS] …` with `type: documentation`.

### ADK targets (`sync-adks.yml`, kind: adk)

- **A. JSON-RPC method coverage** - A2A methods (`message/send`, `message/stream`, `tasks/get`, `tasks/list`, `tasks/cancel`, `tasks/pushNotificationConfig/*`, …) without a corresponding public method on the ADK's server / client surface.
- **B. Generated A2A types** - top-level schemas missing from the committed generated-types file (`types/generated_types.go` for the Go ADK, `src/a2a_types.rs` for the Rust ADK).
- **C. README / examples** - JSON-RPC methods not demonstrated in `README.md` or `examples/`.
- **D. Vendored schema staleness** - ADK's checked-in `schema.yaml` diverging from the canonical `a2a/a2a-schema.yaml`.

Each class maps to one stable issue title and a matching Issue Type (`feature` for A/C, `task` for B/D, `documentation` for E); re-runs refresh in place.
