
# koa2中间件执行的原理浅析

**先感谢[qianlongo](https://github.com/qianlongo)对`koa`的中间件执行原理的解析的[分享](https://github.com/qianlongo/resume-native/issues/1)，结合着源码发现`koa2`已经放弃`Generator`改用`Async`了，所以这边也总结一下自己的理解，加深印象**

[仓库地址](https://github.com/koajs/koa)

> koa的代码不多，但是引用的模块真是多，水平有限，大家结合着自己的理解看

有关中间件的代码被放在了`application.js`的文件里。稍微看一下这边的代码

```js
module.exports = class Application extends Emitter {
    listen(...args) {
        // ...
        const server = http.createServer(this.callback());
        return server.listen(...args);
    }
}
```

`app`是`application`的实例，`app.listen`是`koa`启动核心，所以我们从`listen`方法开始看。这里面的核心代码是[http.createServer([options][, requestListener])](http://nodejs.cn/api/http.html#http_http_createserver_options_requestlistener)这个方法，*返回一个新建的`http.server`实例*。在这里koa直接忽略了第一个`options`（类型应该是`obj`），直接传入的是一个`requestListener`。所以我们继续看一下这个[监听者](http://nodejs.cn/api/http.html#http_event_request)。关于这个解释，官方文档是

> `request`事件。每次接收到一个请求时触发。每个连接可能有多个请求（在HTTP`keep-alive`连接的情况下）

所以我们可以理解在每次接到新请求的时候都会触发这个`this.callback`方法。那在它里面究竟做了什么呢

除开错误边界处理，其实里面就做了两件事
* 新建了一个ctx实例
* 把所有中间件`compose`一下传个一个叫做`handleRequest`的方法

第一点好理解，毕竟`koa`要把每次请求的`req,res`都挂到`ctx`上，肯定需要创建一个`Context`的实例。第二点暂时先不去纠结`compose`是什么，就是理解成把加工过的`ctx`和我们传的中间件放在一起，这样才能处理对话。至此，既处理了`ctx`又处理了中间件，短短两句话，整个流程就结束了....

```js

const compose = require('koa-compose');

module.exports = class Application extends Emitter {
    listen(...args) {
        // ...
        const server = http.createServer(this.callback());
        return server.listen(...args);
    }

    callback() {
        const fn = compose(this.middleware);

        if (!this.listenerCount('error')) this.on('error', this.onerror);

        const handleRequest = (req, res) => {
            const ctx = this.createContext(req, res);
            return this.handleRequest(ctx, fn);
        };

        return handleRequest;
    }
}
```

当然再想挖一点也没完，首先中间件都是通过`app.use`传进去的，那我们看看`use`做了什么

```js
module.exports = class Application extends Emitter {
    constructor() {
        super();

        this.middleware = [];
        //...
    }
    
    use(fn) {
        // ... 错误边界处理

        // app.use就是放进了一个数组里
        this.middleware.push(fn);
        return this;
    }

    callback() {
        const fn = compose(this.middleware);
        // ...
    }
}
```

其实`use`只是把所有的`fn`放在了一个数组里，而通过`compose`把这个数组处理后返回来，那我们深入看下这个看着熟又不熟的“老熟人” [compose](https://github.com/koajs/compose/blob/master/index.js)

`compose`是一个典型的柯里化函数，返回的`fn`也是给中间件处理的函数，其中很重要的是数组的长度，和当前的`index`下标。 `compose`会把`middleware`数组里面的函数变成 `fn1(ctx, fn2(ctx, fn3() ) )`的格式，这里和`redux`那里有异曲同工。再说简单点，这里的`next`，其实就是用数组里的后一个函数替代，也就是下面一段例子

```js
fn1(ctx, ) {
    console.log(1)
    fn2(ctx, ) {
        console.log(2)
        fn3(ctx){
            console.log(3)
        }
        console.log(4)
    }
    console.log(5)
}
```

简单吧，那下面看下源码里面的实现

```js

function compose (middleware) {
    // this.middleware 是否是一个数组， 以及里面每一项是否是一个fn 的错误边界，
    if (!Array.isArray(middleware)) throw new TypeError('Middleware stack must be an array!')
    for (const fn of middleware) {
        if (typeof fn !== 'function') throw new TypeError('Middleware must be composed of functions!')
    }

    /**
    * @param {Object} context
    * @return {Promise}
    * @api public
    */
    // 这就是返回去的fn，也是传到requestHandler的fn
    return function (context, next) {
        // last called middleware #
        let index = -1
        return dispatch(0)
        function dispatch (i) {
            if (i <= index) return Promise.reject(new Error('next() called multiple times'))
            index = i
            let fn = middleware[i]
            // 当i等于数组的长度 -- 即数组里的每一项都走完了走到最后一项了，把fn赋值成next，毕竟洋葱模型是要返回来一次。这里我说的不够明确可以看顶部的qianlongo大神的细节
            if (i === middleware.length) fn = next
            // 最后一个为空的next时候还是要返回一个Promise，这个是关键，才能保证后面的继续调用
            // 换句话也可以说，所有的都需要包装成一个Promise
            // 由于 fn1(ctx, fn2(ctx, fn3() ) )以后 只需要把最后一个用promise包一下，就会自然回流形成洋葱模型
            // 也就是fn3执行完了以后继续执行fn2未执行完的部分，这里和redux那里有异曲同工
            if (!fn) return Promise.resolve()
            try {
                // 每一个中间件执行在ctx环境下，同时递归调用下一个下标的fn
                // 也就是遇到next()以后就开始执行下一个中间件了。
                return Promise.resolve(fn(context, dispatch.bind(null, i + 1)));
            } catch (err) {
                return Promise.reject(err)
            }
        }
    }
}
```

可以看到`compose`的源码就是`koa`中间件“洋葱模型”的原因。通过`next`来控制中间件的下流和回流。那么就剩下一个`handleRequest`方法的源码需要看了

```js
module.exports = class Application extends Emitter {
    callback() {
        const fn = compose(this.middleware);

        const handleRequest = (req, res) => {
            const ctx = this.createContext(req, res);
            return this.handleRequest(ctx, fn);
        };

        return handleRequest;
    }

    handleRequest(ctx, fnMiddleware) {
        // 传进来的ctx已经是被挂在后的实例
        const res = ctx.res;
        res.statusCode = 404;
        const onerror = err => ctx.onerror(err);
        // respond 理解成一个处理ctx.respond的工具函数
        const handleResponse = () => respond(ctx);
        // onFineshed应该是一个处理完成的钩子
        onFinished(res, onerror);
        // 最后返回的是compose后的中间件“合集”处理ctx以后的一个Promise
        return fnMiddleware(ctx).then(handleResponse).catch(onerror);
    }
}


/**
 * Response helper.
 */

function respond(ctx) {
    // allow bypassing koa
    if (false === ctx.respond) return;

    const res = ctx.res;
    if (!ctx.writable) return;

    let body = ctx.body;
    const code = ctx.status;

    // ignore body
    if (statuses.empty[code]) {
        // strip headers
        ctx.body = null;
        return res.end();
    }

    if ('HEAD' == ctx.method) {
        if (!res.headersSent && isJSON(body)) {
        ctx.length = Buffer.byteLength(JSON.stringify(body));
        }
        return res.end();
    }

    // status body
    if (null == body) {
        if (ctx.req.httpVersionMajor >= 2) {
        body = String(code);
        } else {
        body = ctx.message || String(code);
        }
        if (!res.headersSent) {
        ctx.type = 'text';
        ctx.length = Buffer.byteLength(body);
        }
        return res.end(body);
    }

    // responses
    if (Buffer.isBuffer(body)) return res.end(body);
    if ('string' == typeof body) return res.end(body);
    if (body instanceof Stream) return body.pipe(res);

    // body: json
    body = JSON.stringify(body);
    if (!res.headersSent) {
        ctx.length = Buffer.byteLength(body);
    }
    res.end(body);
}
```

至此，大概的流程就这么多了，最核心的主要是在`compose`那部分对中间件的处理。同时，这里可以平行的和`redux`里对`applyMiddleware`里面的`compose`进行横向对比

最后放上`koa` `application.js`的源码片段

```js
const onFinished = require('on-finished');
const response = require('./response');
const compose = require('koa-compose');

module.exports = class Application extends Emitter {
    constructor() {
        super();

        this.middleware = [];
        //...
    }

    use(fn) {
        // ... 错误边界处理

        // app.use就是放进了一个数组里
        this.middleware.push(fn);
        return this;
    }

    callback() {
        const fn = compose(this.middleware);

        if (!this.listenerCount('error')) this.on('error', this.onerror);

        const handleRequest = (req, res) => {
            const ctx = this.createContext(req, res);
            return this.handleRequest(ctx, fn);
        };

        return handleRequest;
    }

    handleRequest(ctx, fnMiddleware) {
        const res = ctx.res;
        res.statusCode = 404;
        const onerror = err => ctx.onerror(err);
        const handleResponse = () => respond(ctx);
        onFinished(res, onerror);
        return fnMiddleware(ctx).then(handleResponse).catch(onerror);
    }

    listen(...args) {
        debug('listen');
        const server = http.createServer(this.callback());
        return server.listen(...args);
    }
}
```


compose源码



```js
'use strict'

/**
 * Expose compositor.
 */

module.exports = compose

/**
 * Compose `middleware` returning
 * a fully valid middleware comprised
 * of all those which are passed.
 *
 * @param {Array} middleware
 * @return {Function}
 * @api public
 */

function compose (middleware) {
    if (!Array.isArray(middleware)) throw new TypeError('Middleware stack must be an array!')
    for (const fn of middleware) {
        if (typeof fn !== 'function') throw new TypeError('Middleware must be composed of functions!')
    }

    /**
    * @param {Object} context
    * @return {Promise}
    * @api public
    */

    return function (context, next) {
        // last called middleware #
        let index = -1
        return dispatch(0)
        function dispatch (i) {
            if (i <= index) return Promise.reject(new Error('next() called multiple times'))
            index = i
            let fn = middleware[i]
            if (i === middleware.length) fn = next
            if (!fn) return Promise.resolve()
            try {
                return Promise.resolve(fn(context, dispatch.bind(null, i + 1)));
            } catch (err) {
                return Promise.reject(err)
            }
        }
    }
}
```