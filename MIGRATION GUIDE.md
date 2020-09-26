[Help by editing this file](https://github.com/webpack/changelog-v5/edit/master/MIGRATION%20GUIDE.md).

This guide should help you migrating to webpack 5 when **using webpack directly**. If you are using some higher-level tool to run webpack, please refer to the guide on migrating to webpack 5 for the tool you are using.

# Preparations

## Upgrade your Node.js version

webpack requires at least 10.13.0, but using Node.js 12 or 14 is recommended.

Newer Node.js version will improve build performance a lot.

## Upgrade to latest webpack 4 version

When using webpack < 4 consult the webpack 4 migration guide.

When using webpack >= 4, this should be possible without problems.

## Upgrade to latest webpack-cli version (when used)

## Upgrade all plugins and loaders to the latest version

Some plugins have a beta version that needs to be used for webpack 5 compatibility. Use them.

Check migration guide when upgrading plugins across the major version.

## Make sure your build has no errors or warnings

There might be new errors or warnings because of the upgraded versions of webpack, cli, plugins and loaders.

## Check for deprecation warnings during build

You can invoke webpack this way

``` sh
node --trace-deprecation node_modules/webpack/bin/webpack.js ...
```

to get stack traces for deprecation warnings to figure out which plugins are responsible.

webpack 5 removes all deprecated features. You must have no webpack deprecation warning to be able to upgrade.

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

Try to set the following options in your webpack 4 builds and check if it still works correctly.

* `node.Buffer: false`
* `node.process: false`

Note: webpack 5 removes these options and will always use `false`.
You have to remove these options again when upgrading to webpack 5.

# Upgrade webpack version

Run `yarn add webpack@next -D` resp. `npm install webpack@next --dev`.

# Cleanup configuration

* Consider removing `optimization.moduleIds` and `optimization.chunkIds`. The defaults are probably perfect. They support Long Term Caching (in production) and debugging (in development)
* Reconsider `optimization.splitChunks`.
  * It's recommended to use either the defaults or `optimization.splitChunks: { chunks: "all" }`
  * When really using a custom configuration replace `name` with `idHint`.
  * When used to disable the defaults with `default: false, vendors: false`: Consider not doing this, but if you really want to `default: false, defaultVendors: false`.
* When using the `WatchIgnorePlugin`, use `watchOptions.ignore` instead.
* When using `[hash]`: Consider changing to `[contenthash]` (not the same, but better)
* Using WASM: Set `experiments.syncWebAssembly: true` (after migration to webpack 5, migrate to `experiments: { asyncWebAssembly: true, importAsync: true }` instead)
* Using `entry: "./src/index.js`: you can omit it, that's the default
* Using `output.path: path.resolve(__dirname, "dist")`: you can omit it, that's the default
* Using `output.filename: "[name].js"`: you can omit it, that's the default
* Using Yarn PnP and the pnp-resolver-plugin: you must omit it, that's supported by default now.
* Using `IgnorePlugin` with a regular expression as argument: It takes an options object now: `new IgnorePlugin({ resourceRegExp: /regExp/ })`.
* Need to support an older browser? Add version to `target` options
  * By default, Webpack's runtime code uses ES2015 syntax to build smaller bundles.
  * If your build targets environments that don't support this syntax (like IE11), you'll need to add `"es5"` to the `target` option, e. g. `target: ["web", "es5"]`.
  * For Node.js builds include the supported node.js in the `target` option and webpack automatically figure out which syntax is supported, e. g. `target: "node8.6"`.

# Cleanup code

* Using `/* webpackChunkName: "..." */`: Make sure to understand the intention.
  * The name here is intended to be public visible.
  * It's not a development-only name
  * webpack will use it to name files in production and development mode
  * webpack 5 will automatically assign useful file names in development even when not using `webpackChunkName`
* Using named exports from JSON modules: This is not supported by the (new) spec and you will get a warning
  * Instead of `import { version } from "./package.json"; console.log(version);` use `import package from "./package.json"; console.log(package.version);`

# Cleanup build code

* Using `const compiler = webpack(...);`: Make sure to close the compiler after use
  * `compiler.close(callback)`
  * This doesn't apply to the `webpack(..., callback)` form.
  * This is optional if you use webpack in watching mode until the user ends the process.

# Run a single build and follow advises

Please make sure to read errors/warnings carefully.

There is no advise? Please create an issue and we will add one.

Repeat this step until you solved at least level 3. Best continue to solve level 4:

## Level 1: Schema validation fails

Some configuration options have changed. There should be a validation error with a `BREAKING CHANGE:` note, or a hint which option should be used instead.

## Level 2: webpack crashes with error

The error message should tell you what needs to be changed.

## Level 3: Build Errors

The error message should have a `BREAKING CHANGE:` note.

## Level 4: Build Warnings

The warning message should tell you what can be improved.

There might be some cases where you don't have direct control over the code, like when using packages from NPM.
There the new `ignoreWarnings` configuration option might become handy to temporary hide the warnings, e. g.

``` js
ignoreWarnings: [
  { module: /node_modules/, message: /only default export is available soon/ }
]
```

## Level 5: Runtime errors

This is tricky. You probably have to debug to find the problem. A general advise is difficult here.

Here are some common things:

* `process` is not defined.
  * Webpack 5 do no longer include a polyfill for this Node.js variable. Avoid using it in frontend code.
  * Want to support frontend and browser usage? Use the `exports` or `imports` package.json field to use different code depending on environment.
    * To support older bundlers use the `browser` field too.
    * Alternative: Wrap code blocks with `typeof process` checks. Note that this is worse regarding bundle size.
  * Want to use environment variables with `process.env.VARIABLE`? You need to use the `DefinePlugin` or `EnvironmentPlugin` to define these variables in the configuration.
    * Consider using `VARIABLE` instead and make sure to check `typeof VARIABLE !== "undefined"` too. `process.env` is Node.js specific and should be avoided in frontend code.

## Level 6: Deprecation warnings

You probably get a lot of deprecation warnings.
This is not directly a problem.
Plugins need time to catch up with core changes.
Please report these deprecations to the plugins.
These deprecations are only warnings and the build will still work with only minor drawbacks (like less performance).

You can hide deprecation warnings with

``` sh
node --no-deprecation node_modules/webpack/bin/webpack.js ...
```

but this should only be a temporary workaround.

Contributors to plugins can follow the advises in the deprecation messages.

## Level 7: Performance issues

Usually performance should improve with webpack 5, but there are also a few cases where performance get worse.

* Profile where the time is spend.
  * `--profile --progress` displays a very simple performance profile now
  * `node --inspect-brk node_modules/webpack/bin/webpack.js` + [`chrome://inspect`](chrome://inspect) / [`edge://inspect`](edge://inspect) (see profiler tab).
    * You can save these profiles to files and provide them in issues.
    * Add `--no-turbo-inlining` for better stack traces in some cases
* Time for building modules in incremental builds can be improved by reverting to unsafe caching like in webpack 4:
  * `module.unsafeCache: true`
  * But this might affect the ability to handle some of the changes to the code base
* Full build
  * Backward-compat layer for deprecated things usually have worse performance compared to the new thing.
  * Creating many warnings can affect build performance, even if they are ignored.
  * Source Maps are expensive. Check `devtool` option in documentation to see a comparison of the different options.
  * Anti-Virus-Protection might affect performance of file system access.
  * Persistent Caching can help to improve repeated full builds.
  * Module Federation allows to split the application into multiple smaller builds.

# Everything working?

Tweet that you successfully migrated to webpack 5.

# Not working?

Create an issue and tell us about your problems.

# Something missing in this guide?

Please send a PR to help the next person using this guide:

[Click here to edit this file](https://github.com/webpack/changelog-v5/edit/master/MIGRATION%20GUIDE.md).
