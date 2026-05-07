# AGENTS.md

This file provides guidance for AI coding agents working with the `inference-gateway/.github` repository. It documents the project's structure, conventions, workflows, and invariants so agents can operate autonomously and correctly.

---

## Project Overview

This is the **org-level `.github` repository** for the `inference-gateway` GitHub organization. It is a **configuration-only repository** -- there is no build step, no test suite, and no lint step. Everything is YAML + Markdown configuration that drives automated cross-repo maintenance.

### What this repo holds

1. **`profile/README.md`** -- The GitHub organization landing page rendered at `github.com/inference-gateway`.
2. **`.github/ISSUE_TEMPLATE/*.md`** -- Org-default issue templates applied to any downstream repo that lacks local templates.
3. **`.github/workflows/sync-sdks.yml` and `sync-adks.yml`** -- Cross-repo orchestrators that audit downstream repos against canonical specs in `inference-gateway/schemas` and file structured drift issues for human review. **These two workflows are the substance of this repo.**

### Key Technologies

| Technology | Purpose |
|---|---|
| **YAML** | Workflow definitions (`*.yml`), target registry (`repos.yaml`) |
| **GitHub Actions** | CI/CD engine for the orchestrator workflows |
| **Claude Code Action** (`anthropics/claude-code-action`) | AI agent that performs the audit logic |
| **yq** | YAML query tool used for reading `repos.yaml` to build job matrices |
| **GitHub CLI (`gh`)** | Issue creation, editing, searching, and GraphQL API calls |
| **Taskfile (`task`)** | Used by downstream targets to pull canonical specs (`task oas-download`, `task a2a:download-schema`) |
| **Flox** | Local development environment manager (pinned to `claude-code 2.1.123`) |
| **GitHub App** (`inference-gateway-maintainer-bot`) | Cross-repo authentication (no PATs) |

---

## Architecture and Structure

### Directory Layout

```
.github/
  ISSUE_TEMPLATE/               # Org-default issue templates
    bug_report.md               #   [BUG] template
    feature_request.md          #   [FEATURE] template
    documentation_request.md    #   [DOCS] template
    refactor_request.md         #   [TASK] Refactor template
  workflows/
    sync-sdks.yml               # SDK + docs audit (kind: sdk | docs)
    sync-adks.yml               # ADK audit (kind: adk)
.flox/                          # Flox development environment
  env/manifest.toml             #   Pins claude-code 2.1.123
profile/
  README.md                     # GitHub org landing page
repos.yaml                      # Downstream target registry (drives both workflow matrices)
CLAUDE.md                       # Guidance for Claude Code (claude.ai/code)
AGENTS.md                       # This file
README.md                       # Project documentation
```

### How the Orchestrators Work

```
schemas (openapi.yaml / a2a/a2a-schema.yaml)
   |  push to main --> repository_dispatch
   v
.github/workflows/sync-*.yml
   |  reads repos.yaml (filtered by kind), fans out matrix
   v
claude-code-action (one per target)
   |  reads spec + target tree, detects drift,
   |  files structured issues on the target repo
   v
issues on downstream repos
   |
   v
maintainer reviews each issue and decides next steps
```

### The `repos.yaml` Registry

This is the single source of truth for downstream targets. Each entry has:

- **`name`**: The GitHub repository name (e.g., `python-sdk`, `docs`, `adk`).
- **`language`**: Human-readable language label (e.g., `Python`, `Go`, `MDX`).
- **`kind`**: Routes the target to the correct workflow:
  - `sdk` -> `sync-sdks.yml`
  - `docs` -> `sync-sdks.yml`
  - `adk` -> `sync-adks.yml`

Adding or removing a target = **one PR to `repos.yaml`** and nothing else.

---

## Development Environment Setup

### Prerequisites

- **Flox** (recommended): The project pins `claude-code 2.1.123`. Activate with:
  ```sh
  flox activate
  ```
- **GitHub CLI (`gh`)**: Needed for workflow dispatch, issue inspection.
- **yq**: For querying YAML files locally.

### No Build/Tests/Lint

There is **no build step, no test suite, no linter configuration** in this repository. Validate changes by:

1. Linting YAML with `yq`:
   ```sh
   yq eval repos.yaml > /dev/null
   ```
2. Running a dry-run workflow dispatch:
   ```sh
   gh workflow run sync-sdks.yml --repo inference-gateway/.github
   gh workflow run sync-adks.yml --repo inference-gateway/.github
   ```

---

## Key Commands

### Workflow Triggers

| Command | Purpose |
|---|---|
| `gh workflow run sync-sdks.yml --repo inference-gateway/.github` | Manual run of the SDK/docs orchestrator |
| `gh workflow run sync-adks.yml --repo inference-gateway/.github` | Manual run of the ADK orchestrator |

### Automatic Triggers

