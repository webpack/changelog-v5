[Help by editing this file](https://github.com/webpack/changelog-v5/edit/master/README.md).

# General direction

This release focus on the following:

- We try to improve build performance with Persistent Caching.
- We try to improve Long Term Caching with better algorithms and defaults.
- We try to cleanup internal structures that were left in a weird state, while implementing features in v4 without introducing any breaking changes.
- We try to prepare for future features by introducing breaking changes now, allowing us to stay on v5 for as long as possible.

# Major Changes

## Removed Deprecated Items

All items deprecated in v4 were removed.

MIGRATION: Make sure that your webpack 4 build does not print deprecation warnings.

Here are a few things that were removed but did not have deprecation warnings in v4:

- IgnorePlugin and BannerPlugin must now be passed an options object.

## Automatic Node.js Polyfills Removed

In the early days, webpack's aim was to allow running most node.js modules in the browser, but the module landscape changed and many module uses are now written mainly for frontend purposes. webpack <= 4 ships with polyfills for many of the node.js core modules, which are automatically applied once a module uses any of the core modules (i.e. the `crypto` module).

While this makes using modules written for node.js easy, it adds these huge polyfills to the bundle. In many cases these polyfills are unnecessary.

webpack 5 stops automatically polyfilling these core modules and focuses on frontend-compatible modules.

MIGRATION:

- Try to use frontend-compatible modules whenever possible.
- It's possible to manually add a polyfill for a node.js core module. An error message will give a hint on how to achieve that.
- Package authors: Use the `browser` field in `package.json` to make a package frontend-compatible. Provide alternative implementations/dependencies for the browser.

FEEDBACK: Please provide us with feedback whether you like or dislike the above mentioned change. We are uncertain whether this will make it into the final release or not.

## Deterministic Chunk and Module IDs

New algorithms were added for long term caching. These are enabled by default in production mode.

`chunkIds: "deterministic", moduleIds: "deterministic"`

The algorithms assign short (3 or 4 charachter) numeric IDs to modules and chunks in a deterministic way.
This is a trade-off between bundle size and long term caching.

MIGRATION: Best use the default values for `chunkIds` and `moduleIds`. You can also opt-in to the old defaults `chunkIds: "size", moduleIds: "size"`, this will generate smaller bundles, but invalidate them more often for caching.

## Named Chunk IDs

A new named chunk id algorithm enabled by default in development mode, gives chunks (and filenames) human-readable names.
A Module ID is detemined by its path, relative to the `context`.
A Chunk ID is determined by the chunk's content.

So you no longer need to use `import(/* webpackChunkName: "name" */ "module")` for debugging.
But it would still make sense, if you want to control the filenames for production environments.

It's possible to use `chunkIds: "named"` in production, but make sure not to accidentically expose sensitive information about module names.

MIGRATION: If you dislike the filenames being changed in development, you can pass `chunkIds: "natural"` in order to use the old numberic mode.

## Compiler Idle and Close

Compilers now need to be closed after being used. Compilers now enter and leave idle state and have hooks for these states. Plugins may use these hooks to do unimportant work. (i. e. the Persistent cache slowy stores the cache to disk). On compiler close - All remaining work should be finished as fast as possible. A callback signals the closing as done.

Plugins and their respective authors should expect that some users may forget to close the Compiler. So, all work should eventually be finishing while in idle too. Processes should be prevented from exiting when the work is being done.

The `webpack()` facade automatically calls `close` when being passed a callback.

MIGRATION: While using the node.js API, make sure to call `Compiler.close` when done.

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

There is now an experimental filesystem cache. It's opt-in and can be enabled with `cache: { type: "filesystem" }` in the configuration.
Currently only the core feature set is ready, and when using it one should be aware of the current limitations to avoid unexpected bugs.
**Please do not use this feature if you don't understand the limitations.**

There is an automatic cache invalidation for resolving module source code and filesystem structure.
There is **no** automatic cache invalidations for configurations and loader/plugin/core changes!
There is an option for manual cache invalidation for configuration or build code changes (`cache.version`).

Do not worry, we plan on adding this, but it's not ready yet.

So, here is a guide to make sure you are fine:

Update the `cache.version` when:

- you upgrade your tooling dependencies (i. e. webpack, loader, plugin)
- you change your configuration

HINT: If you want to automate this, it could be a good idea to hash `webpack.config.js` and `node_modules/.yarn-integrity` and pass them to `cache.version`. That's probably how we will do it internally.

## SplitChunks for single-file-targets

Targets that only allow to startup a single file (like node, WebWorker, electron main) now support loading the dependent pieces required for bootstrap automatically by the runtime.

This allows to use `splitChunks` for these targets with `chunks: "all"`.

(since alpha.3)

## Minimum Node.js Version

The minimum supported node.js version has increased from 6 to 8.

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
  - `cache.loglevel`
  - `cache.hashAlgorithm`
- `resolve.cache` added: Allows to disable/enable the safe resolve cache
- `resolve.concord` removed
- Automatic polyfills for native node.js modules were removed
  - `node.Buffer` removed
  - `node.console` removed
  - `node.process` removed
  - `node.*` (node.js native module) removed
  - MIGRATION: `resolve.alias` and `ProvidePlugin`. Errors will give hints.
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
- `optimization.splitChunks` sizes can now be objects with a size per source type
  - `minSize`
  - `maxSize`
  - `maxAsyncSize`
  - `maxInitialSize`
- `optimization.splitChunks` `maxAsyncSize` and `maxInitialSize` added next to `maxSize`: allows to specify different max sizes for initial and async chunks
- `optimization.splitChunks` `name: true` removed: Automatic names are no longer supported
  - MIGRATION: Use the default. `chunkIds: "named"` will give your files useful names for debugging
- `optimization.splitChunks.cacheGroups[].idHint` added: Gives a hint how the named chunk id should be chosen
- `optimization.splitChunks` `automaticNamePrefix` removed
  - MIGRATION: Use `idHint` instead
- `output.devtoolLineToLine` removed
  - MIGRATION: No replacement
- `output.hotUpdateChunkFilename: Function` is now forbidden: It never worked anyway.
- `output.hotUpdateMainFilename: Function` is now forbidden: It never worked anyway.
- `stats.chunkRootModules` added: Show root modules for chunks
- `stats.orphanModules` added: Show modules which are not emitted
- `stats.runtime` added: Show runtime modules
- `stats.chunkRelations` added: Show parent/children/sibling chunks (since alpha.1)
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

## Changes to the Defaults

- `module.unsafeCache` is now only enabled for `node_modules` by default
- `optimization.moduleIds` defaults to `deterministic` in production mode, instead of `size`
- `optimization.chunkIds` defaults to `deterministic` in production mode, instead of `total-size`
- `optimization.nodeEnv` defaults to `false` in `none` mode
- `resolve(Loader).cache` defaults to `true` when `cache` is used
- `resolve(Loader).cacheWithContext` defaults to `false`
- ~`node.global` defaults to `false`~ (since alpha.4 removed)

# Major Internal Changes

The following changes are only relavant for plugin authors:

## Runtime Modules

A large part of the runtime code was moved into the so called "runtime modules". These special modules are in-charge of adding runtime code. They can be added in to any chunk, but are currently always added to the runtime chunk. "Runtime Requirements" control which runtime modules are added to the bundle. This ensures that only runtime code that is used is added to the bundle. In the future, runtime modules could also added to an on-demand-loaded chunk, to load runtime code when needed.

The core runtime is now very small and only includes the `__webpack_require__` function, module factories and the module instance cache. In the future, an alternative code runtime can be used to avoid wrapping the bundle in a IIFE and allow ESM style exports. This will allow ESM as target.

As some responsibilities from Main and ChunkTemplate were removed, several associated hooks were removed as well.

MIGRATION: If you are injecting runtime code into the webpack runtime in a plugin, consider using RuntimeModules instead.

## Serialization

A serialization mechanism was added to allow serialization of complex objects in webpack. It has an opt-in semantic, so classes that should be serialized need to be explicitly flagged (and their serialization implemented). This has been done for most Modules, all Dependencies and some Errors.

MIGRATION: When using custom Modules or Dependencies, it is recommended to make them serializable to benefit from persistent caching.

## Extensible Caching

A `Cache` class with a plugin interface has been added. This class can be used to write and read to the cache. Depending on configuration, different plugins can add the functionality to the cache. The `MemoryCachePlugin` adds in-memory caching. The `FileCachePlugin` adds persistent (file-system) caching.

The `FileCachePlugin` uses the serialization mechanism to persist and restore cached items to/from the disk.

## Hook Object Frozen

Classes with `hooks` have their `hooks` object frozen, so adding custom hooks is no longer possible this way.

MIGRATION: The recommended way to add custom hooks is using a WeakMap and a static `getXXXHooks(XXX)` (i. e. `getCompilationHook(compilation)`) method. Internal classes use the same mechanism used for custom hooks.

## Tapable Base Class Removed

The compat layer for webpack 3 plugins has been removed. It had already been deprecated for webpack 4.

MIGRATION: Use the new tapable API.

## Staged Hooks

For several steps in the sealing process, there had been multiple hooks for different stages. i. e. `optimizeDependenciesBasic` `optimizeDependencies` and `optimizeDependenciesAdvanced`. These have been removed in favor of a single hook which can be used with a `stage` option. See `OptimizationStages` for possible stage values.

MIGRATION: Hook into the remaining hook instead. You may add a `stage` option.

## Order and IDs

webpack used to order modules and chunks in the Compilation phase, in a specific way, to assign IDs in an incremental order. This is no longer the case. The order will no longer be used for id generation, instead, the full control of ID generation is in the plugin.

Hooks to optimize the order of module and chunks have been removed.

MIGRATION: You cannot rely on the order of modules and chunks in the compilation phase no more.

## Arrays to Sets

- Compilation.modules is now a Set
- Compilation.chunks is now a Set

There is a compat-layer which prints deprecation warnings.

MIGRATION: Use Set methods instead of Array methods.

## Compilation.fileSystemInfo

This new class can be used to access information about the filesystem in a cached way. Currently it allows to ask for both file and directory timestamps. Information about timestamps is transferred from the watcher if possible, otherwise determined by filesystem access.

In the future, asking for file content hashes will be added and modules will be able to check validity with file contents instead of file hashes.

MIGRATION: Instead of using `file/contextTimestamps` use the `compilation.fileSystemInfo` API instead.

## Hot Module Replacement

HMR runtime has be refactored to Runtime Modules. `HotUpdateChunkTemplate` has been merged into `ChunkTemplate`. ChunkTemplates and plugins should also handle `HotUpdateChunk`s now.

The javascript part of HMR runtime has been separated from the core HMR runtime. Other module types can now also handle HMR in their own way. In the future, this will allow i. e. HMR for the mini-css-extract-plugin or for WASM modules.

MIGRATION: As this is a newly intorduced functionality, there is nothing to migrate.

## Work Queues

webpack used to handle module processing by functions calling functions, and a `semaphore` which limits parallelism. The `Compilation.semaphore` has been removed and async queues now handle work queuing and processing. Each step has a separate queue:

- `Compilation.factorizeQueue`: calling the module factory for a group of dependencies.
- `Compilation.addModuleQueue`: adding the module to the compilation queue (may restore module from cache).
- `Compilation.buildQueue`: building the module if neccessary (may stores module to cache).
- `Compilation.rebuildQueue`: building a module again if manually triggered.
- `Compilation.processDependenciesQueue`: processing dependencies of a module.

These queues have some hooks to watch and intercept job processing.

In the future, multiple compilers may work together and job orchestration can be done by intercepting these queues.

MIGRATION: As this is a newly intorduced functionality, there is nothing to migrate.

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

# Minor Changes

- Compiler.name: When generating a compiler name with absolute paths, make sure to separate them with `|` or `!` on both parts of the name.
  - Using space as a separator is now deprecated. (Paths could contain spaces)
  - Hint: `|` is replaced with space in Stats string output.
- top-level return is now allowed in non-ESM modules.
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
- ProgressPlugin `entries` option now defaults to `on`
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
- `stats.assetsByChunkName[x]` is now always an array (since alpha.5)


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
- Module.usedExports deprecated
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
- RuntimeTemplate methods now take `runtimeRequirements` arguments
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
- acorn 5 -> 6
- Testing
  - HotTestCases now runs for multiple targets `async-node` `node` `web` `webworker`
  - TestCases now also runs for filesystem caching with `store: "instant"` and `store: "pack"`
  - TestCases now also runs for deterministic module ids
- Tooling added to order the imports (checked in CI)
- Chunk name mapping in runtime no longer contains entries when chunk name equals chunk id
- add `resolvedModuleId` `resolvedModuleIdentifier` and `resolvedModule` to reasons in Stats which point to the module before optimizations like scope hoisting (since alpha.6)
- show `resolvedModule` in Stats toString output (since alpha.6)
- loader-runner was upgraded: https://github.com/webpack/loader-runner/releases/tag/v3.0.0 (since alpha.6)
