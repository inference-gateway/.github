When the change you implement has ripple effects, file best-effort follow-up tracking issues as you
open the PR. This is best-effort: on any `gh` permission failure follow the "IF A GITHUB PERMISSION
IS MISSING" rule below and carry on; never write `@claude`/`@infer` in anything you file.

- Docs: if the change adds or alters public-facing behavior and you are NOT already working in
  `inference-gateway/docs`, open a `[DOCS]` issue on `inference-gateway/docs` describing what needs
  documenting (user-facing change, affected pages, link to your PR), with `--label documentation`:
  `gh issue create --repo inference-gateway/docs --title "[DOCS] <summary>" --label documentation --body "<...>"`
  Keep that issue ASCII-only (the docs repo forbids en/em dashes - plain hyphen only). Skip it only if
  your PR body states `no docs ticket: internal-only refactor`.
- Downstream impact: if the change ripples to other org repos (e.g. a `schemas` change reaching the
  SDKs / gateway / docs, a gateway release needing `operator` + Helm image bumps, or a new config env
  var needing regenerated docs / Helm values / example `.env`s), call out the affected repos in your
  PR body AND open a follow-up tracking issue describing the impact - on the affected repo when you
  have write access there, otherwise on the current repo (or `inference-gateway/.github`) listing the
  affected repos for a maintainer to route. Do not fan out edits silently.
