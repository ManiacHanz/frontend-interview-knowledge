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

