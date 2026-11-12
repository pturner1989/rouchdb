---
topic: wasm-npm-package
status: draft
branch: feat/wasm-npm-package
prs: []
updated: 2026-11-12
---

# RouchDB for JavaScript (WebAssembly + npm) — Plan

## Approach

Make the existing crates portable to `wasm32-unknown-unknown` without
changing native behaviour, add an IndexedDB adapter, then put a
wasm-bindgen layer and a hand-written TypeScript API on top. Work in phases
that each land as one or more PRs on `main` and keep every native suite
green; nothing user-visible ships to npm until phase 5.

### Key decisions

**D1. `Send` bounds: keep them, wrap JS values in `SendWrapper`.**
`Adapter`, `Plugin`, `MapFn` and `Database` require `Send + Sync`;
browser futures and JS values are `!Send`. Options:
- (a) `#[cfg_attr(target_arch = "wasm32", async_trait(?Send))]` plus
  `MaybeSend`/`MaybeSync` marker traits across every trait, impl and
  `Arc<dyn ...>`: touches the whole public API.
- (b) **Chosen**: keep the bounds; wasm-only code wraps `!Send` futures and
  values in `send_wrapper::SendWrapper`. The wasm build is single-threaded
  (no atomics), so a `SendWrapper` is never touched from another thread and
  never panics. Native API untouched.
Revisit if multi-threaded wasm becomes a goal.

**D2. Runtime shim instead of Tokio time/spawn.** A `rouchdb_core::rt`
module: `spawn`, `sleep`, `sleep_until`, `timeout`, `Instant`. Native maps to
Tokio (no behaviour change), wasm to `wasm_bindgen_futures::spawn_local`,
`setTimeout` (via `gloo-timers` or a hand-rolled `js_sys::Promise`) and
`web-time::Instant`. `tokio::sync` and `tokio::select!` stay (both work on
wasm without a runtime). The test-only uses of `tokio::time` stay native.

**D3. Keep reqwest for HTTP on wasm** (fetch backend: per-request timeout,
`credentials: include`, `bytes_stream` all exist in 0.12.28). Cfg out
`cookie_store`, `connect_timeout`, `read_timeout`, `http2`, `system-proxy`
on wasm; replace with per-request timeouts and our own `AuthSession`
handling in Node. A custom `fetch` from JS is supported by an alternative
transport behind the same internal trait if reqwest cannot take one (spike
S2 decides).

**D4. IndexedDB through `web-sys` directly or a thin crate.** Candidates:
`indexed_db_futures`, `rexie`, raw `web-sys` + a small promise helper.
Spike S3 picks on: transaction lifetime control (no auto-commit while we
await non-IDB futures), Blob support, size, maintenance.

**D5. Node storage = redb over a JS `fs` backend**, not a separate Node
adapter: reuses the most-tested adapter and keeps one file format.
Requires `RedbAdapter::from_backend(Box<dyn redb::StorageBackend>, ...)`
and replacing `spawn_blocking` with inline calls on wasm.

**D6. Hand-written TS API over generated bindings.** wasm-bindgen output is
an internal layer; `npm/src` is the public, documented API (PouchDB naming,
generics, events, async iterators). Serde types shared with Rust come from
`tsify` so they cannot drift. This keeps the public surface stable when the
bindings change.

