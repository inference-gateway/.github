# Repository Guidelines

## Project Structure & Module Organization

This is the org-level `.github` repository for `inference-gateway`. It contains no runtime application code. Keep changes focused on Markdown, YAML, workflow automation, and static assets.

- `README.md` documents workflow behavior and should stay authoritative.
- `profile/README.md` renders on the GitHub organization profile.
- `.github/ISSUE_TEMPLATE/` holds default issue templates inherited by repos without local templates.
- `.github/workflows/` contains cross-repo orchestrators such as `sync-sdks.yml`, `sync-adks.yml`, `bump-adl.yml`, `refresh-agent-manifest.yml`, `trigger-cd.yml`, and `cleanup-skipped-runs.yml`.
- `repos.yaml` is the downstream registry used by workflow matrices.
- `assets/` stores org profile assets, including star history SVGs.

## Build, Test, and Development Commands

There is no build or application test suite. Validate changes with targeted checks:

```sh
yq '.targets[] | .kind' repos.yaml | sort -u
gh workflow run sync-sdks.yml --repo inference-gateway/.github
gh workflow run sync-adks.yml --repo inference-gateway/.github
gh workflow run bump-adl.yml --repo inference-gateway/.github -f adl_version=vX.Y.Z -f dry_run=true
```

Use the `yq` command after editing `repos.yaml`; valid kinds are `adk`, `agent`, `docs`, and `sdk`. Use `gh workflow run` only when repository secrets and GitHub App permissions are available.

## Coding Style & Naming Conventions

Use two-space indentation for YAML. Keep Markdown concise, with descriptive headings and relative links when possible. Preserve existing workflow names, job names, and matrix filter shape, especially `select(.kind == "...")`. In sync workflow prompts, issue titles are idempotency keys; do not rename them casually.

## Testing Guidelines

For workflow edits, prefer manual dispatch with a narrow dry run when available. `bump-adl.yml` supports `dry_run=true`; sync workflows audit and file issues only. For `repos.yaml`, verify every target has `name`, `language`, and `kind`, and that the `kind` routes to the intended workflow family.

## Commit & Pull Request Guidelines

Recent history uses conventional-style subjects such as `chore: Add CLAUDE.md file`, `ci: Migrate ADL schema fields to v0.8.0`, and `chore(deps): Bump ...`. Follow `type(scope): summary` when useful, with common types `chore`, `ci`, `docs`, and `chore(deps)`.

PRs should explain the operational effect, list any workflow permissions or secrets touched, and link related issues. Include screenshots only for `profile/README.md` or asset changes. For workflow changes, mention the validation command or dispatch run used.

## Security & Configuration Tips

Do not introduce personal access tokens. Workflows use the `inference-gateway-maintainer-bot` GitHub App plus documented secrets. Keep sync orchestrators issue-only: they must not open PRs, modify target code, or mention `@claude`.
