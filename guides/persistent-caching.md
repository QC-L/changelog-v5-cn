欢迎来到持久化缓存指南。

# Opt-in

首先，要注意的是默认情况下不会启用持久化缓存。你可以自行选择启用。

为何如此？
webpack 旨在注重构建安全而非性能。
我们没有打算默认启用这一功能，主要原因在于此功能虽然有 95% 几率提升性能，但仍有 5% 的几率中断你的应用程序/工作流/构建。

这可能听起来很糟，但相信我它并非如此。
只不过需要开发人员进行额外的操作来配置它。

序列化与反序列化功能具有无需配置的开箱即用体验，但开箱即用的部分可能致使缓存失效。

什么是缓存失效？
webpack 需要确认 entry 的缓存何时会失效，并在失效时不再将其用于构建。
因此，当你应用程序修改文件时，就会发生此情况。

示例：修改 `magic.js`。
webpack 必须让 entry 为 `magic.js` 的缓存失效。
构建将重新处理该文件，即运行 babel，typescript 诸如此类工具，重新解析文件并运行代码生成。
webpack 可能还会致使 entry 为 `bundle.js` 的缓存失效。
然后根据原模块重新构建此文件。

为此，webpack 追踪了每个模块的 `fileDependencies` `contextDependencies` 以及 `missingDependencies`，并创建了文件系统快照。
此快照会与真实文件系统进行比较，当检测到差异时，将触发对应模块的重新构建。

webpack 给 `bundle.js` 的缓存 entry 设置了一个 `etag`，它为所有贡献者的 hash 值。
比较这个 `etag`，只有当它与缓存 entry 匹配时才能使用。

webpack 4 中的内存缓存也依赖上述这些。
从开发人员角度来说，这些都能够开箱即用，无需额外配置。
但对于 webpack 5 的持久化缓存来说，却充满着挑战。

以下操作均会让 webpack 使 entry 缓存失效：
* 当 npm 升级 loader 或 plugin 时
* 当更改配置时
* 当更改在配置中读取的文件时
* 当 npm 升级配置中使用的 dependencies 时
* 当不同命令行参数传递给 build 脚本时
* 当有自定义构建脚本并进行更改时

这变得非常棘手。
开箱即用的情况下，webpack 无法处理所有这些情况。
这就是我们为什么选择安全的方式，并将持久化缓存变为可选特性的原因。
我们希望读者可以学习如何启用持久化缓存，以为你提供正确的提示。
我们希望你知道需要使用哪种配置来处理你自定义的构建脚本。

# 构建依赖（dependencies），缓存版本（version）和缓存名（name）

为了处理构建过程中的依赖关系，webpack 提供了三个新工具：

## 构建依赖（Build dependencies）

此为全新的配置项 `cache.buildDependencies`，它可以指定构建过程中的代码依赖。
为了使它更简易，webpack 负责解析并遵循配置值的依赖。

值类型有两种：文件和目录。
目录类型必须以斜杠（`/`）结尾。其他所有内容都解析为文件类型。

对于目录类型来说，会解析其最近的 `package.json` 中的 dependencies。
对于文件类型来说，我们将查看 node.js 模块缓存以寻找其依赖。

示例：构建通常取决于 webpack 本身的 lib 文件夹：
你可以这样配置：

``` js
cache.buildDependencies: {
    defaultWebpack: ["webpack/lib/"]
}
```

当 `webpack/lib` 或 webpack 依赖的库（如，`watchpack`，`enhanced-resolved` 等）发生任何变化时，其缓存将失效。
`webpack/lib` 已是默认值，默认情况下无需配置。

另一个示例：构建依旧取决于你的配置文件。
具体配置如下：

``` js
cache.buildDependencies: {
    config: [__filename]
}
```

`__filename` 变量指向 node.js 中的当前文件。

当配置文件或配置文件中通过 `require` 依赖的任何内容发生更改时，也会使得持久化缓存失效。
当配置文件通过 `require()` 引用了所有使用过的插件时，它们也会成为构建依赖项。

如果配置文件通过 `fs.readFile` 读取文件，则将不会成为构建依赖项，因为 webpack 仅遵循 `require()`。
你需要手动将此类文件添加到 `buildDependencies` 中。

## 缓存版本（Version）

