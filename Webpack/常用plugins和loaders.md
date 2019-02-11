
**缓慢更新...**

## Plugins

HtmlWebpackPlugin   [文档地址](https://webpack.docschina.org/plugins/html-webpack-plugin)
新生成一个`html`文件，并自动根据`entry`和`output`解决引用路径问题

clean-webpack-plugin   [文档地址](https://www.npmjs.com/package/clean-webpack-plugin)
打包前可以把dist文件夹清理掉

source-map     [文档地址](https://webpack.docschina.org/configuration/devtool)
开发环境下可以追踪报错信息到源代码，否则只会报道编译后的`bundle`文件中

webpack-dev-server         [文档地址](https://webpack.docschina.org/guides/development/#%E4%BD%BF%E7%94%A8-webpack-dev-server)
`webpack`提供的开发者服务器，本身是一个`express`服务器

CommonsChunkPlugin          [文档地址](https://webpack.docschina.org/plugins/commons-chunk-plugin)
从多个文件中提取公共`chunk`。在`webpack v4`中已经废弃，关注`splitChunks`[文档](https://webpack.docschina.org/plugins/split-chunks-plugin/)

CopyWebpackPlugin           [文档地址](https://webpack.docschina.org/plugins/copy-webpack-plugin)
可以把文件或文件夹复制到指定地址。例如静态资源

DefinePlugin               [文档地址](https://webpack.docschina.org/plugins/define-plugin)
在编译时创建一个可以配置的全局变量。注意`字符串`要保留本身的实际引号，使用`JSON.stringify('production')`来保留

ExtractTextWebpackPlugin    [文档地址](https://webpack.docschina.org/plugins/extract-text-webpack-plugin)
从`js`中把`css`文件抽离出来

HotModuleReplacementPlugin      [文档地址](https://webpack.docschina.org/plugins/hot-module-replacement-plugin)
在*开发*环境下启用**热替换**。可以用命令行`-- hot` 自动开启，并不用在`webpack.config.js`里面配置

IgnorePlugin                   [文档地址](https://webpack.docschina.org/plugins/ignore-plugin)
防止在`import`或者`require`调用时生成以下模块
> 使用moment的时候记住忽略他的本地化内容

NpmInstallWebpackPlugin       [文档地址](https://webpack.docschina.org/plugins/npm-install-webpack-plugin)
在开发环境下自动检测缺失的依赖并自动安装，暂不支持`yarn`

ProvidePlugin                  [文档地址](https://webpack.docschina.org/plugins/provide-plugin)
自动加载模块，不需要使用`import`或者`require`。比如全局使用`$`当`jquery`使用。**不推荐**

UglifyjsWebpackPlugin           [文档地址](https://webpack.docschina.org/plugins/uglifyjs-webpack-plugin)
`webpack v4+`设置`mode: production`会自动配置`UglifyjsWebpackPlugin`



## Loaders

[文档](https://webpack.docschina.org/loaders/)

file-loader / url-loader
将文件发送到输出文件夹，并返回（相对）URL。默认情况下，生成的文件的文件名就是文件内容的 MD5 哈希值并会保留所引用资源的原始扩展名。 后者可以在文件大小低于限制时，返回dataUrl

babel-loader 
编译ES2015+代码

style-loader 
将模块的导出作为样式添加到 DOM 中

css-loader 
解析 CSS 文件后，使用 import 加载，并且返回 CSS 代码

less-loader 
加载和转译 LESS 文件

sass-loader 
加载和转译 SASS/SCSS 文件

postcss-loader 
使用 PostCSS 加载和转译 CSS/SSS 文件

eslint-loader PreLoader
使用 ESLint 清理代码