- `inference-gateway/schemas` push to `main` with `openapi.yaml` changes -> `repository_dispatch` with `event_type: spec-updated` -> triggers `sync-sdks.yml`
- `inference-gateway/schemas` push to `main` with `a2a/a2a-schema.yaml` changes -> `repository_dispatch` with `event_type: a2a-spec-updated` -> triggers `sync-adks.yml`

### Common Edits

| Task | What to Edit |
|---|---|
| Add/remove a target | `repos.yaml` only |
| Change audit logic | The `prompt:` block in the relevant workflow YAML |
| Bump action versions | Pinned tags in `sync-sdks.yml` and `sync-adks.yml` (`actions/checkout@v6.0.2`, `anthropics/claude-code-action@v1.0.114`, etc.) |
| Add a new drift class | Workflow prompt + `README.md` "Drift classes detected" section |
| Add a new issue type | Must be created in GitHub UI first; the workflow looks up IDs via GraphQL |
| Update the org profile | `profile/README.md` |
| Update issue templates | `.github/ISSUE_TEMPLATE/*.md` |

---

## Testing Instructions

Since there is no traditional test suite, validation relies on:

1. **YAML validity**: Run `yq eval` on any modified `.yml` or `yaml` file.
2. **Workflow dispatch**: Run a manual `gh workflow run` to exercise the workflow against current `schemas` main.
3. **Pre-merge checklist** (before either workflow can run end-to-end):
   - [ ] GitHub App `inference-gateway-maintainer-bot` is installed on every target in `repos.yaml`.
   - [ ] Secrets are provisioned: `BOT_MAINTAINER_APP_ID`, `BOT_MAINTAINER_APP_PRIVATE_KEY`, `CLAUDE_CODE_OAUTH_TOKEN`.
   - [ ] Dispatch workflows exist in `inference-gateway/schemas` to fire `repository_dispatch`.
   - [ ] Drift labels exist on each target: `sdk-drift` on `kind: sdk|docs` repos, `adk-drift` on `kind: adk` repos.

---

## Drift Classes

### SDK Targets (`sync-sdks.yml`, `kind: sdk`)

| Class | Drift | Issue Title | Type | Labels |
|---|---|---|---|---|
| A | Operation coverage - `operationId`s without a corresponding public method | `[FEATURE] Implement missing OpenAPI operations` | `feature` | `enhancement,sdk-drift` |
| B | Generated models/types - top-level schemas missing from committed generated-types file | `[TASK] Refactor regenerate models from latest spec` | `task` | `refactor,sdk-drift` |
| C | README/examples - operations not demonstrated | `[FEATURE] Add usage examples for missing operations` | `feature` | `enhancement,sdk-drift` |
| D | Vendored spec staleness - checked-in `openapi.yaml` diverges from canonical | `[TASK] Refactor sync vendored openapi.yaml with schemas` | `task` | `refactor,sdk-drift` |

### Docs Target (`sync-sdks.yml`, `kind: docs`)

| Class | Drift | Issue Title | Type | Labels |
|---|---|---|---|---|
| E | Docs coverage - `operationId`s or schemas not mentioned in docs | `[DOCS] Document missing operations and schemas` | `documentation` | `documentation,sdk-drift` |

### ADK Targets (`sync-adks.yml`, `kind: adk`)

| Class | Drift | Issue Title | Type | Labels |
|---|---|---|---|---|
| A | JSON-RPC method coverage - methods without a corresponding public method | `[FEATURE] Implement missing A2A JSON-RPC methods` | `feature` | `enhancement,adk-drift` |
| B | Generated A2A types - top-level schemas missing from committed generated-types | `[TASK] Refactor regenerate A2A types from latest schema` | `task` | `refactor,adk-drift` |
| C | README/examples - methods not demonstrated | `[FEATURE] Add usage examples for missing A2A methods` | `feature` | `enhancement,adk-drift` |
| D | Vendored schema staleness - checked-in `schema.yaml` diverges from canonical | `[TASK] Refactor sync vendored schema.yaml with schemas` | `task` | `refactor,adk-drift` |

---

## Project Conventions and Coding Standards

### Hard Invariants (Both Workflows)

These are **load-bearing** and must never be broken:

1. **Issues only.** The orchestrator never opens PRs, never modifies any file under `./target/`, and never pushes.
2. **No `@claude` mention** anywhere in issue titles, bodies, or comments -- that would re-trigger downstream Claude automation.
3. **One issue per drift class per run.** Exact stable titles from the drift table must be used for idempotency (byte-exact title match for de-dup via `gh issue list --search 'in:title "<title>"'`).
4. **Issue Type set via GraphQL.** `gh issue create/edit` has no `--type` flag, so the `updateIssueIssueType` mutation must be called after every create or edit.
5. **ASCII-only for `docs` target.** No em dash (`--`, U+2014) or en dash (`--`, U+2013) -- use plain hyphen-minus (`-`, U+002D) everywhere.
6. **Proxy operations exempt from class C.** `proxyGet/Post/Put/Delete/Patch` are covered by the verb-collapsed `proxy_request` helper and never need per-verb examples.
7. **Drift labels must pre-exist** on each target. The orchestrator does not create labels.

