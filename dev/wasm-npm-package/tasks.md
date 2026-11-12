---
topic: wasm-npm-package
status: draft
branch: feat/wasm-npm-package
prs: []
updated: 2026-11-12
---

# RouchDB for JavaScript (WebAssembly + npm) — Tasks

Each phase lands as its own PR(s) from a `<type>/wasm-npm-package-<phase>`
branch (or this branch for the planning docs). Tick tasks as they are
committed and note the PR.

## Planning

- [ ] PRD, spec and plan reviewed; status → approved
- [x] Package name and registry: `@pturner1989/rouchdb` on GitHub Packages
- [x] Versioning: npm follows the workspace version
- [x] Minimum Safari: 16.4 / iOS 16.4
- [ ] Repository made private (detach fork via GitHub Support, or new private repo)

## Phase 0 — Spikes (findings in `notes/`)

- [ ] S1 redb over Node `fs` from wasm
- [ ] S2 reqwest on wasm: timeouts, CORS + credentials, streaming, custom fetch, Node cookies
- [ ] S3 IndexedDB crate choice + transaction semantics
- [ ] S4 JSON boundary benchmark
- [ ] S5 wasm size baseline
- [ ] S6 wasm stack size for deep JSON
- [ ] S7 wasm deps on MSRV 1.88
- [ ] Update plan decisions D1–D8 from findings

## Phase 1 — Portable crates

- [ ] `uuid` `js` feature on wasm (memory, redb, replication, rouchdb)
- [ ] `rouchdb_core::rt` (spawn, sleep, sleep_until, timeout, Instant)
- [ ] `rouchdb-changes` on `rt`
- [ ] `rouchdb-replication` on `rt`
- [ ] `rouchdb` umbrella on `rt`
- [ ] HTTP adapter builds on wasm (client builder cfg, `SendWrapper`, per-request timeouts, retry via `rt`)
- [ ] Redb `from_backend`; inline execution on wasm; fs-only code cfg'd
- [ ] CI: `cargo check` + clippy for `wasm32-unknown-unknown`
- [ ] Native suites green; CHANGELOG

## Phase 2 — Conformance suite + IndexedDB adapter

- [ ] `rouchdb-conformance` crate; scenarios moved; native runs unchanged
- [ ] IndexedDB schema + open/upgrade/destroy
- [ ] `bulk_docs` (new_edits true/false) via `rouchdb_core::write`
- [ ] `get` (revs, open_revs, conflicts), `all_docs`, `changes`, `revs_diff`, `bulk_get`
- [ ] Local docs, security, purge
- [ ] Attachments: content-addressed Blobs, refcounts, `get/put/remove_attachment`
- [ ] Compaction
- [ ] `subscribe` + `BroadcastChannel` cross-tab notices
- [ ] Conformance green in Chromium, Firefox, WebKit
- [ ] CHANGELOG, crate README, book crate guide

## Phase 3 — Attachment-efficient replication

- [ ] `atts_since` from possible ancestors
- [ ] Binary fetch above threshold
- [ ] `multipart/related` push above threshold
- [ ] `feed=longpoll` for live HTTP sources
- [ ] Request-level tests against CouchDB 3.5.1; CHANGELOG

## Phase 4 — Bindings and TypeScript API

- [ ] `rouchdb-wasm` crate: Database, errors, JSON boundary
- [ ] Node fs `StorageBackend`
- [ ] Changes feed handles (async iterator, events, AbortSignal)
- [ ] Replication / sync handles and events
- [ ] Attachments in/out as Blob / Uint8Array
- [ ] `find`, `createIndex`, `query` with JS map/reduce, opt-in design-doc eval
- [ ] Plugins from JS
- [ ] `npm/src` TS API + TSDoc; `tsify` types; type tests
- [ ] vitest (Node) + Playwright (3 engines) API tests
- [ ] CouchDB replication tests from Node and browsers (CORS in compose/CI)
- [ ] PouchDB interop and two-tab tests

## Phase 5 — Packaging, docs, CI, release

- [ ] Build script → `dist/`; exports map; size check
- [ ] Example apps: Vite, webpack, Node ESM/CJS, Electron, Tauri, Capacitor
- [ ] Benchmarks vs PouchDB
- [ ] Book chapter (+ `es/`), npm README, TypeDoc reference
- [ ] CI jobs; GitHub Packages publish workflow with manual approval
- [ ] CLAUDE.md, crate guide, CHANGELOG updated (release checklist bumps `npm/package.json` with the workspace version)
- [ ] First release to GitHub Packages

## Phase 6 — Later

- [ ] Redb on OPFS in a worker + main-thread proxy
- [ ] Multi-tab replication leader election
- [ ] PouchDB compatibility shim
