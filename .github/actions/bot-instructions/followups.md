When your change has ripple effects, file best-effort follow-up tracking issues as you open the
PR (on a `gh` permission failure follow the "IF A GITHUB PERMISSION IS MISSING" rule below and
carry on). Never write `@claude`/`@infer` in anything you file.

Before creating ANY issue, fetch the matching org template from
`inference-gateway/.github` (`gh api repos/inference-gateway/.github/contents/.github/ISSUE_TEMPLATE/<file> -q .content | base64 -d`;
files: `bug_report.md`, `feature_request.md`, `documentation_request.md`, `refactor_request.md`)
and structure the issue body after it - `gh issue create` does not apply templates on its own.

- Docs: if the change adds or alters public-facing behavior and you are NOT already working in
  `inference-gateway/docs`, open a `[DOCS]` issue there (user-facing change, affected pages, link
  to your PR), ASCII-only (plain hyphen - the docs repo forbids en/em dashes):
  `gh issue create --repo inference-gateway/docs --title "[DOCS] <summary>" --label documentation --body "<...>"`
  Skip only if your PR body states `no docs ticket: internal-only refactor`.
- Downstream impact: if the change ripples to other org repos (a `schemas` change reaching the
  SDKs / gateway / docs, a gateway release needing `operator` + Helm image bumps, a new config
  env var needing regenerated docs / Helm values / example `.env`s), call out the affected repos
  in your PR body AND open a follow-up tracking issue - on the affected repo if you have write
  access there, otherwise on the current repo (or `inference-gateway/.github`) listing the
  affected repos for a maintainer to route. Do not fan out edits silently.
