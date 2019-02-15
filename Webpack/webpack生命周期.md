
## 简析webpack生命周期

[官方文档](https://webpack.docschina.org/api/compiler-hooks/)

理解不深，还是直接放上大佬的[解析](http://taobaofed.org/blog/2016/09/09/webpack-flow/)


大概会有`初始化`和`run`开始两个过程

初始化里有大概这几个钩子  这几个都是 `SyncHook` 
* entryOption    entry 配置项 处理过之后，执行插件。
* afterPlugins     设置完初始插件之后，执行插件。
* afterResolvers     resolver 安装完成之后，执行插件。
* environment      environment 准备好之后，执行插件。
* afterEnvironment     environment 安装完成之后，执行插件。

`run`以后里面比较重要的有这几个
* compile 开始编译
* make 从入口点分析模块及其依赖的模块，创建这些模块对象
* build-module 构建模块
* after-compile 完成构建
* seal 封装构建结果
* emit 把各个chunk输出到 output 目录之前。
* after-emit 完成输出

比如说[这里](./plugin机制浅析.md)里面提到的HtmlWebpackPlugin就用了`make`用来分析配置文件中传进来的`option`，用了`emit`在生成之前去分析了各个依赖的包括`favicon``js``css`文件的路径及名称，这样才能在输出之前注入到生成的html里。同时还用了`thisCompilation`钩子，在触发`compilation`事件做了一些例如html缓存是否过期、依赖文件路径是否有变化等分析


![](./assets/webpack-lifecycle.png "webpack-lifecycle的截图")