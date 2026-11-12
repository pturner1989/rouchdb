---
topic: wasm-npm-package
status: draft
branch: feat/wasm-npm-package
prs: [3]
updated: 2026-11-12
---

# RouchDB for JavaScript (WebAssembly + npm) — Spec

## Summary

A single npm package that loads a WebAssembly build of RouchDB and exposes a
typed, PouchDB-style API. Browsers store data in IndexedDB, Node stores it in
redb files, and every runtime can use the memory adapter. Replication speaks
the CouchDB protocol over `fetch`.

## Current state (measured 2026-11-12)

`cargo check --target wasm32-unknown-unknown` per crate:

| Crate | Builds for wasm32 | What blocks it / breaks at runtime |
|---|---|---|
| `rouchdb-core`, `-query`, `-views` | yes | nothing known |
| `rouchdb-changes` | yes | `tokio::time` (`Instant`, `sleep_until`) and `tokio::spawn` in live changes **panic at runtime** (no clock, no Tokio runtime) |
| `rouchdb-adapter-memory`, `-replication` | no | `uuid` needs the `js` randomness feature; replication also uses `tokio::spawn`/`tokio::time` for live mode |
| `rouchdb-adapter-redb` | no | `uuid`; `spawn_blocking` (needs a Tokio runtime with threads); `std::fs` paths |
| `rouchdb-adapter-http` | no (59 errors) | reqwest's wasm futures are `!Send` but `Adapter` (async-trait) needs `Send` futures; `cookie_store` and `connect_timeout` do not exist on wasm; `tokio::time::sleep` in the retry loop |
| `rouchdb` (umbrella) | no | all of the above |

Other relevant facts:
- `AttachmentMeta.data` is `Option<Vec<u8>>`: whole attachment bodies in memory.
- The HTTP adapter pulls with `_bulk_get?attachments=true` (inline base64 in
  JSON) and never sends `atts_since`, so every replicated revision re-downloads
  every attachment as base64.
- Map functions (`rouchdb_views::MapFn`) are `Arc<dyn Fn + Send + Sync>`.
- Write planning (rev generation, conflict checks, local docs) is shared in
  `rouchdb_core::write`, so a new adapter reuses it.

## Package layout

```
crates/rouchdb-adapter-indexeddb/   new, wasm32 only, publishable on crates.io
crates/rouchdb-wasm/                new, wasm-bindgen bindings, publish = false
npm/                                the npm package: TS wrapper, build, tests
  package.json                      "@pturner1989/rouchdb", "type": "module"
  src/                              TypeScript public API over the bindings
  test/                             vitest (Node) + Playwright (browsers)
```

The published package contains:

```
dist/
  index.js / index.d.ts            ESM entry + typings
  index.cjs                        CommonJS entry for Node require()
  rouchdb_bg.wasm                  one wasm binary
  web/ node/                       runtime-specific glue (wasm-bindgen output)
```

`package.json` `exports` uses conditions (`browser`, `node`, `worker`,
`import`, `require`, `types`). Browser builds load the wasm with
`new URL('./rouchdb_bg.wasm', import.meta.url)` so Vite, webpack 5, Rollup,
esbuild and Parcel bundle it without plugins; an `init(wasmUrlOrBytes)`
escape hatch covers unusual setups (CSP, CDNs, Electron `file://`).

## TypeScript API

Names follow PouchDB; behaviour follows CouchDB 3.5.1 (as the Rust crate
does) where the two differ, and the docs list the differences.

```ts
import { Database, replicate, type Doc, type ExistingDoc } from "@pturner1989/rouchdb";

interface Todo { title: string; done: boolean }

const db = await Database.open<Todo>("todos");            // IndexedDB / redb file by runtime
const mem = await Database.memory<Todo>("scratch");
const remote = Database.http<Todo>("https://couch.example.com/todos", {
  auth: { username, password } | { cookie: true } | { bearer: token },
  fetch,                                                   // optional custom fetch
  headers: { "X-Tenant": "a" },
  timeoutMs: 30_000,
});

const { id, rev } = await db.put({ _id: "t1", title: "Buy milk", done: false });
const doc: ExistingDoc<Todo> = await db.get("t1", { conflicts: true });
await db.remove(doc);
const results = await db.bulkDocs([...]);                  // (DocResult | DocError)[]
const page = await db.allDocs({ includeDocs: true, startkey: "t", limit: 50 });

await db.createIndex({ index: { fields: ["done", "title"] } });
const { docs } = await db.find({ selector: { done: false }, sort: ["title"] });

const view = await db.query((doc, emit) => emit(doc.done, 1), { reduce: "_count", group: true });
const view2 = await db.query("app/by_done", { key: false });   // design doc view

for await (const change of db.changes({ live: true, since: "now", includeDocs: true })) { ... }
const feed = db.changes({ live: true });                   // also an EventEmitter-style handle
feed.on("change", c => ...); feed.cancel();

await db.putAttachment("t1", "photo.jpg", rev, blobOrBytes, "image/jpeg");
const blob = await db.getAttachment("t1", "photo.jpg");    // Blob (browser) / Uint8Array (Node)

const rep = db.replicate.to(remote, { live: true, retry: true, filter: { selector: {...} } });
const sync = db.sync(remote, { live: true, retry: true });
sync.on("change" | "paused" | "active" | "denied" | "error" | "complete", ...);
await sync.cancel();

await db.compact(); await db.destroy(); await db.close();
```