### Naming Conventions

- **Issue titles**: Follow the `[PREFIX] Description` pattern (`[FEATURE]`, `[TASK]`, `[DOCS]`, `[BUG]`).
- **Workflow names**: Use kebab-case (`sync-sdks.yml`, `sync-adks.yml`).
- **`kind` values**: Lowercase (`sdk`, `docs`, `adk`).
- **`repos.yaml` keys**: `name`, `language`, `kind`.

### Commit Message Style

The project uses [Conventional Commits](https://www.conventionalcommits.org/):

```
<type>: <short description>

feat      New feature or functionality
fix       Bug fix
ci        CI/CD or workflow changes
chore     Maintenance, tooling, configuration
refactor  Code restructuring
docs      Documentation changes
```

### Version Pinning

All GitHub Actions and tools are pinned to specific versions (tags, not floats):

- `actions/checkout@v6.0.2`
- `actions/create-github-app-token@v3.1.1`
- `anthropics/claude-code-action@v1.0.114`
- `arduino/setup-task@v2` (version `3.x`)
- `claude-code` pinned to `2.1.123` via Flox

Never move pinned versions to `@v1`/`@v2` floats.

### Code Organization

- **One workflow file per orchestrator.** Logic is not split across multiple files.
- **`repos.yaml` is the single registry.** No inline target lists in workflow files.
- **Validation is manual** (no CI in this repo itself -- CI lives in downstream repos).

---

## Important Files and Configurations

| File | Purpose | What to Know |
|---|---|---|
| `repos.yaml` | Target registry driving both workflow matrices | Edit this to add/remove targets. `kind` field routes targets to workflows. |
| `.github/workflows/sync-sdks.yml` | SDK + docs orchestrator | Reads `repos.yaml` with `yq` filtering for `kind: sdk\|docs`. Runs `anthropics/claude-code-action`. |
| `.github/workflows/sync-adks.yml` | ADK orchestrator | Reads `repos.yaml` with `yq` filtering for `kind: adk`. Runs `anthropics/claude-code-action`. |
| `.github/ISSUE_TEMPLATE/*.md` | Org-default issue templates | `type:` frontmatter only honored through GitHub UI (not via `gh issue create`). |
| `profile/README.md` | GitHub org landing page | Hand-maintained ecosystem table -- update when repos are added/renamed. |
| `.flox/env/manifest.toml` | Flox development environment | Pins `claude-code 2.1.123`. |
| `CLAUDE.md` | Guidance for Claude Code | Contains the detailed invariants, drift tables, and common edits for the orchestrators. |
| `README.md` | Project documentation | Contains the "Drift classes detected" tables that must stay in sync with workflow prompts. |

### Downstream Repositories Referenced in `repos.yaml`

| Repo | Kind | Language |
|---|---|---|
| `python-sdk` | `sdk` | Python |
| `typescript-sdk` | `sdk` | TypeScript |
| `rust-sdk` | `sdk` | Rust |
| `sdk` | `sdk` | Go |
| `docs` | `docs` | MDX |
| `adk` | `adk` | Go |
| `rust-adk` | `adk` | Rust |

### Generated-types File Locations (for Drift Class B)

| Target | Generated Types File |
|---|---|
| `python-sdk` | `inference_gateway/models.py` |
| `typescript-sdk` | `src/types/generated/index.ts` |
| `rust-sdk` | `src/generated/schemas.rs` |
| `sdk` (Go) | `generated_types.go` |
| `adk` (Go) | `types/generated_types.go` |
| `rust-adk` | `src/a2a_types.rs` |

---

## Agent Workflow Guide

### When making changes

1. **Read `repos.yaml` first** -- it defines all downstream targets and their `kind`.
2. **Read the relevant workflow YAML** -- understand the prompt structure and drift tables.
3. **Read `CLAUDE.md`** -- it contains the detailed invariants and drift class tables.
4. **Keep prompts in sync** -- if you change a drift class in a workflow prompt, update the matching table in `README.md`'s "Drift classes detected" section.
5. **Never float versions** -- always pin to exact tags.
6. **Validate with `yq`** before committing any YAML changes.

### When running workflows for testing

1. Ensure the GitHub App is installed on all relevant targets.
2. Ensure drift labels (`sdk-drift`, `adk-drift`) exist on each target.
3. Run with `gh workflow run` and monitor the run in the GitHub UI.
4. Check that issues filed have correct titles, types, labels, and no `@claude` mentions.

### Adding a new downstream target

1. Add an entry to `repos.yaml` with `name`, `language`, and `kind`.
2. Install the GitHub App `inference-gateway-maintainer-bot` on the new repo.
3. Create the drift label (`sdk-drift` for SDK/docs, `adk-drift` for ADK).
4. Update `profile/README.md` ecosystem table if the repo should appear on the org page.
