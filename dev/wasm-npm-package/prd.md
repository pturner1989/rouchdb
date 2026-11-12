---
topic: wasm-npm-package
status: draft
branch: feat/wasm-npm-package
prs: []
updated: 2026-11-12
---

# RouchDB for JavaScript (WebAssembly + npm) — PRD

## Problem

RouchDB only exists as a Rust library. JavaScript applications that want an
offline-first store which syncs with CouchDB have one mature choice, PouchDB,
which is pure JavaScript: slow bulk writes and replication on large
databases, base64 attachments end to end, and loose TypeScript types
(`@types/pouchdb` is community-maintained and lags). RouchDB already has a
CouchDB-faithful core (revision trees, merge, Mango, map/reduce, replication
checked against CouchDB 3.5.1) that JavaScript cannot use.

## Goal

Ship RouchDB to JavaScript as an **npm package built from the Rust code
compiled to WebAssembly**, so applications can store JSON documents and
attachments offline and synchronise them with CouchDB or any server speaking
the CouchDB replication protocol.

### Goals

1. **Full functionality** from JavaScript: CRUD and bulk writes, `allDocs`,
   Mango `find` and indexes, map/reduce queries and design documents,
   changes feed (one-shot and live), attachments, local documents,
   conflicts, compaction, destroy, purge, and one-shot / live / two-way
   replication with checkpoints, filters and events.
2. **First-class TypeScript**: complete, hand-reviewed `.d.ts` shipped in the
   package, generic over the application's document type, no `any` in the
   public surface, type tests in CI.
3. **Efficient documents and attachments**: attachments cross the JS/wasm
   boundary and sit in storage as binary (`Blob` / `Uint8Array`), never
   base64; unchanged attachments are not re-transferred during replication;
   bulk operations cross the boundary once per batch, not once per document.
4. **Offline storage** that survives reloads and restarts in every supported
   runtime.
5. **Interoperability**: replicates with CouchDB 3.x, `rouchdb-server`,
   PouchDB (through CouchDB) and other servers implementing the protocol.

### Non-goals (first release)

- Drop-in replacement for PouchDB (same names and semantics where sensible,
  but not passing PouchDB's own test suite; a compatibility shim may follow).
- React Native (Hermes has no WebAssembly; would need a native binding).
- Running CouchDB design-document JavaScript (views, `validate_doc_update`,
  filters stored as source strings) by default; see the spec for the opt-in.
- Redb on OPFS in the browser (phase 2, after the first release).
- Server-side features: `rouchdb-server` and `rouchdb-cli` stay native-only.
- Multi-threaded wasm (`SharedArrayBuffer`/atomics); the build is
  single-threaded and needs no cross-origin isolation headers.

## Users and use cases

| User | Runtime | Use case |
|---|---|---|
| Web app developer | Browser main thread or Web Worker | Offline-first app: local database in IndexedDB, live two-way sync with a CouchDB per user; photos/PDFs as attachments. |
| PWA developer | Service Worker | Background sync of queued writes when the network returns. |
| Desktop app developer | Electron, Tauri (webview) | Local database in the renderer/webview, sync to a company CouchDB. Tauri apps may alternatively use the native Rust crate in the backend. |
| Hybrid mobile developer | Capacitor (WKWebView / Android WebView) | Field app working offline for days, syncing when connected. |
| Node developer | Node 20+ (Bun/Deno best effort) | Scripts, edge services, test fixtures, CLIs that replicate and store to disk. |

## Supported runtimes (first release)

- Browsers: current and previous major of Chrome, Edge and Firefox, and
  **Safari 16.4+ / iOS 16.4+** (`BroadcastChannel`, reliable `Blob`s in
  IndexedDB, and OPFS sync access handles for phase 2); main thread,
  dedicated workers, service workers.
- Node.js 20+ (active LTS lines at release time).
- Electron, Tauri (WebView2, WKWebView, WebKitGTK), Capacitor, as browser
  environments; each gets a smoke test.

## Storage

- Browser: **IndexedDB** adapter (new), working in every browser context and
  in several tabs at once.
- Node: **redb on the filesystem** (the existing adapter over a Node `fs`
  backend), same file format as native RouchDB.
- Everywhere: in-memory adapter.
- Later (phase 2): redb on OPFS in a dedicated worker as an opt-in fast path.

## API

PouchDB-style and typed: familiar names (`put`, `get`, `bulkDocs`,
`allDocs`, `find`, `createIndex`, `query`, `changes`, `replicate`, `sync`,
`putAttachment`, `getAttachment`...), promises everywhere, generics for
document types, async iterators and event emitters for live feeds. Not
guaranteed to be drop-in compatible with PouchDB.

## Success metrics

- Every goal above covered by tests running in Chromium, Firefox, WebKit and
  Node in CI, including replication against CouchDB 3.5.1.
- Bulk insert of 10 000 small documents and a 10 000-document pull
  replication faster than PouchDB (IndexedDB adapter) on the same machine
  (benchmarks published in the repo).
- Attachment replication of a 50 MB attachment completes without base64 in
  storage and with peak memory under ~3x the attachment size.
- Package size: wasm under 1.5 MB gzipped (target; measured in CI).
- Zero `any` in the published typings; type tests pass.

## Compatibility

- Native Rust API and behaviour unchanged; wasm support is additive (cfg'd
  code, new crates). Any change to shared crates passes the existing native
  suites.
- The MSRV (1.88) still holds for the native crates; wasm-only dependencies
  must build on it too, or the wasm crates get their own documented MSRV.
- Node redb files are the same format as native 0.5 files.

## Distribution

- **Registry**: private **GitHub Packages** (`npm.pkg.github.com`) only,
  initially. No public npmjs.com release.
- **Name**: `rouchdb`, published as **`@pturner1989/rouchdb`** because
  GitHub Packages only accepts names scoped to the owning account. If it is
  later published publicly, it can become the unscoped `rouchdb`.
- **Consumers** need a GitHub token with `read:packages` and an `.npmrc`:
  `@pturner1989:registry=https://npm.pkg.github.com`. Documented in the
  package README.
- **Repository**: to be made private. A fork of a public repository cannot
  be made private on GitHub, so this means either asking GitHub Support to
  detach the fork and then changing its visibility, or moving the code to a
  new private repository. Private repositories have limited free Actions
  minutes (macOS runners count 10x), which affects the CI plan (see plan,
  Risks).
- **Version**: the npm package follows the workspace version (0.5.x,
  same semver rules: pre-1.0 minor bumps may break), and is released
  together with the crates.
- **License**: MIT, as the crates; the package keeps the `LICENSE` notice
  (copyright rubylab-app) as MIT requires.

## Open questions

None at present.