Rules:
- Every async method returns a `Promise`; errors reject with `RouchError`
  (a JS `Error` subclass with `status`, `name` = CouchDB `error`, `reason`),
  so `err.status === 409` works as in PouchDB.
- Options are camelCase objects; unknown keys are type errors.
- `Database<T>` documents are `T & { _id: string; _rev?: string; _deleted?;
  _attachments?; _conflicts? }`; read results are `ExistingDoc<T>`
  (`_rev` required).
- Live handles (`changes`, `replicate`, `sync`) are both `AsyncIterable` and
  event emitters, and support `cancel()` and `AbortSignal`.
- `Database.open` picks the storage by runtime; `Database.open(name,
  { adapter: "indexeddb" | "redb" | "memory" })` forces one.
- Plugins: a JS `Plugin` with `beforeWrite` / `afterWrite` / `onDestroy` hooks,
  mapped onto the Rust `Plugin` trait.

Typings: the public `.d.ts` is hand-written in `npm/src` (stable, documented
with TSDoc) and wraps the wasm-bindgen output; serde types shared with Rust are
generated with `tsify` (or checked against generated ones) so they cannot
drift. Type tests (`expect-type` / `tsd`) run in CI. Strict mode, no `any`.

## Crossing the JS/wasm boundary

- **Documents**: passed as JSON strings (`JSON.stringify` in JS, `serde_json`
  in Rust) by default. A benchmark spike compares this with
  `serde-wasm-bindgen`; the faster one for typical documents wins, per call
  site. Bulk calls (`bulkDocs`, `allDocs`, `find`, `changes` batches) cross
  once per call.
- **Attachments**: `Uint8Array` / `ArrayBuffer` / `Blob` in; `Blob` (browser)
  or `Uint8Array` (Node) out. Never base64 at the boundary.
- **Callbacks** (map/reduce functions, plugins, JS filters) are `js_sys::Function`
  values called from Rust with the document as a JS object.
- `!Send` JS values inside `Send`-bound Rust traits are wrapped in
  `send_wrapper::SendWrapper`; the build is single-threaded, so this never
  panics (see plan, decision D1).

## Storage

### IndexedDB adapter (`rouchdb-adapter-indexeddb`)

Implements `Adapter` fully (the same conformance suite as memory and redb).

Object stores (one IndexedDB database per RouchDB database, versioned
schema):

| Store | Key | Value |
|---|---|---|
| `docs` | doc id | rev tree, winning rev, deleted flag, seq of last change |
| `revs` | `[doc id, rev]` | document body JSON for each stored leaf/revision |
| `bySeq` | seq (auto-increment) | doc id (index for the changes feed) |
| `attachments` | digest (`md5-...`) | `Blob`, length, refcount |
| `local` | `_local/` id | local document |
| `meta` | key | db uuid, update seq, purge seq, security, schema version |

- Every `bulk_docs` call is one `readwrite` transaction across the stores it
  touches; planning reuses `rouchdb_core::write`.
- Attachments are **content-addressed**: one `Blob` per digest, shared
  between revisions and documents, removed by compaction when unreferenced.
- **Multi-tab**: IndexedDB handles concurrent access; committed changes are
  announced to other tabs with a `BroadcastChannel` per database so their live
  changes feeds and `Adapter::subscribe` wake up immediately.
- `versionchange` from another tab (schema upgrade, destroy) closes this
  connection cleanly and surfaces an error to open handles.
- Storage persistence: `Database.open` offers `persist: true` to call
  `navigator.storage.persist()`; quota errors map to a typed `RouchError`.

### Node storage

`RedbAdapter` gains a constructor taking a `redb::StorageBackend`; the wasm
crate implements one over Node's synchronous `fs` API (`openSync`,
`readSync`, `writeSync`, `ftruncateSync`, `fsyncSync`). The file format is the
native 0.5 format, so the CLI can inspect files written from Node.
`spawn_blocking` is replaced on wasm by running redb calls inline (they are
synchronous; the caller is already async).

### Memory adapter

Unchanged apart from building on wasm.

## Replication and HTTP

