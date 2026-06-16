**IF A GITHUB PERMISSION IS MISSING:**
If - and only if - a `gh`/`git` command fails because the bot LACKS a GitHub permission
(HTTP 403 "Resource not accessible by integration", "does not have permission", "must
have write/admin access", or a project/issue mutation rejected for authorization), file
a tracking issue in `inference-gateway/.github` so a human can grant the scope. Do NOT do
this for ordinary errors - a 404 (wrong URL), a 422 validation error, or a rate-limit 403
(transient) are not permission gaps; skip them.

This is best-effort and must NEVER abort or block the real task. File the escalation, then
continue (or, if the escalation itself fails, note it in your output and continue).

Keep it to ONE issue org-wide per missing capability, not one per repo:
- Title MUST be exactly `[bot] Missing GitHub permission: <capability>`, where <capability>
  is the short stable scope name, e.g. `organization projects (read & write)` or
  `issues: write on .github`.
- Search first:
  `gh issue list --repo inference-gateway/.github --state open --search 'in:title "[bot] Missing GitHub permission: <capability>"'`
  - If one exists: add a COMMENT recording this occurrence (repo, issue/PR, run URL, error)
    and STOP - do not open a duplicate.
  - If none: `gh issue create --repo inference-gateway/.github --title "[bot] Missing GitHub permission: <capability>" --label bot --body ...`
    (if `--label bot` is rejected because the label is missing, retry without --label).

The body MUST state: what you were doing, the originating repo and issue/PR, the Actions run
URL ($GITHUB_SERVER_URL/$GITHUB_REPOSITORY/actions/runs/$GITHUB_RUN_ID), the exact permission
needed and where to grant it (the maintainer GitHub App's installation -> Organization
permissions -> Projects: Read and write, or the specific repo permission), and the verbatim
error line.

NEVER write `@claude` or `@infer` anywhere in the title, body, or comment, and do not add the
escalation issue to the Roadmap board.
