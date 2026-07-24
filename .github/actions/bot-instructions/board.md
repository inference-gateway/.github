When working on a GitHub issue (not a PR comment or a manual/direct task), track it on the org
Roadmap 2026 board (project #7) as you go. Best-effort: on a permission failure follow the
"IF A GITHUB PERMISSION IS MISSING" rule below and carry on - never abort the task over a board
update. Use these org-stable ids verbatim:
  - project id `PVT_kwDOC6ve6c4BNnSt`, Status field id `PVTSSF_lADOC6ve6c4BNnStzg8jqxU`
  - Status option ids: In progress `47fc9ee4`, QA `0a18abd7` (Todo `f75ad846`, Done `98236657`)
This CI shell disables command substitution `$(...)` and shell variables: run each command, read
the id it prints, and paste that literal id into the next command - never write `ID=$(...)`.

- BEFORE you change any code: add the issue to project #7 and set Status = "In progress".
  - Resolve the issue URL: `gh issue view <number> --json url -q .url`
  - Add it and read the printed project-item id (idempotent - returns the id whether it creates
    the item or it was already on the board):
    `gh project item-add 7 --owner inference-gateway --url <issue-url> --format json -q .id`
  - Set Status to In progress (paste the item id):
    `gh project item-edit --id <item-id> --project-id PVT_kwDOC6ve6c4BNnSt --field-id PVTSSF_lADOC6ve6c4BNnStzg8jqxU --single-select-option-id 47fc9ee4`
- RIGHT AFTER you open the pull request: set Status to "QA" with the same item-edit command and
  option id `0a18abd7`. Do NOT set Done - that happens when the PR merges, outside this run.
