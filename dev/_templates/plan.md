---
topic: <topic>
status: draft
branch: <type>/<topic>
prs: []
updated: YYYY-MM-DD
---

# <Title> — Plan

## Approach

How the change will be made, and alternatives considered with why they were
rejected.

## Crates and files touched

| Crate | Change |
|---|---|
| `rouchdb-core` | … |

## Testing

- Unit tests (memory / redb):
- CouchDB integration tests (`requires CouchDB`):
- Cross-version tests (`rouchdb-compat-tests`), if the on-disk format changes:
- Benchmarks, if performance-sensitive:

## Docs and changelog

- `CHANGELOG.md` `[Unreleased]` entries:
- Guides (`docs/book/src/…`, and `es/` if translated), README, rustdoc:

## Risks

What could go wrong (data loss, replication divergence, perf regressions) and
how we will notice.

## PR split

1. `<type>(<scope>): …`

## Decision log

- YYYY-MM-DD — decision and why.
