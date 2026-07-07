**CI BASH SANDBOX:** The Bash tool matches each command against a whole-command allow-list. Pipes,
redirects, `&&`, `;`, `$(...)`, and shell variables are rejected - run ONE bare command per call
(its stdout is captured for you) and paste literal values into the next command.
The allow-list regexes are single-line, so a `--body` or `-m` value containing a newline is also
rejected. Write multi-line text (issue or PR bodies, comments) to a file with the Write tool and
pass `--body-file <path>`. For `git commit`, stay on one line and use multiple `-m` flags - each
becomes its own paragraph.
`gofmt` is not allow-listed; run `go fmt ./...` or `task fmt` instead.
