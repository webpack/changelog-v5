[Help by editing this file](https://github.com/webpack/changelog-v5/edit/master/README.md).

# General direction

This release focus on the following:

- We try to improve build performance with Persistent Caching.
- We try to improve Long Term Caching with better algorithms and defaults.
- We try to improve bundle size with better Tree Shaking and Code Generation
- We try to cleanup internal structures that were left in a weird state, while implementing features in v4 without introducing any breaking changes.
- We try to prepare for future features by introducing breaking changes now, allowing us to stay on v5 for as long as possible.

# Migration Guide

=> [see here for a migration guide](https://github.com/webpack/changelog-v5/blob/master/MIGRATION%20GUIDE.md) <=

# Major Changes

## Removed Deprecated Items

All items deprecated in v4 were removed.

MIGRATION: Make sure that your webpack 4 build does not print deprecation warnings.

Here are a few things that were removed but did not have deprecation warnings in v4:

- IgnorePlugin and BannerPlugin must now be passed an options object.

## Deprecation codes

New deprecations include a deprecation code so they are easier to reference.

(since beta.7)

## Syntax deprecated

`require.include` has been deprecated and will emit a warning by default when used.

Behavior can be changed with `Rule.parser.requireInclude` to allowed, deprecated or disabled.

(since beta.1)

## Automatic Node.js Polyfills Removed

In the early days, webpack's aim was to allow running most node.js modules in the browser, but the module landscape changed and many module uses are now written mainly for frontend purposes. webpack <= 4 ships with polyfills for many of the node.js core modules, which are automatically applied once a module uses any of the core modules (i.e. the `crypto` module).

While this makes using modules written for node.js easy, it adds these huge polyfills to the bundle. In many cases these polyfills are unnecessary.

webpack 5 stops automatically polyfilling these core modules and focuses on frontend-compatible modules.

MIGRATION:

- Try to use frontend-compatible modules whenever possible.
- It's possible to manually add a polyfill for a node.js core module. An error message will give a hint on how to achieve that.
- Package authors: Use the `browser` field in `package.json` to make a package frontend-compatible. Provide alternative implementations/dependencies for the browser.

## Deterministic Chunk and Module IDs

New algorithms were added for long term caching. These are enabled by default in production mode.

`chunkIds: "deterministic", moduleIds: "deterministic"`

The algorithms assign short (3 or 4 characters) numeric IDs to modules and chunks in a deterministic way.
This is a trade-off between bundle size and long term caching.

`moduleIds/chunkIds: false` disables the default behavior and one can provide a custom algorithm via plugin. Note that in webpack 4 `moduleIds/chunkIds: false` without custom plugin resulted in a working build, while in webpack 5 you must provide a custom plugin.

MIGRATION: Best use the default values for `chunkIds` and `moduleIds`. You can also opt-in to the old defaults `chunkIds: "size", moduleIds: "size"`, this will generate smaller bundles, but invalidate them more often for caching.

Note: In webpack 4 hashed module ids yielded reduced gzip performance. This was related to changed module order and has been fixed. (since beta.1)

## Deterministic Mangled Export Names

A new algorithm was added to mangling export names. It's enabled by default.

It will mangle export names when possible in a deterministic way.

MIGRATION: Nothing to do.

## Named Chunk IDs

A new named chunk id algorithm enabled by default in development mode gives chunks (and filenames) human-readable names.
A Module ID is determined by its path, relative to the `context`.
A Chunk ID is determined by the chunk's content.

So you no longer need to use `import(/* webpackChunkName: "name" */ "module")` for debugging.
But it would still make sense if you want to control the filenames for production environments.

It's possible to use `chunkIds: "named"` in production, but make sure not to accidentally expose sensitive information about module names.

MIGRATION: If you dislike the filenames being changed in development, you can pass `chunkIds: "natural"` to use the old numeric mode.

## JSON modules

JSON modules now align with the spec and emit a warning when using a non-default export.

MIGRATION: Use the default export.

(since alpha.16)

Unused properties are dropped by the `optimization.usedExports` optimization and properties are mangled by the `optimization.mangleExports` optimization. (since beta.3)

JSON modules no longer have named exports when importing from a string EcmaScript module (since beta.7)

It's possible to specify a custom JSON parser in `Rule.parser.parse` to import JSON-like files (e. g. for toml, yaml, json5, etc.). (since beta.8)

## Nested tree-shaking

webpack is now able to track access to nested properties of exports. This can improve Tree Shaking (Unused export elimination and export mangling) when reexporting namespace objects.

``` js
// inner.js
export const a = 1;
export const b = 2;

// module.js
import * as inner from "./inner";
export { inner }

// user.js
import * as module from "./module";
console.log(module.inner.a);
```

In this example, the export `b` can be removed in production mode.

(since alpha.15)

## Inner-module tree-shaking

webpack 4 didn't analyze dependencies between exports and imports of a module. webpack 5 has a new option `optimization.innerGraph`, which is enabled by default in production mode, that runs an analysis on symbols in a module to figure out dependencies from exports to imports.

In a module like this:

``` js
import { something } from "./something";

function usingSomething() {
  return something;
}

export function test() {
  return usingSomething();
}
```

The inner graph algorithm will figure out that `something` is only used when the `test` export is used. This allows to flag more exports as unused and to omit more code from the bundle.

When `"sideEffects": false` is set, this allows to omit even more modules. In this example `./something` will be omitted when the `test` export is unused.

To get the information about unused exports `optimization.unusedExports` is required. To remove side-effect-free modules `optimization.sideEffects` is required.

The following symbols can be analysed:
* function declarations
* class declarations
* `export default` with or variable declarations with
  * function expressions
  * class expressions
  * `/*#__PURE__*/` expressions
  * local variables
  * imported bindings

FEEDBACK: If you find something missing in this analysis, please report an issue and we consider adding it.

This optimization is also known as Deep Scope Analysis.

(since alpha.24)

Using `eval()` will bail-out this optimization for a module, because evaled code could reference any symbol in scope.

(since beta.10)

## CommonJs Tree Shaking

webpack used to opt-out from used exports analysing for CommonJs exports and `require()` calls.

webpack 5 adds support for some CommonJs constructs, allows to eliminate unused CommonJs exports and track referenced export names from `require()` calls.

The following constructs are supported:

- `exports|this|module.exports.xxx = ...`
- `Object.defineProperty(exports|this|module.exports, "xxx", ...)`
- `require("abc").xxx`
- `require("abc").xxx()`
- importing from ESM
- `require()` a ESM
- flagged exportType (special handling for non-strict ESM import):
  - `Object.defineProperty(exports|this|module.exports, "__esModule", { value: true|!0 })`
  - `exports|this|module.exports.__esModule = true|!0`

(since beta.9)

## Compiler Idle and Close

Compilers now need to be closed after being used. Compilers now enter and leave idle state and have hooks for these states. Plugins may use these hooks to do unimportant work. (i. e. the Persistent cache slowly stores the cache to disk). On compiler close - All remaining work should be finished as fast as possible. A callback signals the closing as done.

Plugins and their respective authors should expect that some users may forget to close the Compiler. So, all work should eventually be finishing while in idle too. Processes should be prevented from exiting when the work is being done.

The `webpack()` facade automatically calls `close` when being passed a callback.

MIGRATION: While using the node.js API, make sure to call `Compiler.close` when done.

## Improved Code Generation

There is a new option `output.ecmaVersion` now. It allows specifying the EcmaScript version for runtime code generated by webpack.

webpack 4 used to only emit ES5 code. 

webpack 5 can generate both ES5 and ES6/ES2015 code now. 

The default configuration will generate ES2015 code. If you need to support older browser (like IE11), you can decrease this to `output.ecmaVersion: 5`.

Choosing `output.ecmaVersion: 2015` will generate shorter code using arrow functions and more spec-comform code using `const` declarations with TDZ for `export default`.

(since alpha.23)

The default minimizing in production mode also used this `ecmaVersion` option to generate smaller code. (since alpha.31)

## SplitChunks and Module Sizes

Modules now express size in a better way than displaying a single number. Also, there are now different types of sizes.

The SplitChunksPlugin now knows how to handle these different sizes and uses them for `minSize` and `maxSize`.
By default, only `javascript` size is handled, but you can now pass multiple values to manage them:

```js
minSize: {
  javascript: 30000,
  style: 50000,
}
```

MIGRATION: Check which types of sizes are used in your build and configure these in `splitChunks.minSize` and optionally in `splitChunks.maxSize`.

## Persistent Caching

There is now a filesystem cache. It's opt-in and can be enabled with the following configuration:

``` js
cache: {
  // 1. Set cache type to filesystem
  type: "filesystem",
  
  buildDependencies: {
    // 2. Add your config as buildDependency to get cache invalidation on config change
    config: [__filename]
  
    // 3. If you have other things the build depends on you can add them here
    // Note that webpack, loaders and all modules referenced from your config are automatically added
  }
}
```

Important notes:

By default webpack assumes that the `node_modules` directory, which webpack is inside of, is **only** modified by a package manager. Hashing and timestamping is skipped for node_modules. Instead only the package name and version is used for performance reasons. Symlinks (i. e. `npm/yarn link`) are fine. Do not edit files in `node_modules` directly unless you opt-out of this optimization with `cache.managedPaths: []`

The cache will be stored into `node_modules/.cache/webpack` (when using node_modules) resp. `.pnp/.cache/webpack` (when using Yarn PnP, since alpha.21) by default. You probably never have to delete it manually.

(since alpha.20)

When using Yarn PnP webpack assumes that the yarn cache is immutable (which it usually is). You can opt-out of this optimization with `cache.immutablePaths: []`

(since alpha.21)

`SourceMapDevToolPlugin` uses the Persistent Cache.

(since beta.4)

`ConcatenatedModule` used the Persistent Cache.

(since beta.10)

`ProgressPlugin` used the Persistent Cache.

(since beta.14)

## File Emitting

webpack used to always emit all output files during first build, but skipped writing unchanged files during incremental (watch) builds.
It is assumed that nothing else changes output files while webpack is running.

With Persistent Caching added a watch-like experience should be given even when restarting the webpack process, but it would be a too strong assumption to think that nothing else changes the output directory even when webpack is not running.

So webpack will now check existing files in the output directory and compares their content with the output file in memory. It will only write the file when it has been changed.
This is only done on first build. Any incremental build will always write the file when a new asset has been generated in the running webpack process.

We assume that webpack and plugins only generate new assets when content has been changed. Caching should be used to ensure that no new asset is generated when input is equal.
Not following this advise will degrade performance.

Files that are flagged as `immutable` (including a content hash), will never be written when a file with the same name already exists.
We assume that content hash will change when file content changes. This is true in general, but might not be always true during webpack or plugin development.

(since beta.3)

## SplitChunks for single-file-targets

Targets that only allow to startup a single file (like node, WebWorker, electron main) now supports loading the dependent pieces required for bootstrap automatically by the runtime.

This allows to use `splitChunks` for these targets with `chunks: "all"`.

Note that since chunk loading is async, this makes initial evaluation async too. This can be an issue when using `output.library`, since the exported value is a Promise now. Since alpha.14 this does not apply to `target: "node"` since chunk loading is sync here.

(since alpha.3)

## Updated Resolver

`enhanced-resolve` was updated to v5. This has the following improvements:

- When Yarn PnP is used, the resolver will handle it without an additional plugin
- The resolve tracks more dependencies, like missing files
- aliasing may have multiple alternatives
- aliasing to `false` is possible now
- Increased performance

(since alpha.18)

## Chunks without JS

Chunks that contain no JS code, will no longer generate a JS file.

(since alpha.14)

## Experiments

Not all features are stable from the beginning. In webpack 4 we added experimental features and noted in changelog that they are experimental, but it was not always clear from the configuration that these features are experimental.

In webpack 5 there is a new `experiments` config option which allows to enabled experimental features. This makes it clear which ones are enabled/used.

While webpack follows semantic versioning, it will make an exception for experimental features. Experimental features might contain breaking changes in minor webpack versions. When this happens we will add a clear note into the changelog. This will allow us to interate faster for experimental features, while also allowing us to stay longer on a major version for stable features.

The following experiments will ship with webpack 5:

* `.mjs` support like in webpack 4 (`experiments.mjs`)
* Old WebAssembly support like in webpack 4 (`experiments.syncWebAssembly`)
* New WebAssembly support acording to the [updated spec](https://github.com/WebAssembly/esm-integration) (`experiments.asyncWebAssembly`)
  * This makes a WebAssembly module an async module
* [Top Level Await](https://github.com/tc39/proposal-top-level-await) Stage 3 proposal (`experiments.topLevelAwait`)
  * Using `await` on top-level makes the module an async module
* Importing async modules with `import` (`experiments.importAsync`)
* Importing async modules with `import await` (`experiments.importAwait`)
* The `asset` module type which is similar to the `file-loader`|`url-loader`|`raw-loader` (`experiments.asset`) (since alpha.19)
  * DataUrls and options related to that are supported since beta.8
* Emitting bundle as module (`experiments.outputModule`) (since alpha.31)
  * This removed the wrapper IIFE from the bundle, enforces strict mode, lazy loads via `<script type="module">` and minimized in module mode

Note that this also means `.mjs` support and WebAssembly support are now disabled by default.

(since alpha.15)

## Stats

Chunk relations are hidden by default now. This can be toggled with `stats.chunkRelations`.

(since alpha.1)

Stats differentiate between `files` and `auxiliaryFiles` now.

(since alpha.19)

Stats hide module and chunk ids by default now. This can be toggled with `stats.ids`.

The list of all modules are sorted by distance to entrypoint now. This can be changed with `stats.modulesSort`.

The list of chunk modules resp. chunk root modules is sorted by module name now. This can be changed with `stats.chunkModulesSort` resp. `stats.chunkRootModulesSort`.

The list of nested modules in concatenated modules is sorted topological now. This can be changed with `stats.nestedModulesSort`.

Chunks and Assets show chunk id hints now.

(since alpha.31)

## Progress

A few improvements has been done to the `ProgressPlugin` which is used for `--progress` by the CLI, but can also be used manually as plugin.

It used to only count the processed modules. Now it can count `entries` `dependencies` and `modules`.
All of them are shown by default now.

It used to disable the currently processed module. This caused much process stderr output and yielded a performance problem on some consoles.
This is now disabled by default (`activeModules` option). This also reduces the amount of spam on the console. (since alpha.31)

Now writing to stderr is throttled to 500ms.

(since beta.4)

The profiling mode also got an upgrade and will display timings of nested progress messages.
This makes it easier to figure out while plugin is causing performance problems.

(since beta.10)

Added `percentBy`-options that tells `ProgressPlugin` how to calculate progress percentage.

```js
new webpack.ProgressPlugin({ percentBy: "entries" });
```

To make progress percentage more accurate `ProgressPlugin` caches the last known total modules count and reuses this value on the next build. The first build will warm the cache but the following builds will use and update this value.

(since beta.14)

## Minimum Node.js Version

The minimum supported node.js version has increased from 6 to 10.13.0(LTS).

MIGRATION: Upgrade to the latest node.js version available.

# Changes to the Configuration

## Changes to the Structure

- `cache: Object` removed: Setting to a memory-cache object is no longer possible
- `cache.type` added: It's now possible to choose between `"memory"` and `"filesystem"`
- New configuration options for `cache.type = "filesystem"` added:
  - `cache.cacheDirectory`
  - `cache.name`
  - `cache.version`
  - `cache.store`
  - ~`cache.loglevel`~ (removed since alpha.20)
  - `cache.hashAlgorithm`
  - `cache.idleTimeout` (since alpha.8)
  - `cache.idleTimeoutForIntialStore` (since alpha.8)
  - `cache.managedPaths` (since alpha.20)
  - `cache.immutablePaths` (since alpha.21)
  - `cache.buildDependencies` (since alpha.20)
- `resolve.cache` added: Allows to disable/enable the safe resolve cache
- `resolve.concord` removed
- `resolve.alias` values can be arrays or `false` now (since alpha.18)
- Automatic polyfills for native node.js modules were removed
  - `node.Buffer` removed
  - `node.console` removed
  - `node.process` removed
  - `node.*` (node.js native module) removed
  - MIGRATION: `resolve.alias` and `ProvidePlugin`. Errors will give hints.
- `output.filename` can now be a function (since alpha.17)
- `output.assetModuleFilename` added (since alpha.19)
- `devtool` is more strict (since beta.1)
  - Format: `false | eval | [inline-|hidden-|eval-][nosources-][cheap-[module-]]source-map`
- `optimization.chunkIds: "deterministic"` added
- `optimization.moduleIds: "deterministic"` added
- `optimization.moduleIds: "hashed"` deprecated
- `optimization.moduleIds: "total-size"` removed
- Deprecated flags for module and chunk ids were removed
  - `optimization.hashedModuleIds` removed
  - `optimization.namedChunks` removed (`NamedChunksPlugin` too)
  - `optimization.namedModules` removed (`NamedModulesPlugin` too)
  - `optimization.occurrenceOrder` removed
  - MIGRATION: Use `chunkIds` and `moduleIds`
- `optimization.splitChunks` `test` no longer matches chunk name
  - MIGRATION: Use a test function
    `(module, { chunkGraph }) => chunkGraph.getModuleChunks(module).some(chunk => chunk.name === "name")`
- `optimization.splitChunks` `minRemainingSize` was added (since alpha.13)
- `optimization.splitChunks` `filename` can now be a function (since alpha.17)
- `optimization.splitChunks` sizes can now be objects with a size per source type
  - `minSize`
  - `minRemainingSize`
  - `maxSize`
  - `maxAsyncSize` (since alpha.13)
  - `maxInitialSize`
- `optimization.splitChunks` `maxAsyncSize` and `maxInitialSize` added next to `maxSize`: allows to specify different max sizes for initial and async chunks
- `optimization.splitChunks` `name: true` removed: Automatic names are no longer supported
  - MIGRATION: Use the default. `chunkIds: "named"` will give your files useful names for debugging
- `optimization.splitChunks.cacheGroups[].idHint` added: Gives a hint how the named chunk id should be chosen
- `optimization.splitChunks` `automaticNamePrefix` removed
  - MIGRATION: Use `idHint` instead
- `optimization.splitChunks` `filename` is no longer restricted to initial chunks (since alpha.11)
- `optimization.mangleExports` added (since alpha.10)
- `output.devtoolLineToLine` removed
  - MIGRATION: No replacement
- `output.hotUpdateChunkFilename: Function` is now forbidden: It never worked anyway.
- `output.hotUpdateMainFilename: Function` is now forbidden: It never worked anyway.
- `module.rules` `resolve` and `parser` will merge in a different way (objects are deeply merged, array may include `"..."` to reference to prev value) (since alpha.13)
- `module.rules` `query` and `loaders` were removed (since alpha.13)
- `module.rules` `options` passing a string is deprecated (since beta.10)
  - MIGRATION: Pass an options object instead, open an issue on the loader when this is not supported
- `stats.chunkRootModules` added: Show root modules for chunks
- `stats.orphanModules` added: Show modules which are not emitted
- `stats.runtime` added: Show runtime modules
- `stats.chunkRelations` added: Show parent/children/sibling chunks (since alpha.1)
- `stats.errorStack` added: Show webpack-internal stack trace of errors (since beta.1)
- `stats.preset` added: select a preset (since alpha.1)
- `BannerPlugin.banner` signature changed
  - `data.basename` removed
  - `data.query` removed
  - MIGRATION: extract from `filename`
- `SourceMapDevToolPlugin` `lineToLine` removed
  - MIGRATION: No replacement
- `[hash]` as hash for the full compilation is now deprecated
  - MIGRATION: Use `[fullhash]` instead or better use another hash option
- `[modulehash]` is deprecated
  - MIGRATION: Use `[hash]` instead
- `[moduleid]` is deprecated
  - MIGRATION: Use `[id]` instead
- `[filebase]` removed
  - MIGRATION: Use `[base]` instead
- New placeholders for file-based templates (i. e. SourceMapDevToolPlugin)
  - `[name]`
  - `[base]`
  - `[path]`
  - `[ext]`
- `externals` when passing a function, it has now a different signature `({ context, request }, callback)`
  - MIGRATION: Change signature
- `experiments` added (see Experiments section above, since alpha.19)
- `watchOptions.followSymlinks` added (since alpha.19)

## Changes to the Defaults

- `module.unsafeCache` is now only enabled for `node_modules` by default
- `optimization.moduleIds` defaults to `deterministic` in production mode, instead of `size`
- `optimization.chunkIds` defaults to `deterministic` in production mode, instead of `total-size`
- `optimization.nodeEnv` defaults to `false` in `none` mode
- `optimization.splitChunks` `minRemainingSize` defaults to `minSize` (since alpha.13)
  - This will lead to less splitted chunks created in cases where the remaining part would be too small
- `optimization.splitChunks.cacheGroups.vendors` has be renamed to `optimization.splitChunks.cacheGroups.defaultVendors`
- `optimization.splitChunks.cacheGroups.defaultVendors.reuseExistingChunk` now defaults to `true` (since beta.7)
- `resolve(Loader).cache` defaults to `true` when `cache` is used
- `resolve(Loader).cacheWithContext` defaults to `false`
- ~`node.global` defaults to `false`~ (since alpha.4 removed)
- `resolveLoader.extensions` remove `.json` (since alpha.8)
- `node.global` `node.__filename` and `node.__dirname` defaults to `false` in node-`target`s (since alpha.14)
- `stats.errorStack` defaults to `false` (since beta.1)

# Major Internal Changes

The following changes are only relevant for plugin authors:

## Runtime Modules

A large part of the runtime code was moved into the so called "runtime modules". These special modules are in-charge of adding runtime code. They can be added in to any chunk, but are currently always added to the runtime chunk. "Runtime Requirements" control which runtime modules (or core runtime parts) are added to the bundle. This ensures that only runtime code that is used is added to the bundle. In the future, runtime modules could also added to an on-demand-loaded chunk, to load runtime code when needed.

In most cases the core runtime allows to inline the entry module instead of calling it with `__webpack_require__`. If there is no other module in the bundle, no `__webpack_require__` is needed at all. This combines well with Module Concatenation where multiple modules are concatenated into a single module.

In the best case no runtime code is needed at all.

MIGRATION: If you are injecting runtime code into the webpack runtime in a plugin, consider using RuntimeModules instead.

(since alpha.31)

## Serialization

A serialization mechanism was added to allow serialization of complex objects in webpack. It has an opt-in semantic, so classes that should be serialized need to be explicitly flagged (and their serialization implemented). This has been done for most Modules, all Dependencies and some Errors.

MIGRATION: When using custom Modules or Dependencies, it is recommended to make them serializable to benefit from persistent caching.

## Extensible Caching

A `Cache` class with a plugin interface has been added. This class can be used to write and read to the cache. Depending on configuration, different plugins can add the functionality to the cache. The `MemoryCachePlugin` adds in-memory caching. The `FileCachePlugin` adds persistent (file-system) caching.

The `FileCachePlugin` uses the serialization mechanism to persist and restore cached items to/from the disk.

## Hook Object Frozen

Classes with `hooks` have their `hooks` object frozen, so adding custom hooks is no longer possible this way.

MIGRATION: The recommended way to add custom hooks is using a WeakMap and a static `getXXXHooks(XXX)` (i. e. `getCompilationHook(compilation)`) method. Internal classes use the same mechanism used for custom hooks.

## Tapable Upgrade

The compat layer for webpack 3 plugins has been removed. It had already been deprecated for webpack 4.

Some less used tapable APIs were removed or deprecated. (since alpha.12)

MIGRATION: Use the new tapable API.

## Staged Hooks

For several steps in the sealing process, there had been multiple hooks for different stages. i. e. `optimizeDependenciesBasic` `optimizeDependencies` and `optimizeDependenciesAdvanced`. These have been removed in favor of a single hook which can be used with a `stage` option. See `OptimizationStages` for possible stage values.

MIGRATION: Hook into the remaining hook instead. You may add a `stage` option.

## Main/Chunk/ModuleTemplate deprecation

Bundle templating has been refactored. MainTemplate/ChunkTemplate/ModuleTemplate were deprecated and the JavascriptModulesPlugin takes care of JS templating now.

Before that refactoring JS output was handled by Main/ChunkTemplate while other output (i. e. WASM, CSS) was handled by plugins. This looks like JS is first class, while other output is second class. The refactoring changes that and all output is handled by their plugins.

It's still possible to hook into parts of the templating. The hooks are in JavascriptModulesPlugin instead of Main/ChunkTemplate now. (Yes plugins can have hooks too. I call them attached hooks.)

There is a compat-layer, so Main/Chunk/ModuleTemplate still exist, but only delegate tap calls to the new hook locations.

MIGRATION: Follow the advises in the deprecation messages. Mostly pointing to hooks at different locations.

(since alpha.31)

## Entry point descriptor

If an object is passed as entry point the value might be a string, array of strings or a descriptor:

```js
module.exports = {
  entry: {
    catalog: { 
      import: './catalog.js', 
    }
  }
};
```

Descriptor syntax might be used to pass additional options to an entry point.

### Entry point output filename

By default, the output filename for the entry chunk is extracted from `output.filename` but you can specify a custom output filename for a specific entry:

```js
module.exports = {
  entry: {
    about: { import: './about.js', filename: 'pages/[name][ext]' }
  }
};
```

### Entry point dependency

By default, every entry chunk stores all the modules that it uses. With `dependOn`-option you can share the modules from one entry chunk to another:

```js
module.exports = {
  entry: {
    app: { import: './app.js', dependOn: 'react-vendors' },
    'react-vendors': ['react', 'react-dom', 'prop-types']
  }
};
```

The app chunk will not contain the modules that `react-vendors` has.

(since beta.14)

## Order and IDs

webpack used to order modules and chunks in the Compilation phase, in a specific way, to assign IDs in an incremental order. This is no longer the case. The order will no longer be used for id generation, instead, the full control of ID generation is in the plugin.

Hooks to optimize the order of module and chunks have been removed.

MIGRATION: You cannot rely on the order of modules and chunks in the compilation phase no more.

## Arrays to Sets

- Compilation.modules is now a Set
- Compilation.chunks is now a Set
- Chunk.files is now a Set (since alpha.16)

There is a compat-layer which prints deprecation warnings.

MIGRATION: Use Set methods instead of Array methods.

## Compilation.fileSystemInfo

This new class can be used to access information about the filesystem in a cached way. Currently it allows to ask for both file and directory timestamps. Information about timestamps is transferred from the watcher if possible, otherwise determined by filesystem access.

In the future, asking for file content hashes will be added and modules will be able to check validity with file contents instead of file hashes.

MIGRATION: Instead of using `file/contextTimestamps` use the `compilation.fileSystemInfo` API instead.

Timestamping for directories is possible now, which allows serialization of ContextModules. (since alpha.24)

`Compiler.modifiedFiles` has been added (next to `Compiler.removedFiles`) to make it easier to reference the changed files. (since beta.7)

## Filesystems

Next to `compiler.inputFileSystem` and `compiler.outputFileSystem` there is a new `compiler.intermediateFileSystem` for all fs actions that are not considers as input or output, like writing records, cache or profiling output.

The filesystems have now the `fs` interface and do no longer demand additional methods like `join` or `mkdirp`. But if they have methods like `join` or `dirname` they are used.

(since alpha.16)

## Hot Module Replacement

HMR runtime has be refactored to Runtime Modules. `HotUpdateChunkTemplate` has been merged into `ChunkTemplate`. ChunkTemplates and plugins should also handle `HotUpdateChunk`s now.

The javascript part of HMR runtime has been separated from the core HMR runtime. Other module types can now also handle HMR in their own way. In the future, this will allow i. e. HMR for the mini-css-extract-plugin or for WASM modules.

MIGRATION: As this is a newly introduced functionality, there is nothing to migrate.

## Work Queues

webpack used to handle module processing by functions calling functions, and a `semaphore` which limits parallelism. The `Compilation.semaphore` has been removed and async queues now handle work queuing and processing. Each step has a separate queue:

- `Compilation.factorizeQueue`: calling the module factory for a group of dependencies.
- `Compilation.addModuleQueue`: adding the module to the compilation queue (may restore module from cache).
- `Compilation.buildQueue`: building the module if neccessary (may stores module to cache).
- `Compilation.rebuildQueue`: building a module again if manually triggered.
- `Compilation.processDependenciesQueue`: processing dependencies of a module.

These queues have some hooks to watch and intercept job processing.

In the future, multiple compilers may work together and job orchestration can be done by intercepting these queues.

MIGRATION: As this is a newly introduced functionality, there is nothing to migrate.

## Logging

webpack internals include some logging now.
`stats.logging` and `infrastructureLogging` options can be used to enabled these messages.

(since beta.3)

## Module and Chunk Graph

webpack used to store a resolved module in the dependency, and store the contained modules in the chunk. This is no longer the case. All information about how modules are connected in the module graph are now stored in a ModuleGraph class. All information about how modules are connected with chunks are now stored in the ChunkGraph class. Information which depends on i. e. the chunk graph, is also stored in the related class.

That means the following information about modules has been moved:

- Module connections -> ModuleGraph
- Module issuer -> ModuleGraph
- Module optimization bailout -> ModuleGraph (TODO: check if it should ChunkGraph instead)
- Module usedExports -> ModuleGraph
- Module providedExports -> ModuleGraph (since alpha.4)
- Module pre order index -> ModuleGraph
- Module post order index -> ModuleGraph
- Module depth -> ModuleGraph
- Module profile -> ModuleGraph
- Module id -> ChunkGraph
- Module hash -> ChunkGraph
- Module runtime requirements -> ChunkGraph
- Module is in chunk -> ChunkGraph
- Module is entry in chunk -> ChunkGraph
- Module is runtime module in chunk -> ChunkGraph
- Chunk runtime requirements -> ChunkGraph

webpack used to disconnect modules from the graph when restored from cache. This is no longer necessary. A Module stores no info about the graph and can technically used in multiple graphs. This makes caching easier.

There is a compat-layer for most of these changes, which prints a deprecation warning when used.

MIGRATION: Use the new APIs on ModuleGraph and ChunkGraph

## Init Fragments

`DependenciesBlockVariables` has been removed in favor of InitFragments. `DependencyTemplates` can now add `InitFragments` to inject code to the top of the module's source. `InitFragments` allows deduplication.

MIGRATION: Use `InitFragments` instead of inserting something at a negative index into the source.

## Module Source Types

Modules now have to define which source types they support via `Module.getSourceTypes()`. Depending on that, different plugins call `source()` with these types. i. e. for source type `javascript` the `JavascriptModulesPlugin` embeds the source code into the bundle. Source type `webassembly` will make the `WebAssemblyModulesPlugin` emit a wasm file. Custom source types are also supported, i. e. the mini-css-extract-plugin will probably use the source type `stylesheet` to embed the source code into a css file.

There is no relationship between module type and source type. i. e. module type `json` also uses source type `javascript` and module type `webassembly/experimental` uses source types `javascript` and `webassembly`.

MIGRATION: Custom modules need to implement these new interface methods.

## Extensible Stats

Stats `preset`, `default`, `json` and `toString` are now baked in by a plugin system. Converted the current Stats into plugins.

MIGRATION: Instead of replacing the whole Stats functionality, you can now customize it. Extra information can now be added to the stats json instead of writing a separate file.

## New Watching

The watcher used by webpack was refactored. It was previously using `chokidar` and the native dependency `fsevents` (only on OSX). Now it's only based on native node.js `fs`. This means there is no native dependency left in webpack.

It also captures more information about the filesystem while watching. It now captures mtimes and watch event times, as well as information about missing files. For this the `WatchFileSystem` API changed a little bit. While on it we also converted Arrays to Sets and Objects to Maps.

(since alpha.5)

## SizeOnlySource after emit

webpack now replaces the Sources in `Compilation.assets` with `SizeOnlySource` variants to reduce memory usage.

(since alpha.8)

## Emitting assets multiple times

The warning `Multiple assets emit different content to the same filename` has been made an error.

(since beta.7)

## ExportsInfo

The way how information about exports of modules are stored has been refactored. The ModuleGraph now features a `ExportsInfo` for each `Module`, which stores information per export. It also stores information about unknown exports and if the module is used in side-effect-only way.

For each export the following information is stored:

- Is the export used? yes, no, not statically known, not determined. (see also `optimization.usedExports`)
- Is the export provided? yes, no, not statically known, not determined. (see also `optimization.providedExports`)
- Can be export name be renamed? yes, no, not determined.
- The new name, if the export has been renamed. (see also `optimization.mangleExports`)
- Nested ExportsInfo, if the export is an object with information attached itself
  - Used for reexporting namespace objects: `import * as X from "..."; export { X };`
  - Used for representing structure in JSON modules (since beta.3)

(since alpha.10)

## Code Generation Phase

The Compilation features Code Generation as separate compilation phase now. It no longer runs hidden in `Module.source()` or `Module.getRuntimeRequirements()`.

This should make the flow much cleaner. It also allows to report progress for this phase and makes Code Generation more visible when profiling.

MIGRATION: `Module.source()` and `Module.getRuntimeRequirements()` are deprecated now. Use `Module.codeGeneration()` instead.

(since alpha.31)

## Improved Code Generation

webpack detects when ASI happens and generates shorter code when no semicolons are inserted. `Object(...)` -> `(0, ...)` (since alpha.22)

webpack merges multiple export getters into a single runtime function call: `r.d(x, "a", () => a); r.d(x, "b", () => b);` -> `r.d(x, {a: () => a, b: () => b});` (since alpha.22)

## DependencyReference

webpack used to have a single method and type to represent references of dependencies (`Compilation.getDependencyReference` returning a `DependencyReference`).
This type used to include all information about this reference like the referenced Module, which exports has been imported, if it's a weak reference and also some ordering related information.

Bundling all these information together makes getting the reference expensive and it's also called very often everytime somebody need one piece of information.

In webpack 5 this part of the codebase was refactored and the method has been split up.

- The referenced module can be read from the ModuleGraphConnection
- The imported export names can be get via `Dependency.getReferencedExports()`
- These is a `weak` flag on the `Dependency` class
- Ordering is only relevant to `HarmonyImportDependencies` and can be get via `sourceOrder` property

(since beta.2)

## Presentational Dependencies

There is now a new type of dependency in `NormalModules`: Presentational Dependencies

These dependencies are only used during the Code Generation phase, but are not used during Module Graph building.
So they can never have referenced modules or influence exports/imports.

These dependencies are cheaper to process and webpack uses them when possible

(since beta.2)

## Deprecated loaders

- [`null-loader`](https://github.com/webpack-contrib/null-loader)
  
  It will be deprecated. Use 
  
  ```js
  alias: {
      xyz$: false
  }
  ```  
  
  or use absolute path
 
  ```js
  alias: { 
    [path.resolve(__dirname, "....")]: false 
  } 
  ```
  
# Minor Changes

- Compiler.name: When generating a compiler name with absolute paths, make sure to separate them with `|` or `!` on both parts of the name.
  - Using space as a separator is now deprecated. (Paths could contain spaces)
  - Hint: `|` is replaced with space in Stats string output.
- SystemPlugin is now disabled by default.
  - MIGRATION: Avoid using it as the spec has been removed. You can re-enable it with `Rule.parser.system: true`
- ModuleConcatenationPlugin: concatenation is no longer prevented by `DependencyVariables` as they have been removed
  - This means it can now concatenate in cases of `module`, `global`, `process` or the ProvidePlugin
- `exec` removed from the loader context
  - MIGRATION: This can be implemented in the loader itself
- `Stats.presetToOptions` removed
  - MIGRATION: Use `compilation.createStatsOptions` instead
- SingleEntryPlugin and SingleEntryDependency removed
  - MIGRATION: use EntryPlugin and EntryDependency
- Chunks can now have multiple entry modules
- ExtendedAPIPlugin removed
  - MIGRATION: No longer needed, `__webpack_hash__` and `__webpack_chunkname__` can always be used and runtime code is injected where needed.
- ProgressPlugin no longer uses tapable context for `reportProgress`
  - MIGRATION: Use `ProgressPlugin.getReporter(compiler)` instead
- ProvidePlugin is now re-enabled for `.mjs` files
- Stats json `errors` and `warnings` no longer contain strings but objects with information splitted into properties.
  - MIGRATION: Access the information on the properties. i. e. `message`
- Compilation.hooks.normalModuleLoader is deprecated
  - MIGRATION: Use `NormalModule.getCompilationHooks(compilation).loader` instead
- Changed hooks in `NormalModuleFactory` from waterfall to bailing, changed and renamed hooks that return waterfall functions (since alpha.5)
- Removed `compilationParams.compilationDependencies` (since alpha.5)
  - Plugins can add dependencies to the compilation by adding to `compilation.file/context/missingDependencies`
  - Compat layer will delegate `compilationDependencies.add` to `fileDependencies.add` (since alpha.30)
- `stats.assetsByChunkName[x]` is now always an array (since alpha.5)
- `__webpack_get_script_filename__` function added to get the filename of a script file (since alpha.12)
- `getResolve(options)` in the loader API will merge options in a different way, see `module.rules` `resolve` (since alpha.13)
- `"sideEffects"` in package.json will be handled by `glob-to-regex` instead of `micromatch` (since alpha.13)
  - This may have changed semenatics in edge-cases
- `checkContext` was removed from `IgnorePlugin` (since alpha.16)
- New `__webpack_exports_info__` API allows export usage introspection (since alpha.21)
- SourceMapDevToolPlugin applies to non-chunk assets too now (since alpha.27)

# Other Minor Changes

- removed buildin directory and replaced buildins with runtime modules
- Removed deprecated features
  - BannerPlugin now only support an options object
- removed CachePlugin
- Chunk.entryModule is deprecated
- Chunk.hasEntryModule is deprecated
- Chunk.addModule is deprecated
- Chunk.removeModule is deprecated
- Chunk.getNumberOfModules is deprecated
- Chunk.modulesIterable is deprecated
- Chunk.compareTo is deprecated
- Chunk.containsModule is deprecated
- Chunk.getModules is deprecated
- Chunk.remove is deprecated
- Chunk.moveModule is deprecated
- Chunk.integrate is deprecated
- Chunk.canBeIntegrated is deprecated
- Chunk.isEmpty is deprecated
- Chunk.modulesSize is deprecated
- Chunk.size is deprecated
- Chunk.integratedSize is deprecated
- Chunk.getChunkModuleMaps is deprecated
- Chunk.hasModuleInGraph is deprecated
- Chunk.updateHash signature changed
- Chunk.getChildIdsByOrders signature changed (TODO: consider moving to ChunkGraph)
- Chunk.getChildIdsByOrdersMap signature changed (TODO: consider moving to ChunkGraph)
- Chunk.getChunkModuleMaps removed
- Chunk.setModules removed
- deprecated Chunk methods removed
- ChunkGraph added
- ChunkGroup.setParents removed
- ChunkGroup.containsModule removed
- ChunkGroup.remove no longer disconnected the group from block
- ChunkGroup.compareTo signature changed
- ChunkGroup.getChildrenByOrders signature changed
- ChunkGroup index and index renamed to pre/post order index
  - old getter is deprecated
- ChunkTemplate.hooks.modules sigature changed
- ChunkTemplate.hooks.render sigature changed
- ChunkTemplate.updateHashForChunk sigature changed
- Compilation.hooks.optimizeChunkOrder removed
- Compilation.hooks.optimizeModuleOrder removed
- Compilation.hooks.advancedOptimizeModuleOrder removed
- Compilation.hooks.optimizeDependenciesBasic removed
- Compilation.hooks.optimizeDependenciesAdvanced removed
- Compilation.hooks.optimizeModulesBasic removed
- Compilation.hooks.optimizeModulesAdvanced removed
- Compilation.hooks.optimizeChunksBasic removed
- Compilation.hooks.optimizeChunksAdvanced removed
- Compilation.hooks.optimizeChunkModulesBasic removed
- Compilation.hooks.optimizeChunkModulesAdvanced removed
- Compilation.hooks.optimizeExtractedChunksBasic removed
- Compilation.hooks.optimizeExtractedChunks removed
- Compilation.hooks.optimizeExtractedChunksAdvanced removed
- Compilation.hooks.afterOptimizeExtractedChunks removed
- Compilation.hooks.stillValidModule added
- Compilation.fileDependencies, Compilation.contextDependencies and Compilation.missingDependencies are now LazySets (since alpha.20)
- Compilation.entries removed
  - MIGRATION: Use `Compilation.entryDependencies` instead
- Compilation.\_preparedEntrypoints removed
- dependencyTemplates is now a `DependencyTemplates` class instead of a raw `Map`
- Compilation.fileTimestamps and contextTimestamps removed
  - MIGRATION: Use `Compilation.fileSystemInfo` instead
- Compilation.waitForBuildingFinished removed
  - MIGRATION: Use the new queues
- Compilation.addModuleDependencies removed
- Compilation.prefetch removed
- Compilation.hooks.beforeHash is now called after the hashes of modules are created
  - MIGRATION: Use `Compiliation.hooks.beforeModuleHash` instead
- Compilation.applyModuleIds removed
- Compilation.applyChunkIds removed
- Compiler.root added, which points to the root compiler
  - it can be used to cache data in WeakMaps instead of statically scoped
- Compiler.hooks.afterDone added
- Source.emitted is no longer set by the Compiler
  - MIGRATION: Check `Compilation.emittedAssets` instead
- Compiler/Compilation.compilerPath added: It's a unique name of the compiler in the compiler tree. (Unique to the root compiler scope)
- Module.needRebuild deprecated
  - MIGRATION: use `Module.needBuild` instead
- Dependency.getReference signature changed
- Dependency.getExports signature changed
- Dependency.getWarnings signature changed
- Dependency.getErrors signature changed
- Dependency.updateHash signature changed
- Dependency.module removed
- There is now a base class for DependencyTemplate
- MultiEntryDependency removed
- EntryDependency added
- EntryModuleNotFoundError removed
- SingleEntryPlugin removed
- EntryPlugin added
- Generator.getTypes added
- Generator.getSize added
- Generator.generate signature changed
- HotModuleReplacementPlugin.getParserHooks added
- Parser was moved to JavascriptParser
- ParserHelpers was moved to JavascriptParserHelpers
- MainTemplate.hooks.moduleObj removed
- MainTemplate.hooks.currentHash removed
- MainTemplate.hooks.addModule removed
- MainTemplate.hooks.requireEnsure removed
- MainTemplate.hooks.globalHashPaths removed
- MainTemplate.hooks.globalHash removed
- MainTemplate.hooks.hotBootstrap removed
- MainTemplate.hooks some signatures changed
- Module.hash deprecated
- Module.renderedHash deprecated
- Module.reasons removed
- Module.id deprecated
- Module.index deprecated
- Module.index2 deprecated
- Module.depth deprecated
- Module.issuer deprecated
- Module.profile removed
- Module.prefetched removed
- Module.built removed
- Module.used removed
  - MIGRATION: Use `Module.getUsedExports` instead
- Module.usedExports deprecated
  - MIGRATION: Use `Module.getUsedExports` instead
- Module.optimizationBailout deprecated
- Module.exportsArgument removed
- Module.optional deprecated
- Module.disconnect removed
- Module.unseal removed
- Module.setChunks removed
- Module.addChunk deprecated
- Module.removeChunk deprecated
- Module.isInChunk deprecated
- Module.isEntryModule deprecated
- Module.getChunks deprecated
- Module.getNumberOfChunks deprecated
- Module.chunksIterable deprecated
- Module.hasEqualsChunks removed
- Module.useSourceMap moved to NormalModule
- Module.addReason removed
- Module.removeReason removed
- Module.rewriteChunkInReasons removed
- Module.isUsed removed
  - MIGRATION: Use `isModuleUsed`, `isExportUsed` and `getUsedName` instead
- Module.updateHash signature changed
- Module.sortItems removed
- Module.unbuild removed
  - MIGRATION: Use `invalidateBuild` instead
- Module.getSourceTypes added
- Module.getRuntimeRequirements added
- Module.size signature changed
- ModuleFilenameHelpers.createFilename signature changed
- ModuleProfile class added with more data
- ModuleReason removed
- ModuleTemplate.hooks signatures changed
- ModuleTemplate.render signature changed
- Compiler.dependencies removed
  - MIGRATION: Use `MultiCompiler.setDependencies` instead
- MultiModule removed
- MultiModuleFactory removed
- NormalModuleFactory.fileDependencies, NormalModuleFactory.contextDependencies and NormalModuleFactory.missingDependencies are now LazySets (since alpha.20)
- RuntimeTemplate methods now take `runtimeRequirements` arguments
- serve property is removed
- Stats.jsonToString removed
- Stats.filterWarnings removed
- Stats.getChildOptions removed
- Stats helper methods removed
- Stats.toJson signature changed (second argument removed)
- ExternalModule.external removed
- HarmonyInitDependency removed
- Dependency.getInitFragments deprecated
  - MIGRATION: Use `apply` `initFragements` instead
- DependencyReference now takes a function to a module instead of a Module
- HarmonyImportSpecifierDependency.redirectedId removed
  - MIGRATION: Use `setId` instead
- acorn 5 -> 7 (since alpha.21)
- Testing
  - HotTestCases now runs for multiple targets `async-node` `node` `web` `webworker`
  - TestCases now also runs for filesystem caching with `store: "instant"` and `store: "pack"`
  - TestCases now also runs for deterministic module ids
- Tooling added to order the imports (checked in CI)
- Chunk name mapping in runtime no longer contains entries when chunk name equals chunk id
- add `resolvedModuleId` `resolvedModuleIdentifier` and `resolvedModule` to reasons in Stats which point to the module before optimizations like scope hoisting (since alpha.6)
- show `resolvedModule` in Stats toString output (since alpha.6)
- loader-runner was upgraded: https://github.com/webpack/loader-runner/releases/tag/v3.0.0 (since alpha.6)
- `file/context/missingDependencies` in `Compilation` are no longer sorted for performance reasons (since alpha.8)
  - Do not rely on the order
- webpack-sources was upgraded: https://github.com/webpack/webpack-sources/releases/tag/v2.0.0-beta.0 (since alpha.8)
- webpack-command support was removed (since alpha.12)
- Use schema-utils@2 for schema validation (since alpha.20)
- `Compiler.assetEmitted` has a improved second argument with more information (since alpha.27)
- BannerPlugin omits trailing whitespace (since beta.1)
- removed `minChunkSize` option from `LimitChunkCountPlugin` (since beta.1)
- reorganize from javascript related files into sub-directory (since beta.1)
  - `webpack.JavascriptModulesPlugin` -> `webpack.javascript.JavascriptModulesPlugin`
- Logger.getChildLogger added (since beta.3)
