When you are working on a GitHub issue (not a PR comment or a manual/direct task), track it on the
org Roadmap 2026 board (GitHub org project #7) as you go - best-effort; if a `gh project` call fails
for lack of a GitHub permission, follow the "IF A GITHUB PERMISSION IS MISSING" rule below, then
carry on - never abort the task over a board update. Use these org-stable ids verbatim - do not
improvise:
  - project id `PVT_kwDOC6ve6c4BNnSt`, Status field id `PVTSSF_lADOC6ve6c4BNnStzg8jqxU`
  - Status option ids: In progress `47fc9ee4`, QA `0a18abd7` (Todo `f75ad846`, Done `98236657`)
This CI shell disables command substitution `$(...)` and shell variables: run each command, read the
id it prints, and paste that literal id into the next command - never write `ID=$(...)`.

- BEFORE you change any code: add the issue to project #7 and set Status = "In progress".
  - Resolve the issue URL from your task context (e.g. `gh issue view <number> --json url -q .url`).
  - `gh project item-add 7 --owner inference-gateway --url <issue-url>` (idempotent - if it is already
    on the board this just reports it exists, carry on).
  - Read its project-item id:
    `gh project item-list 7 --owner inference-gateway --format json -L 1000 -q '.items[] | select(.content.url=="<issue-url>") | .id'`
  - Set Status to In progress (paste the item id):
    `gh project item-edit --id <item-id> --project-id PVT_kwDOC6ve6c4BNnSt --field-id PVTSSF_lADOC6ve6c4BNnStzg8jqxU --single-select-option-id 47fc9ee4`
    (if it is already on the board at another Status, just move it to In progress.)
- RIGHT AFTER you open the pull request: set Status to "QA" with the same item id:
  `gh project item-edit --id <item-id> --project-id PVT_kwDOC6ve6c4BNnSt --field-id PVTSSF_lADOC6ve6c4BNnStzg8jqxU --single-select-option-id 0a18abd7`
  Do NOT set Done - that happens when the PR merges, which is outside this run.
