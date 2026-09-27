# DuckDB in the browser — LiveCodes POC

A proof-of-concept that runs **DuckDB compiled to WebAssembly** entirely client-side — no backend,
no code leaves the browser. This is a first step toward adding DuckDB as a language/processor in
[LiveCodes](https://livecodes.io).

## What it does

`index.html` is a self-contained SQL scratchpad:

- boots DuckDB-Wasm (`@duckdb/duckdb-wasm@1.32.0`) in a Web Worker,
- lets you write and run arbitrary DuckDB SQL,
- renders the Arrow result set as a table.

Everything (the WASM engine, the worker, the SQL) runs locally. The only network access is fetching
the engine assets from the jsDelivr CDN on first load.

## Run it

Web Workers (and ES modules) are blocked on `file://` URLs, so open the page over `http(s)`.
Any static host works — this is static hosting, not a code-execution server:

```bash
npx serve .          # or: python -m http.server 8080
# then open http://localhost:3000 (or :8080)
```

Everything else — query parsing, execution, results — happens in the browser.

## How it works

1. `import * as duckdb from 'https://cdn.jsdelivr.net/npm/@duckdb/duckdb-wasm@1.32.0/+esm'`
   The `+esm` endpoint bundles the library *and* its `apache-arrow` dependency, so no import map or
   build step is needed.
2. `duckdb.getJsDelivrBundles()` + `duckdb.selectBundle()` pick the right WASM bundle for the browser.
3. A `Blob` worker (`importScripts("<cdn worker>")`) works around the same-origin rule for workers,
   then `new duckdb.AsyncDuckDB(...)` + `db.instantiate(mainModule, pthreadWorker)` start the engine.
4. `conn.query(sql)` returns an Arrow `Table`, rendered to HTML. `BigInt`/`NULL` cells are handled.

## Notes & caveats

- Assets are loaded from the CDN and can be **self-hosted** (`dist/duckdb-*.wasm`,
  `duckdb-browser-*.worker.js`) for fully offline use — relevant for LiveCodes' offline/self-hosted builds.
- The npm `latest` tag currently points to a dev build; this POC deliberately pins the stable `1.32.0`.
- Multi-statement scripts are not split — one statement per run (enough for a POC).

## Next steps toward LiveCodes

- Wrap `start()`/`run()` as a **language processor** (`loadLanguage` + `compile` + `run`) registered in
  the languages registry, mirroring the existing WASM languages (e.g. Python/Pyodide, SQL/SQL.js).
- Declare the CDN assets in the shared deps/config module (with a self-hosted fallback).
- Support the standard LiveCodes editor interface: `run(code)` + a console/result renderer rather than
  the ad-hoc DOM here.
- Add tests (browser) and docs/examples for the `duckdb` language.
