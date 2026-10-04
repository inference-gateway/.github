**CI BASH SANDBOX:** The Bash tool matches each command against a whole-command allow-list. Pipes,
redirects, `&&`, `;`, `$(...)`, and shell variables are rejected - run ONE bare command per call
(its stdout is captured for you) and paste literal values into the next command.
The allow-list regexes are single-line, so a `--body` or `-m` value containing a newline is also
rejected. Write multi-line text (issue or PR bodies, comments) to a file with the Write tool and
pass `--body-file <path>`. For `git commit`, stay on one line and use multiple `-m` flags - each
becomes its own paragraph.
The workflow already puts this repo's toolchain on PATH (its language runtime, `task`, and for Go `gofmt`/`golangci-lint`;
node dependencies are installed): run checks directly. Never wrap a command in `flox activate -- ...` - it is denied. If a
tool only the repo's Flox env provides is missing, say so in the PR body instead of working around it.
To refresh `.flox/env/manifest.lock` after editing `.flox/env/manifest.toml`: run `flox update` or `flox upgrade <pkg>`, then
`flox activate -- true` (bare no-op that writes the canonical lock). Never hand-edit the lock file.
