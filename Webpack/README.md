
# Knowledge Map


* [抽取公共文件](./抽取公共文件.md)

* [使用过webpack里面哪些plugin和loader](./常用plugins和loaders.md)

* [webpack里面的插件是怎么实现的](./plugin机制浅析.md)

* [webpack里面的Loader机制](./loader机制浅析.md)

* [dev-server是怎么跑起来](./浅析dev-server.md)
使用的`express`库，还是使用`http.createServer()`这个基础的api来跑起来，然后把一些`node`的方法处理后挂载了`dev-server`的`Server`上

* 使用import时，webpack对node_modules里的依赖会做什么   --  todo
目前看到的是会对每个依赖打一个被依赖的数组标识，这样在合成抽象语法树以后会明确的知道这个包被那些地方引用了。例如后期`splitChunks`的时候，就可以把公用的`chunk`提取处来
其他的 todo

* [import { Button } from 'antd'，打包的时候只打包button，分模块加载，是怎么做到的](./分包加载解析.md)

* webpack整个生命周期，loader和plugin有什么区别
loader需要在`build-module`的时候就去解析好，把所有的文件都输出成js文件，才能给接下去的步骤生成`AST`，但是plugin在可以在`compiler`的任何生命周期调用。比如`run`之前在处理配置项的时候，或者`emit`输出文件之前等等

* [webpack生命周期](./webpack生命周期.md)

* webpack打包的整个过程

  大佬的[源码解析](https://lihuanghe.github.io/2016/05/30/webpack-source-analyse.html)

  其中有几个关键节段对应的事件分别是：

  1. `entry-option` 初始化option

  2. `run` 开始编译
    实际就是`Compiler`类的`run`方法，以及编译的时候会产生一个核心的`Compilation`对象

  3. `make` 从entry开始递归的分析依赖，对每个依赖模块进行build
    比如import 'index.js' 会读取`index.js`然后继续进去读取`app.js`，然后继续进去读取`router.js`等等。大致是一个深度递归的过程

  4. before-resolve - after-resolve 对其中一个模块位置进行解析
    主要是option里的`resolve`，分策略去读取是绝对路径，相对路径还是`node_modules`里的等等

  5. build-module 开始构建 (build) 这个module,这里将使用文件对应的loader加载
    调用`do-build`方法，把所有的资源（html, css, js, assets）都生成一段js代码，才能交给`acorn`生成`AST`

  6. normal-module-loader 对用loader加载完成的module(是一段js代码)进行编译,用 `acorn` 编译,生成ast抽象语法树。
    `acorn`可以解决`AMD` `CMD` 或者`import` `require.ensure`等等不同规范引起的差异

  7. program 开始对ast进行遍历，当遇到require等一些调用表达式时，触发call require事件的handler执行，收集依赖，并。如：AMDRequireDependenciesBlockParserPlugin等

  8. seal 所有依赖build完成，下面将开始对chunk进行优化，比如合并,抽取公共模块,加hash
    会逐次对每个`module`和`chunk`进行整理，生成编译后的源码，同时保留编译前的原始内容。比如UglifyWebpackPlugin这种插件会在此时被调用

  9. bootstrap 生成启动代码

  10. emit 把各个chunk输出到结果文件







### DLL 优化

在古老的webpack里，使用dll-plugin开启dll优化，其实就是在第一次打包的时候建立一个映射表，一般叫manifest.json。后续请求的时候就直接去这里面找



### webpack优化

构建优化两大方向： **缓存**和**多核**

开发优化 
  * 使用别名这些配置增加开发体验和效率
  * 增加resolve.modules配置，减少webpack解析路径花费时间

构建优化
  * 合理拆包
  * 异步加载
  * tree shaking

### 理解HMR工作机制

这个机制还是使用的webpack自己的热更新机制，即在`module.hot.accept`里注入回调

理解成，webpack在运行时，在bundle中维护了一个hmr运行的环境，这个环境和浏览器有一个通信 -- websocket。 当webpack监听到文件改变，就会通过websocket向客户端发送两个文件，一个manifest.json表示文件列表，一个chunk.js表示一个或多个更新的代码片段。客户端收到这个后，就会去对应的文件里比对，看它本身能不能完成这个更新，同时是不是还要递归向上，去找到依赖这个文件的文件的更新