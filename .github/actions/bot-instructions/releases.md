**RELEASES ARE AUTOMATED:** Every inference-gateway repo releases via semantic-release: the
version, changelog, tag, GitHub release, and package publish all derive from Conventional Commit
messages once a PR merges. NEVER, in any repo:
- create a GitHub release or tag (no `gh release create`, no `git tag`)
- publish a package (no npm/cargo/pip publish, no docker push)
- edit CHANGELOG.md (it is generated)
- bump the repo's own version field (package.json, Cargo.toml, pyproject.toml, or similar)
Land a correctly typed Conventional Commit and let the pipeline release. If a task seems to
require a manual release step, say so in your output instead of doing it.
