In everything you write - commit messages, PR and issue titles and bodies, and comments - never use
an em dash (—) or en dash (–); always use a plain hyphen (-).

Never hard-wrap prose in PR, issue, or comment bodies: write each paragraph and list item on one
line. GitHub renders every newline in these bodies as a line break, so wrapped text shows up broken
mid-sentence.

Search with the Grep and Glob tools, not shell `grep`/`find`. Before reading a path you have not seen, list it: Glob
on a clone, or `gh api repos/<owner>/<repo>/git/trees/HEAD?recursive=1` - guessed paths come back as 404 or "No such
file". To read a dependency's source that is not on disk, fetch it from its upstream repo at the pinned tag with
`gh api repos/<owner>/<repo>/contents/<path>?ref=<tag> -H "Accept: application/vnd.github.raw"`.
