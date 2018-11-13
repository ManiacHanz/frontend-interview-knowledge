
# koa2中间件执行的原理浅析

**先感谢[qianlongo](https://github.com/qianlongo)对`koa`的中间件执行原理的解析的[分享](https://github.com/qianlongo/resume-native/issues/1)，结合着源码发现`koa2`已经放弃`Generator`改用`Async`了，所以这边也总结一下自己的理解，加深印象**

[仓库地址](https://github.com/koajs/koa)

> koa的代码不多，但是引用的模块真是多，水平有限，大家结合着自己的理解看

有关中间件的代码被放在了`application.js`的文件里。稍微看一下这边的代码

```js
const compose = require('koa-compose');

module.exports = class Application extends Emitter {
    constructor() {
        super();

        this.middleware = [];
        //...
    }

    use(fn) {
        // ... 错误边界处理
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
}
```