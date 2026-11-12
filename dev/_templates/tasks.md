---
topic: <topic>
status: draft
branch: <type>/<topic>
prs: []
updated: YYYY-MM-DD
---

# <Title> — Tasks

Small, independently checkable steps. Tick each one when it is committed;
note the PR number for steps that land in different PRs.

## Build

- [ ] …

## Verify

- [ ] `cargo fmt --all -- --check`
- [ ] `cargo clippy --workspace --all-targets -- -D warnings`
- [ ] `cargo test --workspace`
- [ ] `bash scripts/test-couchdb.sh` (if CouchDB-facing)

## Ship

- [ ] `CHANGELOG.md` `[Unreleased]` updated
- [ ] Docs / README / rustdoc updated
- [ ] PR opened, CI green, squash-merged
- [ ] `status: done` set in this topic's documents
