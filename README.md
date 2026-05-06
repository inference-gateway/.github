# inference-gateway/.github

Org-level repo holding:

- **Org profile** (`profile/README.md`) — what GitHub renders on the org page.
- **Org-default issue templates** (`.github/ISSUE_TEMPLATE/`) — applied to any repo without local templates.
- **Cross-repo orchestrator** — a single workflow that audits each SDK and the docs site against the canonical OpenAPI spec at [`inference-gateway/schemas`](https://github.com/inference-gateway/schemas) and files structured drift issues for human review.

## How the orchestrator works

```
schemas (openapi.yaml)
   │  push to main → repository_dispatch (event_type: spec-updated)
   ▼
.github/workflows/sync-sdks.yml
   │  reads repos.yaml, fans out matrix
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

Key invariants:

- The orchestrator **only files issues**. It never opens PRs, never mentions `@claude`, and never modifies any code on the target repos.
- Issues are notifications. A human reviews each and decides whether to implement, defer, or close.
- One GitHub App (`inference-gateway-sync-bot`) provides cross-repo auth — `issues:write` on the target + `contents:read` on the target and `schemas`. No PATs.
- Adding or removing a target is one PR to `repos.yaml`.

## Layout

```
.github/
  ISSUE_TEMPLATE/   # org-default issue templates (feature, refactor, bug)
  workflows/
    sync-sdks.yml   # the only workflow
repos.yaml          # downstream registry — drives the matrix
profile/            # GitHub-rendered org profile
```

## Triggering

Once the GitHub App secrets (`BOT_GH_APP_ID`, `BOT_GH_APP_PRIVATE_KEY`) and the org-standard `CLAUDE_CODE_OAUTH_TOKEN` are provisioned:

```sh
# Manual run against current schemas main HEAD:
gh workflow run sync-sdks.yml --repo inference-gateway/.github
```

For SDK targets the workflow uses each SDK's own `task oas-download` (every SDK has it) to pull the canonical `openapi.yaml`. For docs it fetches raw from `inference-gateway/schemas`. The workflow always audits against current `main` of `schemas`.

## Pre-merge requirements

Before this workflow can run end-to-end, two pieces need to land separately:

1. **GitHub App** `inference-gateway-sync-bot` provisioned and installed on every target listed in `repos.yaml`. Its `BOT_GH_APP_ID` (client ID) and `BOT_GH_APP_PRIVATE_KEY` saved as repo or org secrets, plus `CLAUDE_CODE_OAUTH_TOKEN` for the Claude Code Max subscription auth (matching the convention in the existing per-SDK `claude.yml` workflows).
2. **Dispatch workflow in `inference-gateway/schemas`** that fires `repository_dispatch` to this repo with `event_type: spec-updated` whenever `openapi.yaml` changes on `main`. Until that lands, the orchestrator only runs on manual trigger.

Until then, `workflow_dispatch` lets a maintainer kick the workflow off manually for testing.

## Drift classes detected

For SDK targets:

- **A. Operation coverage** — `operationId`s without a corresponding public method.
- **B. Generated models / types** — top-level schemas missing from the committed generated-types file.
- **C. README / examples** — operations not demonstrated in `README.md` or `examples/`.
- **D. Vendored spec staleness** — SDK's checked-in `openapi.yaml` diverging from the canonical one.

For the docs target:

- **E. Docs coverage** — `operationId`s or schemas not mentioned in `markdown/**` or `app/**`.

Each class maps to one stable issue title; re-runs refresh in place.
