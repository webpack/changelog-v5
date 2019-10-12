[Help by editing this file](https://github.com/webpack/changelog-v5/edit/master/MIGRATION%20GUIDE.md).

This guide should help you migrating to webpack 5 when **using webpack directly**. If you are using some higher-level tool to run webpack, please see guide in this tool for migration.

# Preparations

## Upgrade your node.js version

webpack requires at least 8.9.0, but using node.js 10 or 12 is recommended.

Newer node.js version will improve build performance a lot.

## Upgrade to latest webpack 4 version

When using webpack < 4 consult the webpack 4 migration guide.

When using webpack >= 4, this should be possible without problems.

## Upgrade to latest webpack-cli version (when used)

## Upgrade all plugins and loaders to latest version

Some plugins have beta version that need to be used for webpack 5 compatibility. Use them.

Check migration guide when upgrading plugins across major version.

## Make sure your build has no errors or warnings

There might be new errors or warnings because of the upgraded versions of webpack, cli, plugins and loaders.

## Check for deprecation warnings during build

You can invoke webpack this way

``` sh
node --trace-deprecation node_modules/webpack/bin/webpack.js ...
```

to get stack traces for deprecation warnings to figure out which plugins are responsible.

webpack 5 removes all deprecated features. You must have no webpack deprecation warning.

## Make sure you are using entrypoint information from stats

When using html-webpack-plugin you are fine.

When using static html or creating HTML in some other way, make sure you use `entrypoints` from stats json to generate `<script>`, `<style>` and `<link>` tags.

If this is not possible avoid setting `splitChunks.chunks: "all"` and `splitChunks.maxSize` later in this guide. Note that this is sub-optimal.

## Make sure to use `mode`

Set `mode` correctly to either `mode: "production"` or `mode: "development"`.

This will make sure you get the correct defaults.

## Update outdated options

Update the following options to their new version:

* `optimization.hashedModuleIds: true` -> `optimization.moduleIds: "hashed"`
* `optimization.namedChunks: true` -> `optimization.chunkIds: "named"`
* `optimization.namedModules: true` -> `optimization.moduleIds: "named"`
* `NamedModulesPlugin` -> `optimization.moduleIds: "named"`
* `NamedChunksPlugin` -> `optimization.chunkIds: "named"`
* `HashedModulesPlugin` -> `optimization.moduleIds: "hashed"`
* `optimization.occurrenceOrder: true` -> `optimization: { chunkIds: "total-size", moduleIds: "size" }`

# Test webpack 5 compatibility

Try to set the following options in your webpack 4 build and check if it still works correctly.

* `node.Buffer: false`
* `node.process: false`

# Upgrade webpack version

Run `yarn add webpack@next -D` resp. `npm install webpack@next --dev`.

# Cleanup configuration

* Remove `optimization.moduleIds` and `optimization.chunkIds`. The defaults are probably perfect. They support Long Term Caching (in production) and debugging (in development)
* Reconsider `optimization.splitChunks`.
  * It's recommended to use either the defaults or `optimization.splitChunks: { chunks: "all" }`
  * When using HTTP/2 and Long Term Caching set `optimization.splitChunks: { chunks: "all", maxInitialRequests: 30, maxAsyncRequests: 30, maxSize: 100_000 }`
  * When really using a custom configuration replace `name` with `idHint`.
* When using the `WatchIgnorePlugin`, use `watchOptions.ignore` instead.
* When using `[hash]`: Consider changing to `[contenthash]` (not the same, but better)
* Using WASM: Set `experiments.syncWebAssembly: true` (after migration to webpack 5, migrate to `experiments: { asyncWebAssembly: true, importAsync: true }` instead)
* Using .mjs: Set `experiments.mjs: true`
* Using `entry: "./src/index.js`: you can omit it, that's the default
* Using `output.path: path.resolve(__dirname, "dist")`: you can omit it, that's the default
* Using `output.filename: "[name].js"`: you can omit it, that's the default
* Using Yarn PnP and the pnp-resolver-plugin: you must omit it, that's supported by default now.

# Cleanup code

* Using `/* webpackChunkName: "..." */`: Make sure to understand the intention.
  * The name here is intended to be public visible.
  * It's not a development-only name
  * webpack will use it to name files in production and development mode
  * webpack 5 will automatically assign useful file names in development even when not using `webpackChunkName`
* Using named exports from JSON modules: This is not supported by the (new) spec and you will get a warning
  * Instead of `import { version } from "./package.json"` use `import package from "./package.json"; const { version } = package;`

# Cleanup build code

* Using `const compiler = webpack(...);`: Make sure to close the compiler after use
  * `compiler.close()`

# Run a single build and follow advises

There is no advise? Please create an issue and we will add one.

Repeat this step until you solved at least level 3. Best continue to solve level 4:

## Level 1: Schema validation fails

Some configuration options have changed. There should be a validation error with a `BREAKING CHANGE:` note.

## Level 2: webpack crashes with error

The error message should tell you what need to be changed.

## Level 3: Build Errors

The error message should have a `BREAKING CHANGE:` note.

## Level 4: Build Warnings

The warning message should tell you what can be improved.

## Deprecation warnings

You probably get a lot of deprecation warnings. This is not a problem right now. Plugins need time to catch up with core changes. Please do not care about them until release candidate.

You can hide deprecation warnings with

``` sh
node --no-deprecation node_modules/webpack/bin/webpack.js ...
```

but this should only be a temporary workaround.

Contributors to plugins can follow the advises in the deprecation messages.

# Everything working?

Tweet that you successfully migrated to webpack 5.

# Not working?

Create an issue and tell us about your problems.

# Something missing in this guide?

Please send a PR to help the next person using this guide:

[Click here to edit this file](https://github.com/webpack/changelog-v5/edit/master/MIGRATION%20GUIDE.md).
