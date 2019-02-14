
## 浅析 webpack 中的 loader 机制

*才疏学浅，抛砖引玉*

需要关注的文档地址

官方文档
[编写一个loader](https://webpack.docschina.org/contribute/writing-a-loader/)
[Loader API](https://webpack.docschina.org/api/loaders/)

简单分析的两个库
[sass-loader](https://github.com/webpack-contrib/sass-loader/blob/master/lib/loader.js)
[bundle-loader](https://github.com/webpack-contrib/bundle-loader/blob/master/index.js)

> 所谓 loader 只是一个导出为函数的 JavaScript 模块。loader runner 会调用这个函数，然后把上一个 loader 产生的结果或者资源文件(resource file)传入进去。函数的 this 上下文将由 webpack 填充，并且 loader runner 具有一些有用方法，可以使 loader 改变为异步调用方式，或者获取 query 参数。

上面的文档解释的很清楚，`loader` 就是一个函数，函数的参数是需要处理的资源文件（也可以传入第二可选参数，代表sourceMap），一般就是`test`正则匹配到的文件，处理之后再返回出来。

一般来说，处理的结果是`String`,也可以是`Buffer`（Buffer通过`raw`来设置）。返回的方式有**同步**和**异步**两种，同步调用`this.callback()`，异步调用`this.async()`，如下

```js
// content 指资源文件 map 指sourcemap  meta可以作为loader间自定义的共享参数，webpack不会解析这个
// callback() 会增加一个error参数，作为解析失败，否则传null
// sync-loader
module.exports = function(content, map, meta) {
  return someSyncOperation(content);
};

module.exports = function(content, map, meta) {
  this.callback(null, someSyncOperation(content), map, meta);
  return; // 当调用 callback() 时总是返回 undefined
};

// async-loader
module.exports = function(content, map, meta) {
  var callback = this.async();
  someAsyncOperation(content, function(err, result) {
    if (err) return callback(err);
    callback(null, result, map, meta);
  });
};

module.exports = function(content, map, meta) {
  var callback = this.async();
  someAsyncOperation(content, function(err, result, sourceMaps, meta) {
    if (err) return callback(err);
    callback(null, result, sourceMaps, meta);
  });
};
```

还有一定比较重要的是`pitch`方法，我们都知道Loader是按照*从右到左*的调用，但是其实在**执行**loader之前，会从左到右的执行`pitch`方法，类似于一个洋葱圈模型

```js
// webpack.config.js
module.exports = {
  //...
  module: {
    rules: [
      {
        //...
        use: [
          'a-loader',
          'b-loader',
          'c-loader'
        ]
      }
    ]
  }
}
```

将会发生这些步骤：

```js
|- a-loader `pitch`
  |- b-loader `pitch`
    |- c-loader `pitch`
      |- requested module is picked up as a dependency
    |- c-loader normal execution
  |- b-loader normal execution
|- a-loader normal execution
```

如果某个loader的`pitch`方法给出了一个结果，那么这个过程会跳过剩下的loader，比如

```js
// b-loader.js
module.exports = function(content) {
  return someSyncOperation(content);
};
// .pitch属性
module.exports.pitch = function(remainingRequest, precedingRequest, data) {
  if (someCondition()) {
    // 返回的要是 String或者Buffer类型往后传递
    return 'module.exports = require(' + JSON.stringify('-!' + remainingRequest) + ');';
  }
};
```

那么上面的例子的步骤会跳过b和c，变成

```js
|- a-loader `pitch`
  |- b-loader `pitch` returns a module
|- a-loader normal execution
```

官方文档上以 bundle-loader 做例子, 上源码

*bundle-loader是在webpack环境下，异步加载bundle的一个loader，核心是require.ensure()，且只能在webpack中使用*
[require.ensure](https://webpack.docschina.org/api/module-methods/#require-ensure)


```js
var loaderUtils = require("loader-utils");

module.exports = function() {};
// 用的pitch方法 ，所以一般情况下，bundle-loader后面不会有其他的loader
module.exports.pitch = function(remainingRequest) {
  // cacheable 表示缓存
	this.cacheable && this.cacheable();
	var query = loaderUtils.getOptions(this) || {};
	if(query.name) {
		var options = {
			context: query.context || this.rootContext || this.options && this.options.context,
			regExp: query.regExp
		};
		var chunkName = loaderUtils.interpolateName(this, query.name, options);
		var chunkNameParam = ", " + JSON.stringify(chunkName);		
	} else {
		var chunkNameParam = '';
	}
	var result;
	if(query.lazy) {
		result = [
			"module.exports = function(cb) {\n",
			"	require.ensure([], function(require) {\n",
			"		cb(require(", loaderUtils.stringifyRequest(this, "!!" + remainingRequest), "));\n",
			"	}" + chunkNameParam + ");\n",
			"}"];
	} else {
		result = [
			"var cbs = [], \n",
			"	data;\n",
			"module.exports = function(cb) {\n",
			"	if(cbs) cbs.push(cb);\n",
			"	else cb(data);\n",
			"}\n",
			"require.ensure([], function(require) {\n",
			"	data = require(", loaderUtils.stringifyRequest(this, "!!" + remainingRequest), ");\n",
			"	var callbacks = cbs;\n",
			"	cbs = null;\n",
			"	for(var i = 0, l = callbacks.length; i < l; i++) {\n",
			"		callbacks[i](data);\n",
			"	}\n",
			"}" + chunkNameParam + ");"];
	}
  // 直接return，返回的必须是一段字符串 或者buffer
	return result.join("");
}
```

可以看出主要的逻辑就是`require.ensure`api，来完成异步加载的任务。再来看sass-loader的大概原理。可以看下`this.async`的方法

```js
const path = require('path');

const async = require('neo-async');
const pify = require('pify');
const semver = require('semver');

const formatSassError = require('./formatSassError');
const webpackImporter = require('./webpackImporter');
const normalizeOptions = require('./normalizeOptions');

let nodeSassJobQueue = null;

function sassLoader(content) {
  // 异步模式中必须调用this.async() 
  const callback = this.async();
  const isSync = typeof callback !== 'function';
  // this是webpack
  const self = this;
  const { resourcePath } = this;

  function addNormalizedDependency(file) {
    // node-sass returns POSIX paths
    // this.addDependency()的缩写。加入文件作为loader的依赖
    self.dependency(path.normalize(file));
  }

  if (isSync) {
    throw new Error(
      'Synchronous compilation is not supported anymore. See https://github.com/webpack-contrib/sass-loader/issues/333'
    );
  }

  let resolve = pify(this.resolve);

  // Supported since v4.27.0
  if (this.getResolve) {
    resolve = this.getResolve({
      mainFields: ['sass', 'main'],
      extensions: ['.scss', '.sass', '.css'],
    });
  }

  const options = normalizeOptions(
    this,
    content,
    webpackImporter(resourcePath, resolve, addNormalizedDependency)
  );

  // Skip empty files, otherwise it will stop webpack, see issue #21
  if (options.data.trim() === '') {
    callback(null, '');
    return;
  }
  // todo 详细方法暂时看不懂...没去看这个包是什么
  const render = getRenderFuncFromSassImpl(
    // 如果传了就用implementation去处理，否则用node-sass，在否则用sass
    options.implementation || getDefaultSassImpl()
  );
  // 这个render是 node-sass里的方法，第二个函数里的result应该就是处理后的结果
  // 然后通过是否需要sourcemap来转成字符串传给callback
  render(options, (err, result) => {
    if (err) {
      formatSassError(err, this.resourcePath);

      if (err.file) {
        this.dependency(err.file);
      }
      // callback()传入第一个参数必须是个错误信息，webpack会在这抛出错误
      callback(err);
      return;
    }

    if (result.map && result.map !== '{}') {
      result.map = JSON.parse(result.map);
      delete result.map.file;
      result.map.sources[0] = path.relative(process.cwd(), resourcePath);
      result.map.sourceRoot = path.normalize(result.map.sourceRoot);
      result.map.sources = result.map.sources.map(path.normalize);
    } else {
      result.map = null;
    }

    result.stats.includedFiles.forEach(addNormalizedDependency);
    // 把result的 css属性作为 content传个下一个loader， map视是否有sourcemap来定
    callback(null, result.css.toString(), result.map);
  });
}

function getRenderFuncFromSassImpl(module) {
  const { info } = module;
  const components = info.split('\t');
  // ...排除掉所有抛错的
  const [implementation, version] = components;

 
  // 调用的都是 module-- dart-sass, node-sass, sass上的render方法，或者是一队列的方式来处理
  if (implementation === 'dart-sass') {
    return module.render.bind(module);
  } else if (implementation === 'node-sass') {
    if (nodeSassJobQueue === null) {
      const threadPoolSize = Number(process.env.UV_THREADPOOL_SIZE || 4);

      nodeSassJobQueue = async.queue(
        module.render.bind(module),
        threadPoolSize - 1
      );
    }

    return nodeSassJobQueue.push.bind(nodeSassJobQueue);
  }

  throw new Error(`Unknown Sass implementation "${implementation}".`);
}
// 获取默认的处理核心
function getDefaultSassImpl() {
  let sassImplPkg = 'node-sass';

  try {
    require.resolve('node-sass');
  } catch (error) {
    try {
      require.resolve('sass');
      sassImplPkg = 'sass';
    } catch (ignoreError) {
      sassImplPkg = 'node-sass';
    }
  }

  // eslint-disable-next-line import/no-dynamic-require, global-require
  return require(sassImplPkg);
}

module.exports = sassLoader;

```

所以看来loader也并不是什么神秘的东西，只不是是把资源经过自己的处理以后转成字符串传给下一个loader。在中间需要调一些`webpack`的api而已