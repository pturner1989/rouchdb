# `dev/` — planning documents

Every PRD, spec, plan and task list for work on RouchDB lives here, grouped by
topic. This folder is for the people (and agents) building RouchDB; user-facing
documentation lives in `docs/book/`.

## Layout

```
dev/
├── README.md              <- this file
├── _templates/            <- copy these to start a document
│   ├── prd.md
│   ├── spec.md
│   ├── plan.md
│   └── tasks.md
└── <topic>/               <- one directory per piece of work
    ├── prd.md             <- why and what: problem, users, goals, scope
    ├── spec.md            <- how it behaves: API, formats, CouchDB parity, edge cases
    ├── plan.md            <- how we build it: approach, crates touched, risks, PR split
    ├── tasks.md           <- checklist of small steps, ticked off as they land
    └── notes/             <- optional: research, benchmarks, CouchDB traces
```

A topic needs only the documents that earn their keep: a bug fix might have
just `plan.md` and `tasks.md`; a new crate would have all four.

## Naming

- `<topic>` is short kebab-case naming the work, not a date or a person:
  `partitioned-views`, `replication-checkpoint-reset`, `sqlite-adapter`.
- Use the same `<topic>` as the branch (`feat/partitioned-views`), so the
  directory and the branch can be found from each other.
- File names are fixed (`prd.md`, `spec.md`, `plan.md`, `tasks.md`). If a topic
  needs more than one of a kind, suffix it: `spec-http.md`, `spec-redb.md`.

## Front matter

Each document starts with:

```yaml
---
topic: partitioned-views
status: draft        # draft | approved | in-progress | done | abandoned
branch: feat/partitioned-views
prs: []              # e.g. [41, 42] once opened
updated: 2026-11-12
---
```

Update `status`, `prs` and `updated` as the work moves on.

## Lifecycle

1. **Start**: create `dev/<topic>/` and the branch `<type>/<topic>` together;
   copy the templates you need from `_templates/`.
2. **Agree** the PRD/spec (`status: approved`) before writing much code when
   the change is breaking, touches on-disk formats or the replication
   protocol, or adds a crate.
3. **Build**: tick off `tasks.md` as commits land; record decisions and
   changes of direction in `plan.md` rather than rewriting history.
4. **Finish**: when the last PR merges, set `status: done`. Leave the
   directory in place as the record of why the code is the way it is.
   Abandoned work stays too, with `status: abandoned` and a line saying why.

Planning documents are committed with the work they describe, normally in the
same PR (or the first PR of a multi-PR topic).
