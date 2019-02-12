
## 浅析 webpack 中的插件机制

*才疏学浅，抛砖引玉*

需要关注的文档地址

官方文档
[编写一个插件](https://webpack.docschina.org/contribute/writing-a-plugin/)
[Plugin API](https://webpack.docschina.org/api/plugins/)

以及大佬分享的
[Tapable分析](https://www.codercto.com/a/21587.html)

> 本质上，webpack 是一个现代 JavaScript 应用程序的静态模块打包器(static module bundler)

而之所以webpack如此强大，可以理解成除了打包以外的各种任务都是由插件堆砌而成。插件可以执行的任务包括但不限于：打包优化、资源管理（新增、删除）和注入环境变量等等。

在弄清楚`Plugin`机制之前有3个非常重要的概念
* compiler 
* compilation
* tapable

这里引用官方文档的解释

* `compiler` 对象代表了完整的 webpack 环境配置。这个对象在启动 webpack 时被一次性建立，并配置好所有可操作的设置，包括 options，loader 和 plugin。当在 webpack 环境中应用一个插件时，插件将收到此 compiler 对象的引用。可以使用 compiler 来访问 webpack 的主环境。

* `compilation` 对象代表了一次资源版本构建。当运行 webpack 开发环境中间件时，每当检测到一个文件变化，就会创建一个新的 compilation，从而生成一组新的编译资源。一个 compilation 对象表现了当前的模块资源、编译生成资源、变化的文件、以及被跟踪依赖的状态信息。compilation 对象也提供了很多关键时机的回调，以供插件做自定义处理时选择使用。

看下面一段代码

```js
const webpack = require('webpack')

const compiler = webpack({
  entry: '',
  output: '',
})
```

是的，其实平时我们都会`npm start`，用`webpack`去跑我们的`webpack.config.js`，而返回的这个对象就是`compiler`，里面包含了这些配置信息，可以让插件在编写的时候获取到。而`compilation`也没什么神秘的，可以这么理解，每次`webpack`重新编译的时候都会新实例化一个`compilation`，这里面包含了重新编译出来的资源的信息，从而让插件能获取到新编译出来的资源

而`tapable`这个库就是`webpack`之所以能这么强大的灵魂，它提供了一系列的钩子，让`webpack`知道在何时应该去调用`Plugin`的方法来管理资源。那么我们先从`tapable`这里说起

当研究官方文档[compiler钩子](https://webpack.docschina.org/api/compiler-hooks/)这一块的时候，每一个钩子下总会有一个`SyncHook`,`SyncBailHook`,`AsyncHook`等等的标识，查了资料后终于知道，其实这就是`tapable`提供给`webpack`的灵魂。

在`tapable`中，一共暴露了9个钩子，包括4个同步和5个异步，

```js
const {
	SyncHook,
	SyncBailHook,
	SyncWaterfallHook,
	SyncLoopHook,
	AsyncParallelHook,
	AsyncParallelBailHook,
	AsyncSeriesHook,
	AsyncSeriesBailHook,
	AsyncSeriesWaterfallHook
 } = require("tapable")
```

当然这里面是有明确的分工的。
* 首先`Sync`的钩子是同步执行，只能用`tap`注册（就理解成 `on` ），而`Async`的钩子是异步任务，可以用`tap`、`tapSync`、`tapPromise`注册。在这里可以先想成`tap`直接写成回调函数的形式，而后面两个可以把回调改成`Promise`来执行，后面会在例子中解释

* `Sync`的钩子直接如果需要通信，往往是通过`return`的返回值来进行联络；而`Async`的钩子需要联络，一般是通过`callback()` -- 可以理解成koa中的`next()`的那种形式 -- 来进行联络。

* 其次，带`Bail`的钩子，一般只如果有一个返回值不为 `null` ，就会跳过剩下的逻辑，否则和普通的顺序执行相同

* 带`Waterfall`的钩子，后一个任务要拿到上一个任务的返回值，或者说是`callback()`有参数

* `SyncLoopHook`, 监听函数返回`true`就继续，否则就中断


有了这些知识基础，我们看一下一个插件的基本框架

> 插件是由「具有 apply 方法的 prototype 对象」所实例化出来的。这个 apply 方法在安装插件时，会被 webpack compiler 调用一次。apply 方法可以接收一个 webpack compiler 对象的引用，从而可以在回调函数中访问到 compiler 对象。

而`compiler.hooks`就要用到上面关于`tapable`的知识了。后面的`done`需要去查阅[文档](https://webpack.docschina.org/api/compiler-hooks/)看是在`webpack`编译到哪一个阶段执行。`tap`中接收两个参数，第一个是这个钩子的标志，是一个字符串，没有什么意义；第二个参数就是执行的逻辑

```js
// /webpack.config.js
module.exports = {
  // ...
  plugins: [
    new HelloWorldPlugin({})
  ]
}

// /HelloWorldPlugin.js
class HelloWorldPlugin {
  constructor(options) {
    this.options = options;
  }

  apply(compiler) {
    compiler.hooks.done.tap('HelloWorldPlugin', () => {
      console.log('Hello World!');
      console.log(this.options);
    });
  }
}

module.exports = HelloWorldPlugin;
```

如此一个简单的`Plugin`就完成了。当你执行的`webpack`相关任务的时候，会发现在编译完成后打印出'Hello world'。当然这个过于浮夸，下面我拿一个重用的插件 `html-webpack-plugin` 来做浅析。如有错误，请多指正

[html-webpack-plugin](https://github.com/jantimon/html-webpack-plugin)

实力不够，这里就针对常用场景看一下源码里的大概实现

具体主要有一下几个作用
* 新生成一个html文件
* 自动引用带有hash值的资源地址
* 可以自定义meta, title, filename等
* 可以在head, body中注入内容
* 可以控制hash, cache等

那么大概看一下源码


*结论：* 大致结论是，`html-webpack-plugin`利用钩子函数，去获取`output`出来的文件，把需要的静态资源注入到统一管理的`对象`内，然后在另外的钩子中去生成一个（和选项）合并后的`.html`，然后把静态文件的地址注入进去。同时利用`compilation`对象，判断`assets`资源是否发生变化，发生变化就去更新`.html`中的路径


```js

class HtmlWebpackPlugin {
  // constructor 主要是做配置项的合并
  constructor (options) {
    
    const userOptions = options || {};

    const defaultOptions = {
      template: path.join(__dirname, 'default_index.ejs'),
      templateContent: false,
      templateParameters: templateParametersGenerator,
      filename: 'index.html',
      hash: false,
      inject: true,
      compile: true,
      favicon: false,
      minify: undefined,
      cache: true,
      showErrors: true,
      chunks: 'all',
      excludeChunks: [],
      chunksSortMode: 'auto',
      meta: {},
      title: 'Webpack App',
      xhtml: false
    };

    this.options = Object.assign(defaultOptions, userOptions);

    if (!userOptions.template && this.options.templateContent === false && this.options.meta) {
      const defaultMeta = {
        viewport: 'width=device-width, initial-scale=1'
      };
      this.options.meta = Object.assign({}, this.options.meta, defaultMeta, userOptions.meta);
    }

    this.childCompilerHash = undefined;
    this.childCompilationOutputName = undefined;
    this.assetJson = undefined;
    this.hash = undefined;
    this.version = HtmlWebpackPlugin.version;
  }

  // apply 核心函数
  apply (compiler) {
    const self = this;
    let isCompilationCached = false;
    let compilationPromise;

    this.options.template = this.getFullTemplatePath(this.options.template, compiler.context);

    const filename = this.options.filename;
    if (path.resolve(filename) === path.normalize(filename)) {
      this.options.filename = path.relative(compiler.options.output.path, filename);
    }
    // compiler的mode
    const isProductionLikeMode = compiler.options.mode === 'production' || !compiler.options.mode;

    const minify = this.options.minify;
    if (minify === true || (minify === undefined && isProductionLikeMode)) {
      this.options.minify = {
        collapseWhitespace: true,
        removeComments: true,
        removeRedundantAttributes: true,
        removeScriptTypeAttributes: true,
        removeStyleLinkTypeAttributes: true,
        useShortDoctype: true
      };
    }

    // Clear the cache once a new HtmlWebpackPlugin is added
    childCompiler.clearCache(compiler);

    // 第一个遇到的钩子 thisCompilation  -- SyncHook  触发 compilation 事件之前执行
    compiler.hooks.thisCompilation.tap('HtmlWebpackPlugin', (compilation) => {
      // 过期的清除
      if (childCompiler.hasOutDatedTemplateCache(compilation)) {
        childCompiler.clearCache(compiler);
      }
      // Add this instances template to the child compiler
      childCompiler.addTemplateToCompiler(compiler, this.options.template);

      // additionalChunkAssets     为 chunk 创建附加资源(asset) 来保持监听
      compilation.hooks.additionalChunkAssets.tap('HtmlWebpackPlugin', () => {
        const childCompilerDependencies = childCompiler.getFileDependencies(compiler);
        childCompilerDependencies.forEach(fileDependency => {
          compilation.compilationDependencies.add(fileDependency);
        });
      });
    });

    // 第二个compiler钩子 make  AsyncParallelHook
    // 这里的作用有合并option里的模板文件
    // 异步使用 callback() 往后执行
    compiler.hooks.make.tapAsync('HtmlWebpackPlugin', (compilation, callback) => {
      // Compile the template (queued)
      compilationPromise = childCompiler.compileTemplate(self.options.template, self.options.filename, compilation)
        .catch(err => {})
        .then(compilationResult => {
          // If the compilation change didnt change the cache is valid
          isCompilationCached = Boolean(compilationResult.hash) && self.childCompilerHash === compilationResult.hash;
          self.childCompilerHash = compilationResult.hash;
          self.childCompilationOutputName = compilationResult.outputName;
          callback();
          return compilationResult.content;
        });
    });

    // 第三个compiler钩子 emit  --   AsyncSeriesHook 异步并行插件    生成资源到 output 目录之前。
    compiler.hooks.emit.tapAsync('HtmlWebpackPlugin',
      (compilation, callback) => {
        // entry点循环出来
        const entryNames = Array.from(compilation.entrypoints.keys());
        const filteredEntryNames = self.filterChunks(entryNames, self.options.chunks, self.options.excludeChunks);
        const sortedEntryNames = self.sortEntryChunks(filteredEntryNames, this.options.chunksSortMode, compilation);
        const childCompilationOutputName = self.childCompilationOutputName;

        // ...

        // assets不变  html就不用变，直接跳到下一个中间件
        const assetJson = JSON.stringify(self.getAssetFiles(assets));
        if (isCompilationCached && self.options.cache && assetJson === self.assetJson) {
          return callback();
        } else {
          self.assetJson = assetJson;
        }

        // 获取 favicon的路径的函数
        const assetsPromise = this.getFaviconPublicPath(this.options.favicon, compilation, assets.publicPath)
          .then((faviconPath) => {});

        // 转换js和css路径到一个统一管理的对象里，然后后面根据这个对象输出出来
        const assetTagGroupsPromise = assetsPromise
          .then(({assets}) => getHtmlWebpackPluginHooks(compilation).alterAssetTags.promise({ 
            // ...
          }))
          .then(({assetTags}) => {
           // ...
          });

        // 执行 上面那些
        const templateExectutionPromise = Promise.all([assetsPromise, assetTagGroupsPromise, templateEvaluationPromise])
          // Execute the template
          .then(([assetsHookResult, assetTags, compilationResult]) => typeof compilationResult !== 'function'
            ? compilationResult
            : self.executeTemplate(compilationResult, assetsHookResult.assets, { headTags: assetTags.headTags, bodyTags: assetTags.bodyTags }, compilation));

        // 注入meta script link等信息的promise
        const injectedHtmlPromise = Promise.all([assetTagGroupsPromise, templateExectutionPromise])
          // Allow plugins to change the html before assets are injected
          .then(([assetTags, html]) => {
            const pluginArgs = {html, headTags: assetTags.headTags, bodyTags: assetTags.bodyTags, plugin: self, outputName: childCompilationOutputName};
            return getHtmlWebpackPluginHooks(compilation).afterTemplateExecution.promise(pluginArgs);
          })
          .then(({html, headTags, bodyTags}) => {
            return self.postProcessHtml(html, assets, {headTags, bodyTags});
          });

        const emitHtmlPromise = injectedHtmlPromise
          // Allow plugins to change the html after assets are injected
          .then((html) => {
            const pluginArgs = {html, plugin: self, outputName: childCompilationOutputName};
            return getHtmlWebpackPluginHooks(compilation).beforeEmit.promise(pluginArgs)
              .then(result => result.html);
          })
          .catch(err => {
          })
          .then(html => {
          })
          .then((finalOutputName) => getHtmlWebpackPluginHooks(compilation).afterEmit.promise({
            outputName: finalOutputName,
            plugin: self
          }).catch(err => {
            console.error(err);
            return null;
          }).then(() => null));

        // Once all files are added to the webpack compilation
        // let the webpack compiler continue
        emitHtmlPromise.then(() => {
          callback();
        });
      });
  }

}


HtmlWebpackPlugin.version = 4;

HtmlWebpackPlugin.getHooks = getHtmlWebpackPluginHooks;
HtmlWebpackPlugin.createHtmlTagObject = createHtmlTagObject;

module.exports = HtmlWebpackPlugin;
```