- `rouchdb-adapter-http` builds on wasm on top of reqwest's fetch backend.
  - Auth: browsers rely on the browser's cookie jar (`credentials: "include"`,
    needs CouchDB CORS with credentials) after `POST /_session`; Node keeps the
    `AuthSession` cookie itself (Node's fetch exposes `set-cookie`). Basic and
    bearer headers work in both. A user-supplied `fetch` replaces the default.
  - Timeouts per request (reqwest wasm `timeout`), retries with a wasm timer.
  - Streams response bodies (`bytes_stream`) for large attachments.
- **Attachment-efficient pull**: `_bulk_get` and `GET ?open_revs` send
  `atts_since=[the target's known ancestors]` (from `revs_diff`'s
  `possible_ancestors`), so unchanged attachments come back as stubs and are
  not re-downloaded. Attachments above a size threshold (default 64 KiB) are
  fetched individually as binary (`GET /db/doc/att?rev=`) instead of inline
  base64.
- **Attachment-efficient push**: documents with new attachments above the
  threshold are sent as `multipart/related` (`PUT /db/doc?new_edits=false`),
  like the CouchDB replicator; others stay in `_bulk_docs`.
- Live replication on an HTTP source uses `feed=longpoll` (with `heartbeat`)
  instead of fixed-interval polling, on native and wasm.
- Checkpoints, filters (`doc_ids`, Mango `selector`, JS function), and events
  as in the Rust crate. A JS filter function runs in the page, so it only
  applies when the source is local.
- `retry: true` uses exponential backoff and pauses while `navigator.onLine`
  is false (browser), resuming on the `online` event.

## Async runtime on wasm

A small runtime module (in `rouchdb-core`, cfg'd by target) provides `spawn`
(Tokio on native, `wasm_bindgen_futures::spawn_local` on wasm), `sleep` /
`sleep_until` / `timeout` (Tokio vs `setTimeout`), and `Instant` (`web-time`
on wasm). `tokio::sync` and `tokio::select!` keep working on wasm and stay.
All library crates use the module instead of `tokio::spawn` / `tokio::time`.

## Views and design documents

- `db.query(mapFn, opts)` and persistent views with JS `map` / `reduce`
  functions, plus the built-in reducers (`_count`, `_sum`, `_stats`) in Rust.
- Design documents replicated from CouchDB keep their JS source losslessly.
  Running that source needs `new Function`, which some CSPs forbid, so it is
  **opt-in**: `Database.open(name, { evalDesignDocs: true })`. Without it,
  `query("ddoc/view")` errors with a clear message unless a JS function was
  registered for that view.

## Edge cases

- Deleted documents and tombstones replicate; `_conflicts` and
  `_deleted_conflicts` work as natively.
- Very deep JSON: the wasm stack is smaller than native threads; the build
  sets a larger stack (`-C link-arg=-zstack-size=`) and the Mango/JSON depth
  limits (`MAX_NESTING_DEPTH`) are tested in the browser.
- Quota exceeded (IndexedDB), private browsing (Safari, Firefox), storage
  eviction: typed errors, documented.
- Tab closed mid-transaction: IndexedDB aborts the transaction; no partial
  `bulk_docs`.
- Page reload during live replication: resumes from the checkpoint.
- Clock: revision hashes do not depend on time; seq is a counter.
- Large attachments (100 MB+): stored and returned as `Blob` without
  materialising base64; replication streams where the platform allows.
- Same database opened twice in one page: one shared connection and change
  notifier.

## Acceptance criteria

- [ ] Every library crate except `rouchdb-adapter-redb`'s `std::fs` paths
      builds for `wasm32-unknown-unknown`; CI checks it.
- [ ] The native test suites (unit + CouchDB) pass unchanged.
- [ ] The adapter conformance suite passes for memory, redb, IndexedDB
      (Chromium, Firefox, WebKit) and redb-on-Node-fs.
- [ ] Every API call in the TypeScript section has a test in Node and in the
      three browser engines.
- [ ] Replication (one-shot, live, two-way, filtered, with attachments,
      with conflicts) against CouchDB 3.5.1 passes from Node and browsers.
- [ ] A document written by PouchDB replicates to RouchDB-JS and back through
      CouchDB unchanged (revs, conflicts, attachments).
- [ ] Pulling a document whose attachment is unchanged does not re-download
      the attachment (asserted on the HTTP requests made).
- [ ] Attachments are stored as `Blob` in IndexedDB and returned without base64.
- [ ] Two tabs on the same database see each other's changes in their live
      feeds within 1 s.
- [ ] Type tests pass; no `any` in the published `.d.ts`.
- [ ] Package installs and works in Vite, webpack, Node ESM, Node CJS,
      Electron, Tauri and Capacitor example apps.
- [ ] Performance and size targets in the PRD measured in CI.
