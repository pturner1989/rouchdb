# Working in this repository

These are the standards for changing RouchDB. They write down the conventions
the project already follows (see the git history, `CHANGELOG.md`, CI and the
[contributor guide](docs/book/src/contributing/)) plus a few new ones. When a
rule here and the code disagree, fix one of them in the same PR.

## 1. Planning documents live in `dev/`

PRDs, specs, plans and task lists for a piece of work go in **`dev/<topic>/`**,
one directory per topic. See [`dev/README.md`](dev/README.md) for the layout,
naming, lifecycle and templates.

- Not every change needs them: a one-commit fix or a doc tweak goes straight
  to a branch. Write a plan when the work spans several commits, crates or
  PRs, changes a public API or on-disk format, or someone else needs to
  review the approach before the code.
- Do not put planning documents anywhere else (not `docs/`, which is the
  published mdBook; not the repository root; not crate directories).

## 2. Branches

**Never commit features or bug fixes directly to `main`.** Every change goes
on a branch and lands through a pull request.

- Name the branch `<type>/<topic>`: `<type>` is a commit type from §3 and
  `<topic>` is short kebab-case, e.g. `feat/partitioned-views`,
  `fix/replication-checkpoint-reset`, `docs/dev-standards`.
- When the work has a `dev/<topic>/` directory, use the same `<topic>` for the
  branch so the two can be found from each other.
- Branch from an up-to-date `main`; keep one topic per branch. Rebase on
  `main` rather than merging `main` into the branch.
- Delete the branch once its PR is merged.

## 3. Commits

The history uses [Conventional Commits](https://www.conventionalcommits.org/):

```
<type>(<scope>)!: <summary in the imperative, lower case, no full stop>

<body: what changed and why, wrapped at ~72 columns>

BREAKING CHANGE: <what callers must do differently>
Co-Authored-By: ...
```

- **Types**: `feat`, `fix`, `docs`, `test`, `refactor`, `perf`, `build`, `ci`,
  `bench`, `chore` (`chore(release)` for release preparation, `chore(deps)`
  for dependency bumps).
- **Scope** (optional): the area changed, by short crate or component name:
  `core`, `memory`, `redb`, `http`, `query`, `views`, `changes`, `replication`,
  `server`, `cli`, `storage`, `adapters`, `test`. Several are comma-separated:
  `fix(server,query): ...`.
- **Breaking changes** get `!` after the type/scope **and** a
  `BREAKING CHANGE:` footer. Pre-1.0, breaking changes ship in a minor release.
- The **body** explains the problem and the reasoning, not a list of files:
  what was wrong, how it showed up, why this fix. Reference the CouchDB
  behaviour (version, endpoint, source file) when that is the justification.
- Keep each commit buildable and its tests passing.

## 4. Pull requests

- PR title = the conventional commit summary for the whole change; PRs are
  **squash-merged**, so the title (plus ` (#NN)`) becomes the commit on
  `main` and the body becomes its message.
- The description says what changed and why, lists breaking changes and
  links the `dev/<topic>/` directory when there is one.
- Before opening (or marking ready), run locally:

  ```bash
  cargo fmt --all -- --check
  cargo clippy --workspace --all-targets -- -D warnings
  cargo test --workspace
  bash scripts/test-couchdb.sh      # when the change touches anything CouchDB-facing
  ```

- CI must be green. The non-blocking jobs (xfail check, mutants, stable/beta
  clippy) are read, not ignored: a surviving mutant in changed code usually
  means a missing assertion.
- Update `CHANGELOG.md` (§7) and the docs (§8) **in the same PR** as the code.

## 5. Code

- Edition 2024, stable Rust, MSRV 1.88 (`rust-version`); the toolchain is
  pinned in `rust-toolchain.toml`. Do not use features newer than the MSRV.
- `cargo fmt` and `cargo clippy -D warnings` clean. Collapse nested `if let`
  into let-chains (edition 2024).
- All crate versions follow the workspace `version`; internal dependencies
  carry both `path` and the matching `version`.
- Async is Tokio throughout; `Adapter` implementations are `Send + Sync`.
- Errors are `RouchError` variants mapped to CouchDB's `{"error", "reason"}`;
  do not add stringly-typed errors where a variant fits.
- New public types that may grow are `#[non_exhaustive]` with constructors.
- Writes through `Database` go through `bulk_docs()` so plugin hooks fire.
- **CouchDB is the reference.** When behaviour is in question, check it
  against CouchDB 3.5.1 (the version CI runs) and match it; document any
  deliberate divergence.
- No new system-library dependencies: storage stays pure Rust (redb) and TLS
  defaults to rustls.

## 6. Tests

Full details in the [testing guide](docs/book/src/contributing/testing.md).

- Every bug fix comes with a test that fails without it; every feature with
  tests of its behaviour (not just that it runs). Assert on real values;
  no tautological asserts.
- Unit tests: `#[tokio::test]`, `Database::memory("test")` for speed,
  `tempfile::tempdir()` for redb.
- CouchDB tests: `#[ignore = "requires CouchDB"]`, create databases with
  `common::fresh_remote_db` and credentials from `common::couchdb()`; never
  log in as the admin with a wrong password.
- A test that exposes a known, unfixed bug keeps its assertions and is marked
  `#[ignore = "blocked on Fxx"]` with a `blocked_on_fxx_` name prefix; the PR
  that fixes the bug removes both. `#[ignore]` takes no other reasons.
- On-disk format changes need cross-version tests in `rouchdb-compat-tests`.

## 7. Changelog and releases

- `CHANGELOG.md` follows Semantic Versioning (pre-1.0: minor bumps may break).
  Add user-visible changes under `## [Unreleased]` in the PR that makes them,
  in the existing sections (`Security`, `Breaking changes`, `Added`,
  `Changed`, `Fixed`, `Documentation`, `CI and tooling`), with a bold lead
  sentence and the PR number, e.g. `(#33)`.
- Breaking entries say what breaks and what to do instead; large ones get a
  migration guide in `docs/book/src/upgrading/`.
- A release is its own PR, `chore(release): prepare X.Y.Z`: bump the
  workspace version and internal dependency versions, date the changelog
  section. Publish the library crates in dependency order (see `CLAUDE.md`).

## 8. Documentation

- Public API changes update rustdoc, the relevant guide in `docs/book/src/`
  and, when examples change, `README.md` (its examples are doctests).
- Guides that have a Spanish version under `docs/book/src/es/` are updated
  together with the English one.
- Keep `CLAUDE.md` and the contributor guide accurate when the architecture,
  commands or CI change.