构建的某些依赖项不能单纯的依靠对文件的引用，如，从数据库读取的值，环境变量或命令行上传递的值。
对于这些值，我们给出了新的配置项 `cache.version`。

`cache.version` 类型为 string。传递不同的字符串将使持久化缓存失效。

示例：你的配置中可能会读取环境变量中的 `GIT_REV` 并将其与 `DefinePlugin` 一起使用以将其嵌入到 bundle 中。
这使得 `GIT_REV` 成为你构建的依赖项。
具体配置如下：

``` js
cache: {
    version: `${process.env.GIT_REV}`
}
```

## 缓存名（Name）

在某些情况下，依赖关系会在多个不同的值间切换，并且对于每个值更改都会使得持久化缓存失效，这显然是浪费资源的。
对于这类值，我们给出了新的配置项 `cache.name`。

`cache.name` 类型为 string。传递值将创建一个隔离且独立的持久化缓存。

`cache.name` 被用于对文件名进行持久化缓存。确保仅传递短小且 fs-safe 的名称。

示例：你的配置可以使用 `--env.target mobile|desktop` 参数为移动端或 PC 用户创建不同的构建。
具体配置如下：

``` js
cache: {
    name: `${env.target}`
}
```

# 性能优化

对大部分 node_modules 进行哈希处理并加盖时间戳以生存构建和常规依赖项，其代价非常昂贵，并且还会大大降低 webpack 的执行速度。
为避免这种情况出现，webpack 引入了相关的性能优化，默认情况下会跳过 `node_modules`，并使用 `package.json` 中的 `version` 和 `name` 作为数据源。

此优化将用于配置项 `cache.managedPaths` 中的所有 path。
它默认为 webpack 安装了 `node_modules` 目录。

启用此优化后，**请勿手动编辑 `node_modules`**。
你可以使用 `cache.managedPaths: []` 禁用它。

当使用 Yarn PnP 时，将启用另一个优化。
由于缓存内容不可变，yarn 缓存中的所有文件都将完全跳过哈希和时间戳的操作（甚至不会追踪 `version` 和 `name`）。

此操作由配置项 `cache.immutablePaths` 控制。
启用 Yarn PnP 时，默认为安装了 webpack 的 yarn 缓存。

不要手动编辑 yarn 缓存，因为这根本不可行。

# 使用持久化缓存

确保你已阅读并理解以上信息！

此为启用持久化缓存的典型配置：

``` js
cache: {
    type: "filesystem",
    buildDependencies: {
        config: [ __filename ] // 当你 CLI 自动添加它时，你可以忽略它
    }
}
```

## Watching

持久化缓存可用于单独构建和连续构建（watch）。

当设置 `cache.type: "filesystem"` 时，webpack 会在内部以分层方式启用文件系统缓存和内存缓存。
从缓存读取时，会先查看内存缓存，如果内存缓存未找到，则降级到文件系统缓存。
写入缓存将同时写入内存缓存和文件系统缓存。

文件系统缓存不会直接将对磁盘写入的请求进行序列化。它将等到编译过程完成且编译器处于空闲状态才会执行。
如此处理的原因是序列化和磁盘写入会占用资源，并且我们不想额外延迟编译过程。

针对单一构建，其工作流为：

* Loading cache
* Building
* Emitting
* Display results (stats)
* Persisting cache (if changed)
* Process exits

针对连续构建（watch），其工作流为：

* Loading cache
* Building
* Emitting
* Display results (stats)
* Attach filesystem watchers
* Wait `cache.idleTimeoutForInitialStore`
* Persisting cache (if changed)
* On change:
  * Building
  * Emitting
  * Display results (stats)
  * Wait `cache.idleTimeout`
  * Persisting cache (if changed)

你会发现两个新的配置项 `cache.idleTimeout` 和 `cache.idleTimeoutForInitialStore`，它们控制着持久化缓存之前编译器必须空闲的时长。
`cache.idleTimeout` 默认为 60s，`cache.idleTimeoutForInitialStore` 默认为 0s。
由于序列化阻止了事件循环，因此在序列化缓存时不进行缓存检测。
此延迟尝试避免由于快速编辑文件，而在 watch 模式下导致重新编译造成的延迟，同时尝试为下一次冷启动保持持久化缓存的最新状态。
这是一个折中的解决方案，可以设置适合你工作流的值。较小的值会缩短冷启动时间，但会增加延迟重新构建的风险。

