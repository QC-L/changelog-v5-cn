[原文链接](README-EN.md)

# 简要说明

此版本重点关注以下内容：

- 我们尝试通过持久化存储优化构建性能。
- 我们尝试采用更好的算法与 defalut 来改善长效缓存。
- 我们尝试通过更好的 Tree Shaking 和代码生成来改善 bundle 的大小。
- 我们尝试清除内部结构中奇怪的代码，同时在不影响 v4 功能基础上实现了新特性。
- 我们目前尝试通过引入破坏性更改来为新特性做准备，以便于我们能尽可能长期地使用 v5。

# 迁移指南

=> [查阅迁移指南](https://github.com/webpack/changelog-v5/blob/master/MIGRATION%20GUIDE.md) <=

# 主要更改

## 移除废弃的代码

v4 中所有废弃的代码均已删除。

**迁移**：以确保你的 webapck 4 不打印弃用警告。

以下是已删除但在 v4 中没有弃用警告的内容：

- 现在必须为 IgnorePlugin 和 BannerPlugin 传递一个 options 对象。

## 自动移除 Node.js Polyfills

早期，webpack 的目的是允许在浏览器中运行大多数 node.js 模块，但是模块整体格局发生了变化，现在许多模块的主要用途是以编写前端为目的。webpack <= 4 附带了许多 Node.js 核心模块的 polyfil，一旦模块中使用了任何核心模块（即 ”crypto“ 模块），这些模块就会被自动启用。

虽然这使得为 Node.js 编写模块变得简单，但它会将超大的 polyfill 添加到 package 中。在许多情况下，这些 polyfill 并非必要。

webpack 5 会停止自动 polyfill 这些核心模块，并专注于与前端兼容的模块。

**迁移**:

- 尽可能尝试使用与前端兼容的模块。
- 可以为 Node.js 核心模块手动添加 polyfill。错误信息将提示如何进行此操作。
- package 作者：在 `package.json` 中使用 `browser` 字段，以使得 package 与前端代码兼容。为 borwser 提供可选的 implementations/dependencies。

**反馈**：无论是否喜欢上述修改，请都向我们提出反馈。我们并不确定是否会纳入最终版本。

## 采用新算法生成 chunk ID 以及 module ID

添加了用于长效缓存的新算法。在生产模式下，默认启用这些功能。

`chunkIds: "deterministic", moduleIds: "deterministic"`

此算法采用确定性的方式将短数字 ID（3 或 4 个字符）分配给 modules 和 chunks。
这是基于 bundle 大小和长效缓存间的折中方案。

**迁移**：最好使用 `chunkIds` 和 `moduleIds` 的默认值。你还可以选择使用旧的默认值，`chunkIds: "size", modules: "size"`，这将生成较小的 bundle，但这会使得它们频繁地进行缓存。

## 以新算法混淆 export 名称

添加了新算法来处理 export 的名称。默认情况下启用。

如果可能，它将以确定性方式破坏 export 的名称。

**迁移**：不需要进行任何操作。

## 为 chunk IDs 命名

在开发模式下默认启用，以新的算法为 chunk id 命名，给 chunk（以及文件名）提供易于理解的名称。
module ID 由其相对于 `context` 的路径决定。
chunk ID 由 chunk 的内容决定。

因此，你不再需要使用 `import(/* webpackChunkName: "name" */ "module")` 进行调试。
但是，如果你要控制生产环境的文件名，那仍可使用。

可以在生产中使用 `chunkIds: "named"`，但要确保在使用时不会意外地泄露有关模块名称的敏感信息。

**迁移**：如果你不喜欢在开发中更改文件名，则可以传递 `chunkIds: "natural"` 以使用旧的数字模式。

## JSON 模块

JSON 模块现在符合规范，并会在使用非默认导出时发出警告。

**迁移**：使用 `default export`。

（自 alpha.16 起）

## 嵌套 tree-shaking

webpack 现在可以追踪对 exports 嵌套属性的访问。重新导出 namespace 对象，这可以改善 Tree Shaking 操作（未使用 export elimination 和 export mangling）。

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

在此示例中，可以在生成模式下移除 export `b`。

（从 alpha.15 起）

## 内部模块（inner-module） tree-shaking

webpack 4 没有分析模块 export 与 import 之间的依赖关系。webpack 5 有一个新的选项 `optimization.innerGraph`，该选项在生产模式下默认启用，它对模块中的符号进行分析以找出从 export 到 import 的依赖关系。

如下述模块所示：

``` js
import { something } from "./something";

function usingSomething() {
  return something;
}

export function test() {
  return usingSomething();
}
```

内部图算法将确定仅在使用 export 的 `test` 时使用 `something`。这样可以将更多 export 标记为未使用，并从 bundle 中删除更多的代码。

如果设置了 `"sideEffects": false`，则可以省略更多模块。在此示例中，当未使用 export 的 `test` 时，将忽略 `./something`。

如需获取有关未使用的 export 的信息，需使用 `optimization.unusedExports`。如需删除无副作用的模块，需使用 `optimization.sideEffects`。

此方式可以分析以下符号：
* 函数声明（function declarations）
* class 声明（class declarations）
* 带有 `export default` 或带有变量声明（variable declarations）的
  * 函数表达式（function expressions）
  * class 语句（class expressions）
  * `/*#__PURE__*/` 表达式
  * 局部变量（local variables）
  * imported bindings

反馈：如果您发现此分析中缺少某些内容，请反馈 issues，我们考虑将其添加。

此优化也称为深度作用域分析（Deep Scope Analysis）。

（自 alpha.24 起）

## 编译器空闲并关闭（idle and close）

现在需要再使用编译器（compilers）后将其关闭。编译器具有 enter 和 leave 空闲状态，并具有这些状态的 hook。插件可以使用这些 hook 执行不重要的工作。（即，持久化缓存将延迟存储到磁盘）。在编译器关闭时，所有剩余工作应尽快完成。回调执行时，表明关闭已完成。

插件及其各自的作者应该会期望某些用户可能会忘记关闭编译器。因此，所有工作最终也应该在空闲时完成。当工作完成时，应防止进程退出。

当传递 callback 时，`webpack()` 实例会自动调用 `close`。

**迁移**：使用 node.js API 时，请确保在完成后调用 `Complier.close`。

## 改进代码生成

此版本添加了新的选项 `output.ecmaVersion`。它允许为 webpack 生成的运行时代码指定最大 EcmaScript 版本。

webpack 4 仅能于生成 ES5 的代码。webpack 5 现支持 ES5 或 ES2015 的代码。

默认配置将生成 ES2015 的代码。如果你需要支持旧版浏览器（例如，IE11），则可以将其降为 `output.ecmaVersion: 5`。

设置为 `output.ecmaVersion: 2015` 将使用箭头函数生成较短的代码，以及更多符合规范的代码，使用 const 声明（TDZ）作为 `export default`。

（自 alpha.23 起）

生产模式中的默认压缩（default minimizing）也使用 `ecmaVersion` 选项生成较小的代码。（自 alpha.31 起）

## chunk 分割以及 module size

与之前展示单个数值相比，模块现在以更好的方式展示其 size。除此之外，现在也拥有了不同类型的 size。

目前，SplitChunksPlugin 已知道如何处理这些不同的 size，并将它们应用于 `minSize` 和 `maxSize`。
默认情况下，仅处理 javascript 的 size，但你可以传递多个参数来管理它们：

```js
minSize: {
	javascript: 30000,
	style: 50000,
}
```

**迁移**：检查构建中使用了哪些类型的 size，并在 `splitChunks.minSize` 和可选的 `splitChunks.maxSize` 中进行配置。

## 持久化缓存

目前包含文件系统缓存。它是可选的，可以通过以下配置启用：

``` js
cache: {
  // 1. 设置缓存类型为 filesystem
  type: "filesystem",
  
  buildDependencies: {
    // 2. 将你的配置添加为 buildDependency 以在更改配置时，使得缓存失效。
    config: [__filename]
  
    // 3. 如果你还有其他需要构建的内容，可以在此处添加它们
    // 请注意，loader 和所有模块中配置中引用的内容会自动添加
  }
}
```

**重要内容**：

默认情况下，webpack 会假定其所处的 `node_modules` 目录**仅**由包管理器修改。针对 node_modules 目录，将跳过哈希和时间戳处理。出于性能方面考虑，仅使用 package 的名称和版本。symlinks（例如，`npm/yarn link`）很友好。除非你使用 `cache.managedPaths: []` 选项取消此优化，否则请不要直接在 `node_modules` 中编辑文件。

默认情况下，缓存将分别存储在 `node_modules/.cache/webpack` 中（当使用 node_modules 时）和 `.pnp/.cache/webpack`（当使用 Yarn PnP 时，自 alpha.21 起）。你可能永远不必手动删除它。

（自 alpha.20 起）

当使用 Yarn PnP webpack 时，如果 yarn 的缓存不可变（通常不会发生变化）。你可以通过 `cache.immutablePaths: []` 退出此优化。

（自 alpha.21 起）

## 用于 single-file-target 的 chunk 分割

目前，仅允许启动单个文件 target（如 node，WebWorker，electron main）支持在运行时自动加载引导程序所需的相关代码片段。

这允许对带有 `chunks: "all"` 的 target 使用 `splitChunks`。

值得注意的是，由于 chunk 加载是异步的，因此这也会使初始估算也为异步操作。当使用 `output.library` 时，这可能会出现问题，因为导出的值的类型目前为 Promise。从 alpha.14 开始，这将不适用于 `target: "node"`，因为 chunk 加载在此 target 下为同步。

（自 alpha.3 起）

## 更新解析器

`enhanced-resolve` 已更新至 v5。具体改进如下：

- 当使用 Yarn PnP 时，解析器将直接处理无需其他插件
- 此 resolve 可追踪更多的依赖项，例如文件缺失
- 别名（aliasing）可能包含多种选择
- 可以设置别名（aliasing）为 `false`
- 性能提升

（自 alpha.18 起）

## 不包含 JS 的 chunk

不包含 JS 代码的 chunk 将不再生成 JS 文件。

（自 alpha.14 起）

## 实验阶段特性

并非所有特性从开始就文档。在 webpack 4 中，我们添加了实验性功能，并在 changelog 中指出它们是实验性的，但是从配置中并不能很清楚的了解这些功能是实验性的。

在 webpack 5 中，有一个新的 `experiments` 配置项，允许启用实验性功能。这样可以清楚地了解启用/使用了哪些实验特性。

虽然 webpack 遵循语义版本控制，但是实验性功能将成为例外。它可能包含 webpack 次要版本的破坏性更改。发生这种情况时，我们将在 changelog 中添加清晰的注释。这促使我们可以更快地迭代实验性功能，同时还可以使用我们在主要版本上停留更长时间以获得稳定的功能。

以下实验性功能将随 webpack 5 一同发布：

* 像 webpack 4 一样对 `.mjs` 提供支持（`experiments.mjs`）
* 像 webpack 4 一样对旧版 WebAssembly 提供支持（`experiments.syncWebAssembly`）
* 根据[更新规范](https://github.com/WebAssembly/esm-integration) 对新版 WebAssembly 提供支持（`experiments.asyncWebAssembly`）
  * 这使得 WebAssembly 模块成为异步模块
* [Top Level Await](https://github.com/tc39/proposal-top-level-await) Stage 3 阶段提案（`experiments.topLevelAwait`）
  * 在顶层使用 `await` 使模块成为异步模块
* 使用 `import` 引入异步模块（`experiments.importAsync`）
* 使用 `import await` 引入异步模块（`experiments.importAwait`）
* `asset` 模块类似类似于 `file-loader`（`experiments.asset`）（自 alpha.19 起）
* 导出 bundle 作为模块（`experiments.outputModule`）（自 alpha.31 起）
  * 这将从 bundle 中移除 IIFE 的包装器，强制执行严格模式，通过 `<script type="module">` 进行懒加载，并在 `module` 模式下将其进行压缩

请注意，这也意味着针对 `.mjs` 的支持和 WebAssembly 的支持将被**默认禁用**。

（自 alpha.15 起）

## Stats

chunk 间关系默认情况下是隐藏的。可以使用 `stats.chunkRelations` 进行切换。

（自 alpha.1 起）

Stats 现阶段可以区分 `files` 和 `auxiliaryFiles`。

（自 alpha.19 起）

默认情况下，Stats 会隐藏模块和 chunk id。可以使用 `stats.ids` 进行切换。

所有模块的列表均按照到 entrypoint 的距离排序。可以使用 `stats.modulesSort` 进行切换。

chunk 模块列表和 chunk 根模块列表分别根据模块名进行排序。可以分别使用 `stats.chunkModulesSort` 和 `stats.chunkRootModulesSort` 进行更改。

在串联模块中，嵌套模块列表进行拓扑排序。可以通过 `stats.nestedModulesSort` 进行更改。

chunks 和 assets 会显示 chunk id 的提示。

（自 alpha.31 起）

## 最低 Node.js 版本

Node.js 的最低支持版本从 6 变更为 8。

**迁移**：升级到最新的 node.js 可用版本。

# 配置变更

## 结构变更

- 移除 `cache: Object`：不能设置为内存缓存对象
- 添加 `cache.type`：可以设置为 `"memory"` 或 `"filesystem"`
- 为 `cache.type = "filesystem"` 添加新配置项：
  - `cache.cacheDirectory`
  - `cache.name`
  - `cache.version`
  - `cache.store`
  - ~`cache.loglevel`~（自 alpha.20 起被移除）
  - `cache.hashAlgorithm`
  - `cache.idleTimeout`（自 alpha.8 起）
  - `cache.idleTimeoutForIntialStore`（自 alpha.8 起）
  - `cache.managedPaths`（自 alpha.20 起）
  - `cache.immutablePaths`（自 alpha.21 起）
  - `cache.buildDependencies`（自 alpha.20 起）
- 添加 `resolve.cache`：允许禁用/启用安全 resolve 缓存
- 移除 `resolve.concord`
- 移除用于原生 node.js 模块自动的 polyfill
  - 移除 `node.Buffer`
  - 移除 `node.console`
  - 移除 `node.process`
  - 移除 `node.*`（node.js 原生模块）
  - 迁移：使用 `resolve.alias` 和 `ProvidePlugin`。发生错误会给出提示。
- `output.filename` 可以赋值函数（自 alpha.17 起）
- 添加 `output.assetModuleFilename`（自 alpha.19 起）
- `resolve.alias` 的值可以为数组或 `false`（自 alpha.18起）
- 添加 `optimization.chunkIds: "deterministic"`
- 添加 `optimization.moduleIds: "deterministic"`
- 添加 `optimization.moduleIds: "hashed"`
- 添加 `optimization.moduleIds: "total-size"`
- 移除 module 和 chunk id 的相关的弃用选项
  - 移除 `optimization.hashedModuleIds`
  - 移除 `optimization.namedChunks`（`NamedChunksPlugin` 与之相同）
  - 移除 `optimization.namedModules`（`NamedModulesPlugin` 与之相同）
  - 移除 `optimization.occurrenceOrder`
  - 迁移：使用 `chunkIds` 或 `moduleIds`
- `optimization.splitChunks` `test` 不在匹配 chunk 名
  - 迁移：使用 test 函数
    `(module, { chunkGraph }) => chunkGraph.getModuleChunks(module).some(chunk => chunk.name === "name")`
- 在 `optimization.splitChunks` 中添加 `minRemainingSize`（自 alpha.13 起）
- `optimization.splitChunks` `filename` 可以赋值函数 (since alpha.17)
- `optimization.splitChunks` sizes 可以为每个源类型的 size 对象
  - `minSize`
  - `minRemainingSize`
  - `maxSize`
  - `maxAsyncSize`（自 alpha.13起）
  - `maxInitialSize`
- 在 `optimization.splitChunks` 中添加 `maxAsyncSize` 和 `maxInitialSize`：允许为初始和异步 chunk 指定不同的最大 size。
- 移除 `optimization.splitChunks` 中的 `name: true`：不再支持自动命名
  - 迁移：使用默认值。`chunkIds: "named"` 将为你的文件提供有用的名称以便于调试
- 添加 `optimization.splitChunks.cacheGroups[].idHint`：将提示如何命名 chunk id
- 移除 `optimization.splitChunks` 中的 `automaticNamePrefix`
  - **迁移**：使用 `idHint` 替代
- `optimization.splitChunks` 中的 `filename` 不再局限于初始 chunk（自 alpha.11 起）
- 添加 `optimization.mangleExports`（自 alpha.10 起）
- 移除 `output.devtoolLineToLine`
  - **迁移**：无替代方式
- `output.hotUpdateChunkFilename: Function` 现在被禁止：不会生效。
- `output.hotUpdateMainFilename: Function` 现在被禁止：不会生效。
- `module.rules` 中的 `resolve` 和 `parser` 将以不同的方式合并（对象会进行深度合并，数组将采用 `"..."` 进行展开以获取之前的值）（自 alpha.13 起）
- `module.rules` 中的 `query` 和 `loaders` 已被移除（自 alpha.13 起）
- 添加 `stats.chunkRootModules`：展示 chunk 的根模块
- 添加 `stats.orphanModules`：展示未触发的模块。
- 添加 `stats.runtime`：展示 runtime 模块
- 添加 `stats.chunkRelations`：显示 parent/children/sibling chunk（自 alpha.1 起）
- 添加 `stats.preset`：选择 preset（自 alpha.1 起）
- `BannerPlugin.banner` 签名变更
  - 移除 `data.basename`
  - 移除 `data.query`
  - **迁移**：从 `filename` 中进行提取
- 移除 `SourceMapDevToolPlugin` 中的 `lineToLine`
  - **迁移**：无替代方式
- `[hash]` 不支持完整编译的 hash
  - **迁移**：使用 `[fullhash]` 或采用更好的 hash 选项
- `[modulehash]` 被废弃
  - **迁移**：使用 `[hash]` 代替
- `[moduleid]` 被废弃
  - **迁移**：使用 `[id]` 代替
- 移除 `[filebase]`
  - **迁移**：使用 `[base]` 代替
- 基于文件模板的新占位符（即 SourceMapDevToolPlugin）
  - `[name]`
  - `[base]`
  - `[path]`
  - `[ext]`
- 当向 `externals` 传递一个函数时，它将具有不同的函数签名 `({ context, request }, callback)`
  - **迁移**：更改函数签名
- 添加 `experiments`（请参阅上述实验部分，自 alpha.19 起）
- 添加 `watchOptions.followSymlinks`（自 alpha.19 起）

## 默认值变更

- 默认情况下，仅对 `node_modules` 启用 `module.unsafeCache`
- 在生产模式下，`optimization.moduleIds` 的默认值从 `size` 替换为 `deterministic`
- 在生产模式下，`optimization.chunkIds` 的默认值从 `total-size` 替换为 `deterministic`
- 在 `none` 模式下，`optimization.nodeEnv` 默认为 `false`
- `optimization.splitChunks` 中的 `minRemainingSize` 默认为 `minSize`（自 alpha.13 起）
  - 如果剩余部分过小，这将减少创建 chunk 的数量
- 当使用 `cache` 时，`resolve(Loader).cache` 默认为 `true`
- `resolve(Loader).cacheWithContext` 默认为 `false`
- ~`node.global` 默认为 `false`~（自 alpha.4 起被移除）
- `resolveLoader.extensions` 移除 `.json`（自 alpha.8 起）
- 当 node-`target` 时，`node.global` 中的 `node.__filename` 和 `node.__dirname` 默认为 `false`（自 alpha.14 起）

# Major Internal Changes

The following changes are only relavant for plugin authors:

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

(since alpha.24) Timestamping for directories is possible now, which allows serialization of ContextModules

## Filesystems

Next to `compiler.inputFileSystem` and `compiler.outputFileSystem` there is a new `compiler.intermediateFileSystem` for all fs actions that are not considers as input or output, like writing records, cache or profiling output.

The filesystems have now the `fs` interface and do no longer demand additional methods like `join` or `mkdirp`. But if they have methods like `join` or `dirname` they are used.

(since alpha.16)

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

## SizeOnlySource after emit

webpack now replaces the Sources in `Compilation.assets` with `SizeOnlySource` variants to reduce memory usage.

(since alpha.8)

## ExportsInfo

The way how information about exports of modules are stored has been refactored. The ModuleGraph now features a `ExportsInfo` for each `Module`, which stores information per export. It also stores information about unknown exports and if the module is used in side-effect-only way.

For each export the following information is stored:

* Is the export used? yes, no, not statically known, not determined. (see also `optimization.usedExports`)
* Is the export provided? yes, no, not statically known, not determined. (see also `optimization.providedExports`)
* Can be export name be renamed? yes, no, not determined.
* The new name, if the export has been renamed. (see also `optimization.mangleExports`)

(since alpha.10)

## Code Generation Phase

The Compilation features Code Generation as separate compilation phase now. It no longer runs hidden in `Module.source()` or `Module.getRuntimeRequirements()`.

This should make the flow much cleaner. It also allows to report progress for this phase and makes Code Generation more visible when profiling.

MIGRATION: `Module.source()` and `Module.getRuntimeRequirements()` are deprecated now. Use `Module.codeGeneration()` instead.

(since alpha.31)

## Improved Code Generation

webpack detects when ASI happens and generates shorter code when no semicolons are inserted. `Object(...)` -> `(0, ...)` (since alpha.22)

webpack merges multiple export getters into a single runtime function call: `r.d(x, "a", () => a); r.d(x, "b", () => b);` -> `r.d(x, {a: () => a, b: () => b});` (since alpha.22)

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
- ProgressPlugin `entries` option now defaults to `on`
- ProgressPlugin `activeModules` option now defaults to `off` (since alpha.31)
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
- NormalModuleFactory.fileDependencies, NormalModuleFactory.contextDependencies and NormalModuleFactory.missingDependencies are now LazySets (since alpha.20)
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
