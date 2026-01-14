# Webpack core: idea and implementation (repo overview)

This document is a high-level map of webpack’s core architecture as implemented in this repository: the main objects involved in a build, how they connect, and where to start reading.

## Core idea

Webpack is a **module graph compiler**:

- Start from one or more **entry points**.
- **Resolve** each import/require to a module, forming a dependency graph.
- **Transform** modules as needed (loaders), and **analyze** them (parsers) to discover more dependencies.
- Split the graph into one or more **chunks** (initial + async/code-split).
- **Emit** compiled assets (JS/CSS/etc.) and a small runtime to load chunks.

## Public entry points

- `lib/index.js`: the package entry that exports the `webpack` function and exposes many built-in plugins via lazy getters.
- `lib/webpack.js`: implements `webpack(options, callback)`; validates/normalizes options, applies defaults, creates a `Compiler`/`MultiCompiler`, applies plugins, then runs or watches.
- `bin/webpack.js`: CLI shim that delegates to `webpack-cli` (webpack itself is primarily a library).

## The build pipeline (very roughly)

1. **Options**: normalize + apply defaults, then apply built-in plugins based on options.
2. **Compiler run/watch**: orchestrates compilation(s) and output emission.
3. **Compilation**: build modules, seal/optimize into chunks, generate assets.
4. **Emit**: write assets to the configured output filesystem and report stats.

## `Compiler`: the orchestrator

- `lib/Compiler.js`: long-lived object for a build session.
  - Owns most lifecycle hooks (Tapable) used by plugins.
  - `run()`/`watch()` drive compilation.
  - `compile()` creates a new `Compilation`, runs `make`/`finishMake`, then calls `compilation.finish()` and `compilation.seal()`, and finally triggers `afterCompile`.
  - After compilation, `emitAssets()` writes results and triggers `emit`/`afterEmit`, records, and `done`.

## `Compilation`: per-build state + optimization

- `lib/Compilation.js`: per-build object holding:
  - modules, dependencies, and the module graph
  - chunks/chunk groups and the chunk graph
  - assets (and associated metadata), errors, warnings
- `seal()` is the “turn the module graph into output” phase:
  - create chunks from entries
  - build the chunk graph
  - run optimization hooks and assign module/chunk ids
  - prepare for code generation and emitting

## Module creation: resolving + loader pipelines

- `lib/NormalModuleFactory.js`: turns a “request” into a buildable module.
  - Applies module rules (loaders, type, parser/generator options, etc.).
  - Uses `enhanced-resolve` for resolution and `loader-runner` to execute loader chains.
- `lib/javascript/JavascriptParser.js`: JS parser based on Acorn that discovers dependency edges (ESM, CommonJS, `import()`, etc.).

## Built-in feature wiring: “everything is plugins”

Webpack core is heavily plugin-driven:

- User plugins are applied in `lib/webpack.js` (functions or `{ apply(compiler) {} }` objects).
- Most built-in behavior is implemented as plugins and enabled based on config in `lib/WebpackOptionsApply.js` (e.g. ESM/Harmony, CommonJS, JSON modules, asset modules, chunk loading, stats defaults).
- Hooks come from Tapable and are the core extension mechanism.

## Related internals docs

- Hot Module Replacement (HMR) internals: `HMR.md`