**D7. One package, conditional exports.** One wasm binary, glue per runtime
(`web` for browsers/workers, `nodejs` for Node), ESM-first with a CJS entry.
wasm-bindgen CLI directly (wasm-pack's maintenance is uncertain since the rustwasm organisation was wound down; verify in S5) + `wasm-opt
-Oz`, with `lto = true`, `opt-level = "z"` / `"s"` (whichever benchmarks
better), `codegen-units = 1`, `panic = "abort"` in a dedicated
`[profile.wasm-release]`.

**D8. Attachment efficiency is in scope for the first release**, not a
follow-up: binary at the boundary, content-addressed Blobs in IndexedDB,
`atts_since` and binary/multipart transfer in replication (spec, Replication).
Native users benefit from the replication changes too.

### Alternatives rejected

- **Pure TypeScript port**: loses the CouchDB-verified Rust core; two
  codebases to keep in sync.
- **napi-rs native addon for Node**: faster on Node but a second binding
  and per-platform binaries; wasm covers Node adequately. Revisit if Node
  performance becomes a goal.
- **Redb on OPFS only**: worker-only, no Service Worker, one tab at a time;
  kept as phase 2 opt-in.
- **wasm-pack**: uncertain maintenance; wasm-bindgen CLI + a small build script
  gives the same output with pinned versions.

## Phases and PR split

Each phase is one or more PRs into `main` (on the fork), squash-merged.

### Phase 0 — Spikes (throwaway code, findings recorded in `notes/`)

- S1 redb with a custom `StorageBackend` over Node `fs` from wasm: works,
  file readable by native CLI, rough throughput.
- S2 reqwest on wasm: timeouts, CORS + credentials against CouchDB,
  streaming, injecting a JS `fetch`; Node cookie handling.
- S3 IndexedDB crate choice (D4) and transaction semantics with async Rust.
- S4 JSON across the boundary: `JSON.stringify` + `serde_json` vs
  `serde-wasm-bindgen`, for 1 KB / 10 KB / 1 MB documents and 10k-doc batches.
- S5 Size budget: current wasm size with all crates, what `fancy-regex`,
  `serde_json` `unbounded_depth` and reqwest cost.
- S6 wasm stack size needed for `MAX_NESTING_DEPTH` documents and selectors.
- S7 MSRV: do `wasm-bindgen`, `web-sys`, `web-time`, `send_wrapper` build on
  1.88?

### Phase 1 — Portable crates (PRs: `build(wasm): ...`, `refactor(core): rt ...`, `fix(http): ...`)

1. `uuid` `js` feature on wasm (target-specific dependency).
2. `rouchdb_core::rt` (D2); move `changes`, `replication`, `http`, `redb` and
   `rouchdb` production code onto it.
3. HTTP adapter on wasm (D3): cfg the client builder, `SendWrapper` futures
   (D1), per-request timeouts, retry sleep via `rt`.
4. Redb: `from_backend` constructor; inline execution on wasm instead of
   `spawn_blocking`; `std::fs`-dependent code (upgrade with backups, path
   open) native-only.
5. CI job: `cargo check --target wasm32-unknown-unknown` for every library
   crate; `cargo clippy` on the target.
Native behaviour unchanged; all native suites green.

### Phase 2 — Shared conformance suite + IndexedDB adapter (PRs: `test: ...`, `feat(indexeddb): ...`)

1. Extract `crates/rouchdb/tests/adapter_conformance.rs` scenarios into a
   `rouchdb-conformance` crate (publish = false): scenarios generic over a
   fixture trait, runnable with `#[tokio::test]` natively and
   `wasm_bindgen_test` in browsers.
2. `rouchdb-adapter-indexeddb`: schema (spec), `bulk_docs` via
   `rouchdb_core::write`, `all_docs`, `changes`, `revs_diff`, `bulk_get`,
   local docs, attachments (content-addressed Blobs), compaction, destroy,
   purge, security, `subscribe` + `BroadcastChannel`.
3. Conformance suite green in headless Chromium, Firefox and WebKit
   (`wasm-bindgen-test-runner` + Playwright/WebDriver).

### Phase 3 — Attachment-efficient replication (PRs: `feat(replication): ...`, `feat(http): ...`)

1. `atts_since` from `revs_diff` possible ancestors in `_bulk_get` / `open_revs`.
2. Binary fetch for attachments above the threshold.
3. `multipart/related` push for large new attachments.
4. `feed=longpoll` for live replication from HTTP sources.
5. Tests asserting the HTTP requests made (no re-download of unchanged
   attachments), against CouchDB 3.5.1; CHANGELOG entries (native users get
   this too).

### Phase 4 — Bindings and TypeScript API (PRs: `feat(wasm): ...`, `feat(npm): ...`)

1. `crates/rouchdb-wasm`: wasm-bindgen exports for `Database`, replication
   handles, changes feeds, plugins, map/reduce callbacks; Node fs backend
   (S1); `RouchError` → JS error mapping.
2. `npm/`: TS public API (spec), live handles (async iterator + events +
   `AbortSignal`), runtime detection for `Database.open`, `init()`.
3. `tsify` types; type tests; TSDoc on every export.
4. Tests: vitest (Node), Playwright (Chromium, Firefox, WebKit), CouchDB
   replication from both, PouchDB interop test, two-tab test.

### Phase 5 — Packaging, docs, CI, first release (PRs: `build(npm): ...`, `docs: ...`, `ci: ...`)

1. Build script: cargo → wasm-bindgen (web + nodejs) → wasm-opt → tsc →
   `dist/`; `package.json` exports; size check.
2. Example apps as smoke tests: Vite, webpack, Node ESM/CJS, Electron,
   Tauri, Capacitor (CI where practical, manual checklist otherwise).
3. Benchmarks vs PouchDB (spec targets) in `crates/rouchdb-bench` or `npm/bench`.
4. Docs: book chapter "JavaScript and TypeScript" (+ `es/`), npm README,
   API reference generated from TSDoc (TypeDoc).
5. CI: wasm build + tests + size + type tests; a check that
   `npm/package.json` `version` equals the workspace version; a publish workflow that runs
   `npm publish` to GitHub Packages (`npm.pkg.github.com`, `GITHUB_TOKEN`
   with `packages: write`) on a version tag, behind a manual-approval
   environment. (npm provenance is not available for GitHub Packages.)
6. Version, CHANGELOG, publish `@pturner1989/rouchdb` to GitHub Packages;
   README section on installing from it (`.npmrc`, `read:packages` token).

### Phase 6 — After the first release

- Redb on OPFS in a dedicated worker with a main-thread proxy and Web Locks
  leader election (fast path, opt-in).
- PouchDB compatibility shim.
- Multi-tab replication leader election (only one tab runs live sync).

## Crates and files touched

| Crate | Change |
|---|---|
| `rouchdb-core` | `rt` module; wasm target deps |
| `rouchdb-adapter-memory` | `uuid` js feature |
| `rouchdb-adapter-redb` | `from_backend`; no `spawn_blocking` on wasm; fs-only code cfg'd |
| `rouchdb-adapter-http` | wasm client builder, `SendWrapper`, timeouts, cookies, `atts_since`, binary/multipart attachments, longpoll |
| `rouchdb-changes`, `rouchdb-replication`, `rouchdb` | `rt` instead of `tokio::spawn` / `tokio::time` |
| `rouchdb-adapter-indexeddb` | new |
| `rouchdb-conformance` | new (publish = false), scenarios moved from `rouchdb/tests` |
| `rouchdb-wasm` | new (publish = false) |
| `npm/` | new: TS API, build, tests, examples |
| `.github/workflows/` | wasm job(s), npm publish workflow |
| `docs/book/src/` | JavaScript chapter (+ `es/`) |
| `docker-compose.yml` / CI CouchDB | CORS with credentials for browser tests |

## Testing

- Native: all existing suites unchanged; conformance suite now from
  `rouchdb-conformance`.
- wasm: `wasm-bindgen-test` for Rust-level tests in Node and headless
  Chromium/Firefox/WebKit.
- JS: vitest (Node) and Playwright (three engines) on the public TS API;
  type tests; CouchDB replication tests (CORS enabled on the CI CouchDB).
- Interop: PouchDB ↔ CouchDB ↔ RouchDB-JS round trip.
- Benchmarks and size checks in CI (non-blocking first, blocking once stable).

## Docs and changelog

- CHANGELOG `[Unreleased]`: phase 1 (wasm builds), phase 3 (replication
  efficiency, longpoll), new crates.
- Book: new "JavaScript and TypeScript" chapter, Spanish translation, crate
  guide and CLAUDE.md updated for the new crates and `npm/`.

## Risks

| Risk | Mitigation |
|---|---|
| IndexedDB transactions auto-commit when we await a non-IDB future | S3; keep each adapter operation inside one sync-scheduled chain of IDB requests; plan writes before opening the transaction |
| wasm size too large for web apps | S5; size budget in CI; feature flags (e.g. no HTTP for local-only use, no regex) |
| Safari IndexedDB quirks (Blob storage, eviction, private mode) | WebKit in CI; documented limits; persist() |
| CORS/cookie auth friction in browsers | Docs + `fetch`/bearer options; CouchDB CORS config example |
| Behaviour drift between adapters | Shared conformance crate on every adapter |
| `SendWrapper` misuse if threads are added later | D1 documented; a compile error guard (`#[cfg(target_feature = "atomics")] compile_error!`) |
| Native regressions from the `rt` refactor | Native suites + mutants; refactor in a PR of its own |
| Private repo: limited free Actions minutes, macOS at 10x | Run the browser matrix and CouchDB suites on PRs touching wasm/npm paths only; WebKit on Linux (Playwright) instead of macOS runners; nightly/bench workflows on demand; review `nightly.yml`'s macOS job once private |
| Repo cannot be made private while it is a fork | Detach via GitHub Support or move to a new private repo before the first publish; update CLAUDE.md's fork rule and the memory if the repo name or remote changes |

## Decision log

- 2026-11-12 — Scope set with the user: runtimes Browser, Node, Electron,
  Capacitor, Tauri; storage IndexedDB first, OPFS later, redb files on Node;
  API PouchDB-style and typed, not drop-in.
- 2026-11-12 — Distribution: private GitHub Packages only, as
  `@pturner1989/rouchdb` (GitHub Packages requires a scope); repo to go
  private.
- 2026-11-12 — npm version follows the workspace version; release PRs
  (`chore(release): prepare X.Y.Z`) bump it with the crates.
- 2026-11-12 — Minimum Safari 16.4 / iOS 16.4.
- 2026-11-12 — D1–D8 proposed (this plan); to confirm at plan review.