## 错误处理

发生错误要恢复持久化缓存的方式，可以通过删除整个缓存并进行全新的构建，或者通过删除有问题的缓存 entry 并使得该项目保持未缓存状态来进行。

在这种情况下，webpack 的 logger 会发出警告。
欲了解更多，请参阅 `infrastructureLogging` 的配置项。

---

# Details

正常使用不需要以下信息。

## 使用 webpack 的高级工具指南

封装 webpack 的工具可以选择其他默认值。
当不允许使用自定义扩展的 webpack 时，由于可以完全控制所有构建的依赖项，因此可以默认打开持久化存储。

## CLI 指南

默认情况下，使用 webpack 的 CLI 可能会添加一些构建依赖关系，而 webpack 本身不会。

* 默认情况下，CLI 会将 `cache.buildDependencies.defaultConfig` 设置为所用的配置文件
* CLI 会将命令行参数附加到 `cache.version`
* 使用命令行参数时，CLI 可能会在 `cache.name` 中添加注释。

## 调试信息

使用如下配置，将输出额外的调试信息：

``` js
infrastructureLogging: {
    debug: /webpack\.cache/
}
```

## 内部工作流

* webpack 读取缓存文件。
  * 没有缓存文件 -> 没有构建缓存
  * 缓存文件中的 `version` 与 `cache.version` 不匹配 -> 没有构建缓存
* webpack 将解析快照（`resolve snapshot`）与文件系统进行对比
  * 匹配到 -> 继续后续流程
  * 没有匹配到：
    * 再次解析所有解析结果（`resolve results`）
      * 没有匹配到 -> 没有构建缓存
      * 匹配到 -> 继续后续流程
* webpack 将构建依赖快照（`build dependencies snapshot`）与文件系统进行对比
  * 没有匹配到 -> 没有构建缓存
  * 匹配到 -> 继续后续流程
* 对缓存 entry 进行反序列化（在构建过程中对较大的缓存 entry 进行延迟反序列化）
* 构建运行（有缓存或没有缓存）
  * 追踪构建依赖关系
    * 追踪 `cache.buildDependencies`
    * 追踪已使用的 loader
* 新的构建依赖关系已解析完成
  * 解析依赖关系已追踪
  * 解析结果已追踪
* 创建来自所有新解析依赖项的快照
* 创建来自所有新构建依赖项的快照
* 持久化缓存文件序列化到磁盘

## 序列化

所有支持序列化的 class 都需要注册一个序列化器：

``` js
webpack.util.serialization.register(Constructor, request, name, serializer);
```

`Constructor` 应为一个 class 或构造器函数。
对于任何需要序列化的对象的 `object.constructor` 将被用于查找序列化器（serializer）。

`request` 将被用于加载调用 `register` 模块。
它应指向当前模块。
它将以这种方式使用：`require(request)`。

`name` 被用于区分具有相同 `request` 的多个 `register` 调用。

`serializer` 是至少拥有 `serialize` 和 `deserialize` 两个方法的对象。

当需序列化对象时，请调用 `serializer.serialize(object, context)`。
`context` 是至少拥有一个 `write(anything)` 方法的对象
此方法将内容写入输出流。
传递的值也会被序列化。

当需要反序列化对象时，请调用 `serializer.deserialize(context)`。
`context` 是至少拥有一个 `read(): anything` 方法的对象。
此方法会反序列化输入流中的某些内容。
`deserialize` 必须返回反序列化后的对象。

`serialize` 和 `deserialize` 应以相同的顺序读取和写入相同的对象。

示例：

``` js
// some-module/lib/MyClass.js
class MyClass {
    constructor(a, b) {
        this.a = a;
        this.b = b;
        this.c = undefined;
    }
}

register(MyClass, "some-module/lib/MyClass", null, {
    seralize(obj, { write }) {
        write(obj.a);
        write(obj.b);
        write(obj.c);
    }
    deserialize({ read }) {
        const obj = new MyClass(read(), read());
        obj.c = read();
        return obj;
    }
});
```

基本数据类型和引用数据类型的序列化器都已被注册，即 string，number，Array，Set，Map，RegExp，plain objects，Error。
