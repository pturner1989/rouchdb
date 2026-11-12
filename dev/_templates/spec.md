---
topic: <topic>
status: draft
branch: <type>/<topic>
prs: []
updated: YYYY-MM-DD
---

# <Title> — Spec

## Summary

One paragraph: what will exist when this is done.

## Public API

New or changed types, functions, options and error variants (with signatures).
Mark new extensible types `#[non_exhaustive]`.

## Behaviour

Precise rules, including ordering, defaults and error cases. Cite the CouchDB
endpoint or source where the behaviour comes from.

## Storage and wire formats

Changes to redb tables, `_local` documents, checkpoints or HTTP payloads, and
how existing data is read or upgraded.

## Edge cases

- Deleted documents / tombstones, conflicts, attachments, `_local` docs,
  design docs, deep nesting, empty databases, concurrent writers.

## Acceptance criteria

- [ ] Testable statements; each maps to at least one test.
