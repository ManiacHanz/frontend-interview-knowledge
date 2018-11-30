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
        // 这里表示name没传。 middleware会转成数组
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


在这里面知道了每一个`router`实例的`stack`属性里面装的是什么东西了。这里通过引用指针的改变了这个数组

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
  // stack是一个Array 这里是引用指针，只用给了下面的stack.push()
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
  // 这个methods是上面通过slice处理过后的中间处理函数数组
  // 得到的route实例多了路径的正则属性regexp, 请求的method属性，中间件的stack属性
  var route = new Layer(path, methods, middleware, {
    end: opts.end === false ? opts.end : true,
    // router.get( 'user', '/user/:id' ,...)  第一个参数 传了就直接传下去，不传就是null
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
  // 这里把注册好的路由放进栈里 。指针其实也改了this.stack
  // route.stack就是对应的中间处理函数
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
 * @param {String|RegExp} path Path string or regular expression. 路径
 * @param {Array} methods Array of HTTP verbs.  HTTP的GET POST那些方法的字符串
 * @param {Array} middleware Layer callback/middleware or series of.  路径匹配后的中间件
 * @param {Object=} opts
 * @param {String=} opts.name route name
 * @param {String=} opts.sensitive case sensitive (default: false)区分大小写
 * @param {String=} opts.strict require the trailing slash (default: false)
 * @returns {Layer}
 * @private
 */

function Layer(path, methods, middleware, opts) {
  this.opts = opts || {};
  // 把name挂过来
  this.name = this.opts.name || null;
  this.methods = [];
  this.paramNames = [];
  // 不论是一个middleware，还是多个。都转成 数组
  this.stack = Array.isArray(middleware) ? middleware : [middleware];

  methods.forEach(function(method) {
    // 数组长度
    var l = this.methods.push(method.toUpperCase());
    // 如果最后一位是GET，就在顶位添加'HEAD'方法
    if (this.methods[l-1] === 'GET') {
      this.methods.unshift('HEAD');
    }
  }, this);

  // ensure middleware is a function
  // 只是一个debug 可以不看 保证每一个middleware都是一个func
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
  // pathToRegExp  把路径字符串转换成正则表达式
  this.regexp = pathToRegExp(path, this.paramNames, this.opts);

  debug('defined route %s %s', this.methods, this.opts.prefix + this.path);
};

```

看完`Layer`的实例化过程，其实就是在`route`上把路径，路劲正则，中间件数组，参数处理后挂在实例上，没什么神奇的。所以我们可以接着往下看了，如果遇到`route`上的方法再回头来看看`Layer`上挂在的方法的原理


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

还有一个比较关键的方法需要先了解一下

##### router.match

这里的逻辑大概是遍历所有的`route`实例--也就是`new Layer`的实例。当某一个实例上的`path`命中时放进一个数组，然后我们还需要把对应的`http method`匹配一次，同时命中才会把第三个变量改成`true`。然后把带着这3个变量的对象返回回去。因为有可能`/user`路径可以对应`GET`或者`POST`,不能同时命中他们的中间件

```js
/**
 * Match given `path` and return corresponding routes.
 * 根据传进去的path，返回应该响应的routes
 * @param {String} path
 * @param {String} method
 * @returns {Object.<path, pathAndMethod>} returns layers that matched path and
 * path and method.
 * @private
 */

Router.prototype.match = function (path, method) {
  // 上面看了register方法就知道这里 this.stack讲的是什么了-- 包括了所有的Layer实例
  // 一个实例对应唯一的path，但是可以对应很多个middleware
  var layers = this.stack;
  var layer;
  // 也就是说返回的一个标识是否命中的对象，3个值代表路径命中，请求方法命中才能matched
  var matched = {
    path: [],              // 匹配上的路由
    pathAndMethod: [],      // 匹配的请求方法
    route: false            // 如果有写方法就会改成true
  };

  for (var len = layers.length, i = 0; i < len; i++) {
    layer = layers[i];

    debug('test %s %s', layer.path, layer.regexp);
    // layer里包装了.match方法，其实就是用正则去匹配路径和这个layer（route）
    // 正则是每一个layer根据路径转化（path-to-regexp）的一个正则表达式
    if (layer.match(path)) {
      matched.path.push(layer);
      // ~-1=0 ~1=-2 ~2=-3...    0转换了就是false  其他都是true 
      // 如果是长度为0 或者layer.methods里包含method
      if (layer.methods.length === 0 || ~layer.methods.indexOf(method)) {
        matched.pathAndMethod.push(layer);
        if (layer.methods.length) matched.route = true;
      }
    }
  }

  return matched;
};
```

积累的东西差不多了，可以往下看了

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
    // 代表命中的路径和http method
    var matched = router.match(path, ctx.method);
    var layerChain, layer, i;

    // 同一个路径有可能对应多个中间件处理，如果有多个需要把他们放在一起
    if (ctx.matched) {
      ctx.matched.push.apply(ctx.matched, matched.path);
    } else {
      ctx.matched = matched.path;
    }

    // 把这个实例挂在ctx上
    ctx.router = router;

    // 表明没有命中路由
    if (!matched.route) return next();
    
    // 下面的都是命中了路由
    // 即命中path也命中method的layer实例
    var matchedLayers = matched.pathAndMethod
    // 取最后一个
    var mostSpecificLayer = matchedLayers[matchedLayers.length - 1]
    // 把路径挂到ctx上
    ctx._matchedRoute = mostSpecificLayer.path;
    // 如果给路径命名了就再挂在ctx上
    if (mostSpecificLayer.name) {
      ctx._matchedRouteName = mostSpecificLayer.name;
    }

    // 最后又是用老朋友compose链式调用所有命中的路由 这里也是用的koa-compose实现一个洋葱圈模型
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

再回头看`app.use(router.routes())`这句话就能理解，整个的流程大概就是先去匹配`path`和`method`两个关键的因素，匹配上了以后把所有的`middleware`数组通过`koa-compose`函数进行洋葱圈模型的调用。也就是说，除开`path`和`method`两个匹配因子，`router`就是一个普通的`koa`中间件集合


##### allowedMethods

`allowedMethods`是用来针对`OPTIONS`请求头 -- 这个请求头带有包含请求方法的`Allow` -- 返回单独的响应中间件处理。 目前在下没怎么用过。以后需要的时候在更新一下