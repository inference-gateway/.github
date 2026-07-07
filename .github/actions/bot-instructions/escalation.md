**IF A GITHUB PERMISSION IS MISSING:**
If - and only if - a `gh`/`git` command fails because the bot LACKS a GitHub permission (HTTP 403
"Resource not accessible by integration", "does not have permission", "must have write/admin
access", or an authorization-rejected project/issue mutation), file a tracking issue in
`inference-gateway/.github` so a human can grant the scope. A 404 (wrong URL), a 422 validation
error, or a transient rate-limit 403 is NOT a permission gap - skip those. Best-effort: never
abort or block the real task; if the escalation itself fails, note it in your output and continue.

Keep ONE issue org-wide per missing capability:
- Title MUST be exactly `[bot] Missing GitHub permission: <capability>` - a short stable scope
  name, e.g. `organization projects (read & write)`.
- Search first:
  `gh issue list --repo inference-gateway/.github --state open --search 'in:title "[bot] Missing GitHub permission: <capability>"'`
  - If one exists: add a COMMENT recording this occurrence (repo, issue/PR, run URL, error) and
    STOP - do not open a duplicate.
  - If none: `gh issue create --repo inference-gateway/.github --title "[bot] Missing GitHub permission: <capability>" --label bot --body ...`
    (if `--label bot` is rejected because the label is missing, retry without --label).

The body MUST state: what you were doing, the originating repo and issue/PR, the Actions run URL
($GITHUB_SERVER_URL/$GITHUB_REPOSITORY/actions/runs/$GITHUB_RUN_ID), the exact permission needed
and where to grant it (e.g. the maintainer GitHub App installation's org or repo permissions),
and the verbatim error line.

NEVER write `@claude` or `@infer` anywhere in the title, body, or comment, and do not add the
escalation issue to the Roadmap board.
