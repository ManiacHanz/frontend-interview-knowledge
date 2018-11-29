# koa-router

*个人对于koa-router的源码解读，仅供参考，欢迎纠错*

[仓库地址](https://github.com/alexmingoia/koa-router)


基础使用例子

```js
var Koa = require('koa');
var Router = require('koa-router');

var app = new Koa();
var router = new Router();

router.get('/', (ctx, next) => {
  // ctx.router available
});

app
  .use(router.routes())
  .use(router.allowedMethods());
```

那么肯定`Router`暴露出来的是一个`构造函数`, `new`的时候做了下面的操作

```js
module.exports = Router;

function Router(opts) {
  if (!(this instanceof Router)) {
    return new Router(opts);
  }

  this.opts = opts || {};
  this.methods = this.opts.methods || [
    'HEAD',
    'OPTIONS',
    'GET',
    'PUT',
    'PATCH',
    'POST',
    'DELETE'
  ];

  // 存放中的value是 一个fn，也就是真正当成middleware被调用的函数
  this.params = {};
  // 用来存放注册好的路由实例 也就是Layer实例
  this.stack = [];
};
```

接着往下看，第一段是一个`forEach`代码，这里的`methods`是引用的一个[仓库](https://github.com/jshttp/methods)，大概作用就是把`node`里`http`模块中的`methods`属性提出来小写了一下，然后提供一个`callback`

```js
[
    'get',
    'post',
    'put',
    'head',
    'delete',
    'options',
    'trace',
    'copy',
    'lock',
    'mkcol',
    //....
  ]
```

接着把这些`methods`循环，挂在`Router`的原型上。这里3个参数由于第一个`name`是选传，这里用了一个技巧，在`redux`的`createStore(reducers, initStore, applyMiddleware)`方法里也见到了。 接着调用了一下 `register`方法

```js
var methods = require('methods');

methods.forEach(function (method) {
  Router.prototype[method] = function (name, path, middleware) {
    var middleware;

    if (typeof path === 'string' || path instanceof RegExp) {
      middleware = Array.prototype.slice.call(arguments, 2);
    } else {
        // 这里表示name没传
      middleware = Array.prototype.slice.call(arguments, 1);
      path = name;
      name = null;
    }

    this.register(path, [method], middleware, {
      name: name
    });

    return this;
  };
});
```



```js
/**
 * Create and register a route.
 *
 * @param {String} path 路径
 * @param {Array.<String>} methods HTTP method  GET POST... 
 * @param {Function} middleware 处理的中间件 接收多个
 * @param {Object} opts 对象参数
 * @returns {Layer}
 * @private
 */

Router.prototype.register = function (path, methods, middleware, opts) {
  opts = opts || {};

  var router = this;
  // stack是一个Array
  var stack = this.stack;

  // support array of paths
  // 如果path是数组。就循环递归调用
  if (Array.isArray(path)) {
    path.forEach(function (p) {
      router.register.call(router, p, methods, middleware, opts);
    });

    return this;
  }

  // create route
  // 见下面Layer的分析 以及配置的参数
  var route = new Layer(path, methods, middleware, {
    end: opts.end === false ? opts.end : true,
    name: opts.name,
    sensitive: opts.sensitive || this.opts.sensitive || false,
    strict: opts.strict || this.opts.strict || false,
    prefix: opts.prefix || this.opts.prefix || "",
    ignoreCaptures: opts.ignoreCaptures
  });

  // prefix是前置 比如/user/id /user/getInfo 都在/user的prefix下
  // setPrefix就会把这个拼接起来
  if (this.opts.prefix) {
    route.setPrefix(this.opts.prefix);
  }

  // add parameter middleware
  // 初始化的时候this.params是一个空对象
  // 第二个this是循环出来的key形成的数组, 然后把key和value分别传进param处理
  // 这里面 value是一个fn，在param函数里会声明一个在ctx的上下文中调用的函数
  // 同时对Layer实例也就是 这里的这个route上的stack和param属性做一些处理，然后返回route出来
  Object.keys(this.params).forEach(function (param) {
    route.param(param, this.params[param]);
  }, this);
  // 这里把注册好的路由放进栈里
  stack.push(route);
  // 注册完毕  返回route 保证链式调用
  return route;
};
```

看到这里，知道了注册返回的`route`其实是一个`Layer`的实例，那么我们来看一下`Layer`是何方神圣

##### Layer

[仓库地址](https://github.com/pillarjs/path-to-regexp)

是通过给定的`请求方法`，`路径`，`中间件处理函数`，以及`配置`来初始化一个新的路由

```js
/**
 * Initialize a new routing Layer with given `method`, `path`, and `middleware`.
 *
 * @param {String|RegExp} path Path string or regular expression.
 * @param {Array} methods Array of HTTP verbs.
 * @param {Array} middleware Layer callback/middleware or series of.
 * @param {Object=} opts
 * @param {String=} opts.name route name
 * @param {String=} opts.sensitive case sensitive (default: false)区分大小写
 * @param {String=} opts.strict require the trailing slash (default: false)
 * @returns {Layer}
 * @private
 */

function Layer(path, methods, middleware, opts) {
  this.opts = opts || {};
  this.name = this.opts.name || null;
  this.methods = [];
  this.paramNames = [];
  this.stack = Array.isArray(middleware) ? middleware : [middleware];

  methods.forEach(function(method) {
    var l = this.methods.push(method.toUpperCase());
    if (this.methods[l-1] === 'GET') {
      this.methods.unshift('HEAD');
    }
  }, this);

  // ensure middleware is a function
  this.stack.forEach(function(fn) {
    var type = (typeof fn);
    if (type !== 'function') {
      throw new Error(
        methods.toString() + " `" + (this.opts.name || path) +"`: `middleware` "
        + "must be a function, not `" + type + "`"
      );
    }
  }, this);

  this.path = path;
  this.regexp = pathToRegExp(path, this.paramNames, this.opts);

  debug('defined route %s %s', this.methods, this.opts.prefix + this.path);
};

```


##### router prefixed

```js
// example
var router = new Router({
  prefix: '/users'
});

router.get('/', ...); // responds to "/users"
router.get('/:id', ...); // responds to "/users/:id"
```

很简单，就是经过一个正则处理掉`prefix`的`'/'`， 然后在调用`Layer`的`setPrefix`方法，把传进来的`prefix`和`path`拼接起来

```js
Router.prototype.prefix = function (prefix) {
  prefix = prefix.replace(/\/$/, '');

  this.opts.prefix = prefix;

  this.stack.forEach(function (route) {
    route.setPrefix(prefix);
  });

  return this;
};
```


```js
app.use(router.routes())
app.use(router.allowedMethods())
```

##### router.routes

由于要注册进`app`当成`middleware`处理，所以我们来看看下面两个方法

```js
/**
 * Returns router middleware which dispatches a route matching the request.
 *
 * @returns {Function}
 */

Router.prototype.routes = Router.prototype.middleware = function () {
  // 实例
  var router = this;

  var dispatch = function dispatch(ctx, next) {
    debug('%s %s', ctx.method, ctx.path);
    // ctx.path 最终指向的就是request.path 代表请求的路径名
    var path = router.opts.routerPath || ctx.routerPath || ctx.path;
    // matched是一个含有path, pathAndMethod, route三个属性的对象，下面我们在详解这个方法
    var matched = router.match(path, ctx.method);
    var layerChain, layer, i;

    // .matched是一个自定义属性挂在ctx上
    if (ctx.matched) {
      ctx.matched.push.apply(ctx.matched, matched.path);
    } else {
      ctx.matched = matched.path;
    }

    // 把这个实例挂在ctx上
    ctx.router = router;

    // 如果不匹配就 跳到下个中间件
    if (!matched.route) return next();

    var matchedLayers = matched.pathAndMethod
    var mostSpecificLayer = matchedLayers[matchedLayers.length - 1]
    ctx._matchedRoute = mostSpecificLayer.path;
    if (mostSpecificLayer.name) {
      ctx._matchedRouteName = mostSpecificLayer.name;
    }

    layerChain = matchedLayers.reduce(function(memo, layer) {
      memo.push(function(ctx, next) {
        ctx.captures = layer.captures(path, ctx.captures);
        ctx.params = layer.params(path, ctx.captures, ctx.params);
        ctx.routerName = layer.name;
        return next();
      });
      return memo.concat(layer.stack);
    }, []);

    return compose(layerChain)(ctx, next);
  };

  dispatch.router = this;

  return dispatch;
};
```

