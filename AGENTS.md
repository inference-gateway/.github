# Repository Guidelines

## Project Structure & Module Organization

This is the org-level `.github` repository for `inference-gateway`. It contains no runtime application code. Keep changes focused on Markdown, YAML, workflow automation, and static assets.

- `README.md` documents workflow behavior and should stay authoritative.
- `profile/README.md` renders on the GitHub organization profile.
- `.github/ISSUE_TEMPLATE/` holds default issue templates inherited by repos without local templates.
- `.github/workflows/` contains cross-repo orchestrators such as `sync-docs.yml`, `sync-adks.yml`, `bump-adl.yml`, `refresh-agent-manifest.yml`, `trigger-cd.yml`, `migrate-claude.yml`, `migrate-infer.yml`, `stale.yml`, and `cleanup-runs.yml`, plus the reusable (`workflow_call`) workflows: the `claude.yml` / `infer.yml` bot workflows and `schemas-sync.yml`, the deterministic OpenAPI type sync each schemas consumer repo calls from its own thin caller.
- `.github/actions/resolve-targets/` is the shared composite action that turns `repos.yaml` + a `select` expression into a fan-out matrix.
- `repos.yaml` is the single downstream registry used by every workflow matrix (one `targets` list; each entry may carry a nested `orchestrators:` block with `claude` and/or `infer` sub-blocks).
- `assets/` stores org profile assets, including star history SVGs.

## Build, Test, and Development Commands

There is no build or application test suite. Validate changes with targeted checks:

```sh
yq '.targets[] | .kind' repos.yaml | sort -u
gh workflow run sync-docs.yml --repo inference-gateway/.github                                      # dry preview (default)
gh workflow run sync-adks.yml --repo inference-gateway/.github
gh workflow run bump-adl.yml --repo inference-gateway/.github -f adl_version=vX.Y.Z                 # dry preview (default)
gh workflow run bump-adl.yml --repo inference-gateway/.github -f adl_version=vX.Y.Z -f dry_run=false  # open PRs for real
```

Use the `yq` command after editing `repos.yaml`; valid kinds are `adk`, `agent`, `docs`, `none`, and `sdk`. Every fan-out workflow defaults to `dry_run: true` on manual dispatch and accepts `-f repository=<name>` to run on a single target first. Use `gh workflow run` only when repository secrets and GitHub App permissions are available.

## Coding Style & Naming Conventions

Use two-space indentation for YAML. Keep Markdown concise, with descriptive headings and relative links when possible. Preserve existing workflow names and job names. Matrix selection is centralized in the `.github/actions/resolve-targets` composite action — pass it a jq `select` expression (e.g. `'.kind == "agent"'`, `'.orchestrators.claude != null'`, `'.orchestrators.infer != null'`) rather than reintroducing per-workflow `yq` filters. In sync workflow prompts, issue titles are idempotency keys; do not rename them casually.

## Testing Guidelines

For workflow edits, prefer manual dispatch with a narrow dry run: every fan-out workflow defaults `dry_run=true` and takes `-f repository=<name>`, so preview on one target before fanning out. For `repos.yaml`, verify every target has `name`, `kind`, and `language`; entries with an `orchestrators.claude` / `orchestrators.infer` block must set `.language` on each; and the `kind` routes to the intended workflow family (`docs` → sync-docs, `adk` → sync-adks, `agent` → bump-adl/refresh/trigger-cd, `sdk`/`none` → lifecycle + migrations only — the deterministic type sync for the OpenAPI consumers runs via each repo's own `schemas-sync.yml` thin caller of the org reusable workflow, not a `repos.yaml` fan-out).

## Commit & Pull Request Guidelines

Recent history uses conventional-style subjects such as `chore: Add CLAUDE.md file`, `ci: Migrate ADL schema fields to v0.8.0`, and `chore(deps): bump ...`. Follow `type(scope): summary` when useful, with common types `chore`, `ci`, `docs`, and `chore(deps)`.

PRs should explain the operational effect, list any workflow permissions or secrets touched, and link related issues. Include screenshots only for `profile/README.md` or asset changes. For workflow changes, mention the validation command or dispatch run used.

## Security & Configuration Tips

Do not introduce personal access tokens. Workflows use the `inference-gateway-maintainer-bot` GitHub App plus documented secrets. Keep sync orchestrators issue-only: they must not open PRs, modify target code, or mention `@claude`.
