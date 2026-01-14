# Hot Module Replacement (HMR) in webpack (core)

This document describes how webpack’s Hot Module Replacement works internally: what webpack emits at build time, what runtime code it injects, and how updates are applied. The **transport** that tells the browser “an update is available” (WebSocket/SSE/polling) is typically provided by `webpack-dev-server` or other tooling, not webpack core.

## Big picture

HMR has two halves:

1. **Build-time:** on rebuild, webpack generates a “hot update manifest” plus “hot update chunks” containing only changed module factories.
2. **Runtime:** the bundle runtime exposes `module.hot` / `import.meta.webpackHot` APIs, downloads the manifest + update chunks, figures out what can be safely replaced, disposes old modules, swaps in new factories, and runs accept handlers.

## Build-time: producing hot updates

Webpack’s core HMR implementation is driven by `HotModuleReplacementPlugin`:

- `lib/HotModuleReplacementPlugin.js`
- `lib/HotUpdateChunk.js`

### 1) Parsing `module.hot.*` / `import.meta.webpackHot.*`

During parsing, the plugin wires hooks for:

- `module.hot.accept(...)` / `module.hot.decline(...)`
- `import.meta.webpackHot.accept(...)` / `import.meta.webpackHot.decline(...)`

and turns string requests into dependency objects such as:

- `lib/dependencies/ModuleHotAcceptDependency.js`
- `lib/dependencies/ModuleHotDeclineDependency.js`
- `lib/dependencies/ImportMetaHotAcceptDependency.js`
- `lib/dependencies/ImportMetaHotDeclineDependency.js`

This matters because HMR is ultimately keyed by **module id**, so webpack needs to resolve the accept/decline requests into module ids at build time.

### 2) Recording hashes to detect changes

On each compilation, the plugin records (in `compilation.records`) the hashes needed to compute incremental updates on the next build (chunk hashes, module hashes, runtime “keys”, etc.). On the next build it compares old vs new to determine:

- which chunks have updates
- which modules were removed
- which chunks were removed

### 3) Emitting update assets

When there are changes, webpack emits:

- A **hot update manifest** (by default):
  - `output.hotUpdateMainFilename`: ``[runtime].[fullhash].hot-update.json``
- One or more **hot update chunks** (by default):
  - `output.hotUpdateChunkFilename`: ``[id].[fullhash].hot-update.js``

Default filename patterns are defined in `lib/config/defaults.js`.

The manifest JSON shape produced by `HotModuleReplacementPlugin` looks like:

- `c`: updated chunk ids
- `r`: removed chunk ids
- `m`: removed module ids
- `css` (optional): CSS-related removals when `experiments.css` is enabled

The update chunk files contain “moreModules” (new module factory functions) and are wired to call a runtime callback (see “Chunk loading / download” below).

### 4) Injecting runtime requirements

The plugin ensures the HMR runtime is attached by adding:

- `lib/hmr/HotModuleReplacementRuntimeModule.js`

This runtime module injects the core `module.hot` implementation and expects a chunk-loading runtime to provide:

- `__webpack_require__.hmrM` (download manifest)
- `__webpack_require__.hmrC` (download update chunks)
- `__webpack_require__.hmrI` (invalidate handlers)

These names are defined in `lib/RuntimeGlobals.js`.

## Runtime: `module.hot` API + state machine

The core HMR runtime code is `lib/hmr/HotModuleReplacement.runtime.js`, embedded via `HotModuleReplacementRuntimeModule`.

### 1) Attaching `module.hot` and tracking parents/children

The runtime uses `__webpack_require__.i` (intercept module execution) to:

- attach a `hot` object to each module as it executes
- wrap `require()` so that parent/child relationships are tracked (`module.parents` / `module.children`)

This parent graph is critical for deciding update propagation.

### 2) The HMR lifecycle (`check` → `apply`)

Key APIs:

- `module.hot.check([autoApply])`
- `module.hot.apply(options)`
- `module.hot.accept(...)` / `decline(...)` / `dispose(...)` / `invalidate()`
- `module.hot.status(...)` + status handlers
- `module.hot.data` (persisted across dispose/apply)

The runtime maintains a status machine roughly like:

`idle → check → prepare → ready → dispose → apply → idle` (with abort/fail states on errors).

At a high level:

1. `check()` calls `__webpack_require__.hmrM()` to fetch the update manifest.
2. If updates exist, it calls each registered download handler in `__webpack_require__.hmrC` to load update chunks and collect “apply handlers”.
3. `apply()` runs all apply handlers; if any handler reports an error, HMR aborts/fails and a full reload is usually required.

## Runtime: applying JavaScript updates

Applying JS module updates is implemented by a chunk-loading-specific runtime generated from:

- `lib/hmr/JavascriptHotModuleReplacement.runtime.js`
- `lib/hmr/JavascriptHotModuleReplacementHelper.js`

There is one “type” per chunk loading mechanism (e.g. `"jsonp"` for the browser, `"require"` for Node.js). Each type installs:

- a handler in `__webpack_require__.hmrC[type]` to download and stage updates
- a handler in `__webpack_require__.hmrI[type]` to support `module.hot.invalidate()`

### Update propagation algorithm (why accepts/declines matter)

For each updated module id, the apply handler computes “affected module effects” by walking up through `module.parents`:

- If the module is **self-declined**: abort.
- If a parent **declines** that dependency: abort.
- If the update reaches a “main” entry module without being accepted: abort (“unaccepted”).
- If a parent **accepts** the dependency: stop propagation there; mark that parent’s accept callback to run and mark the child module as outdated.

From this it derives:

- `outdatedModules`: modules to dispose (removed from cache)
- `outdatedDependencies`: per-parent accepted dependencies (to run accept callbacks for)

### Dispose phase

For each outdated module:

- call dispose handlers; store `module.hot.data` into `__webpack_require__.hmrD`
- mark the module inactive
- delete it from the module cache
- clean up parent/child links

### Apply phase

The runtime then:

- swaps in the new factories into `__webpack_require__.m` (module factories)
- runs accept callbacks for parents that accepted dependencies
- re-requires “self-accepted” modules (optionally using their error handlers)

## Chunk loading / download (example: web JSONP)

How update chunks and the manifest are fetched depends on the chunk loading runtime.

For the browser JSONP chunk loader (`lib/web/JsonpChunkLoadingRuntimeModule.js`):

- `__webpack_require__.hmrM` uses `fetch()` to read the manifest JSON.
- update chunks are loaded by inserting a `<script>` pointing at `getChunkUpdateScriptFilename(chunkId)`.
- when an update chunk arrives, it calls a global callback (based on `output.hotUpdateGlobal`) with:
  - `chunkId`
  - `moreModules` (module factories)
  - `runtime` (optional runtime code)

Other environments (ESM chunk loading, Node.js `require`, `importScripts`, etc.) implement the same concepts with different download mechanisms.

## Transport: who calls `check()`?

Webpack core provides the HMR machinery, but typically something else triggers it. This repo contains example “clients” in `hot/`:

- `hot/dev-server.js`: waits for an update signal event and runs `module.hot.check(true)`
- `hot/only-dev-server.js`: like above, but more forgiving (ignores unaccepted/declined/errored updates)
- `hot/poll.js`: polls `check(true)` periodically

In practice, `webpack-dev-server` (or similar tooling) recompiles on file changes, serves the updated assets, and notifies the browser to call into the HMR runtime.

## Source map / “where to look”

- Build-time diffing + update asset emission: `lib/HotModuleReplacementPlugin.js`
- Core `module.hot` runtime: `lib/hmr/HotModuleReplacement.runtime.js`
- JS update apply logic: `lib/hmr/JavascriptHotModuleReplacement.runtime.js`
- Web (JSONP) download + callback wiring: `lib/web/JsonpChunkLoadingRuntimeModule.js`
- Runtime global names: `lib/RuntimeGlobals.js`

