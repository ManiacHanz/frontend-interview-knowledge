# Readme

## 不定期收集一些常用中间件和简单解析在下

### 目录
* [cors](#跨域)
* [koa-logger](#logger)
* [koa-send](#koa-send)
* [koa-static](#koa-static)

------ 留位置给超链接--------

### 跨域
[cors](https://github.com/koajs/cors/blob/master/index.js)

这个源码的注释已经很明显了，各个配置代表的request Header里面的几个配置项，这里增加一点`http`的请求头知识点。
[MDN地址](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Access_control_CORS)

* Access-Control-Allow-Methods  表明服务器允许客户端使用哪些方法请求 POST GET？
* Access-Control-Allow-Headers  表明服务器允许请求中携带字段`X-PINGOTHER` 与 `Content-Type`。
* Access-Control-Allow-Origin   表明服务器允许哪些域访问，设置成`*`表明可以被任意外域访问
* Access-Control-Expose-Headers 跨域访问时浏览器的XHR对象的`getResponseHeader()`方法能访问的响应头是有限的，如果想访问其他的，则需要这个配置把其他的添加到白名单，用逗号连接。比如:`X-My-Custom-Header`, `X-Another-Custom-Header`
* Access-Control-Max-Age        指定了preflight请求的结果能够被缓存多久
* Access-Control-Allow-Credentials  响应头表示是否可以将对请求的响应暴露给页面。返回true则可以，其他值均不可以。

> 同时这里有一点，附带身份凭证的请求，前端方面必须把`Access-Control-Allow-Credentials`设置成`true`，才会把服务端返回的`cookie`等身份信息携带过去；与此同时，后端`Access-Control-Allow-Origin`的值不得设置为`"*"`，否则请求将会失败！！！

```js
/**
 * CORS middleware
 *
 * @param {Object} [options]
 *  - {String|Function(ctx)} origin `Access-Control-Allow-Origin`, default is request Origin header
 *  - {String|Array} allowMethods `Access-Control-Allow-Methods`, default is 'GET,HEAD,PUT,POST,DELETE,PATCH'
 *  - {String|Array} exposeHeaders `Access-Control-Expose-Headers`
 *  - {String|Array} allowHeaders `Access-Control-Allow-Headers`
 *  - {String|Number} maxAge `Access-Control-Max-Age` in seconds
 *  - {Boolean} credentials `Access-Control-Allow-Credentials`
 *  - {Boolean} keepHeadersOnError Add set headers to `err.header` if an error is thrown
 * @return {Function} cors middleware
 * @api public
 */
module.exports = function(options) {
  // 跨域需要验证request header里面的 Access-Control-Allow-Methods
  const defaults = {
    allowMethods: 'GET,HEAD,PUT,POST,DELETE,PATCH',
  };

  options = Object.assign({}, defaults, options);

  // getResponseHeader()能访问的响应头
  if (Array.isArray(options.exposeHeaders)) {
    options.exposeHeaders = options.exposeHeaders.join(',');
  }

  // 允许的访问方法
  if (Array.isArray(options.allowMethods)) {
    options.allowMethods = options.allowMethods.join(',');
  }

  // 允许携带的字段 携带字段`X-PINGOTHER` 与 `Content-Type`
  if (Array.isArray(options.allowHeaders)) {
    options.allowHeaders = options.allowHeaders.join(',');
  }

  // 预检的缓存时间
  if (options.maxAge) {
    options.maxAge = String(options.maxAge);
  }

  // 是否携带身份凭证，由于只有true才可以，所以这里转化成bool
  options.credentials = !!options.credentials;
  options.keepHeadersOnError = options.keepHeadersOnError === undefined || !!options.keepHeadersOnError;

  // 根据中间件的分析，这里的ctx就是 koa里面的ctx对象。而next的是实现洋葱模型的关键
  return function cors(ctx, next) {
    // If the Origin header is not present terminate this set of steps.
    // The request is outside the scope of this specification.
    // 获取 request.Origin
    // 携带身份信息的跨域请求不能把origin设置成* 所以要动态获取以后去动态set
    const requestOrigin = ctx.get('Origin');

    // Always set Vary header
    // https://github.com/rs/cors/issues/10
    // 设置 Origin
    ctx.vary('Origin');

    // 不合法的情况下直接执行下一个中间件
    if (!requestOrigin) {
      return next();
    }

    let origin;

    if (typeof options.origin === 'function') {
      // 这里控制options里的origin配置如果传入函数的话，参数是ctx，且必须有返回
      origin = options.origin(ctx);
      if (!origin) {
        return next();
      }
    } else {
      origin = options.origin || requestOrigin;
    }

    const headersSet = {};

    function set(key, value) {
      ctx.set(key, value);
      headersSet[key] = value;
    }

    // OPTIONS预检
    if (ctx.method !== 'OPTIONS') {
      // Simple Cross-Origin Request, Actual Request, and Redirects
      set('Access-Control-Allow-Origin', origin);

      if (options.credentials === true) {
        set('Access-Control-Allow-Credentials', 'true');
      }

      if (options.exposeHeaders) {
        set('Access-Control-Expose-Headers', options.exposeHeaders);
      }

      if (!options.keepHeadersOnError) {
        return next();
      }
      return next().catch(err => {
        err.headers = Object.assign({}, err.headers, headersSet);
        throw err;
      });
    } else {
      // Preflight Request

      // If there is no Access-Control-Request-Method header or if parsing failed,
      // do not set any additional headers and terminate this set of steps.
      // The request is outside the scope of this specification.
      if (!ctx.get('Access-Control-Request-Method')) {
        // this not preflight request, ignore it
        return next();
      }
      // 动态设置
      ctx.set('Access-Control-Allow-Origin', origin);

      if (options.credentials === true) {
        ctx.set('Access-Control-Allow-Credentials', 'true');
      }

      if (options.maxAge) {
        ctx.set('Access-Control-Max-Age', options.maxAge);
      }

      if (options.allowMethods) {
        ctx.set('Access-Control-Allow-Methods', options.allowMethods);
      }

      let allowHeaders = options.allowHeaders;
      if (!allowHeaders) {
        allowHeaders = ctx.get('Access-Control-Request-Headers');
      }
      if (allowHeaders) {
        ctx.set('Access-Control-Allow-Headers', allowHeaders);
      }

      ctx.status = 204;
    }
  };
};
```

[回到顶部](#readme)


### logger
koa-logger

> 用于打印一些常用日志的中间件

```js

const Counter = require('passthrough-counter')
const humanize = require('humanize-number')
const bytes = require('bytes')
const chalk = require('chalk')
const util = require('util')

/**
 * Expose logger.
 */

module.exports = dev

/**
 * Color map.
 */

const colorCodes = {
  7: 'magenta',
  5: 'red',
  4: 'yellow',
  3: 'cyan',
  2: 'green',
  1: 'green',
  0: 'yellow'
}

/**
 * Development logger.
 */

function dev (opts) {
  // print to console helper.
  // 在控制台打印的核心方法
  const print = (function () {
    // 这个地方通过柯里化节约性能，让transporter值被赋值一次
    let transporter
    if (typeof opts === 'function') {
      transporter = opts
    } else if (opts && opts.transporter) {
      transporter = opts.transporter
    }

    return function printFunc (...args) {
      let str = util.format(...args)
      if (transporter) {
        transporter(str, args)
      } else {
        console.log(...args)
      }
    }
  }())

  return async function logger (ctx, next) {
    // request
    const start = Date.now()
    print('  ' + chalk.gray('<--') +
      ' ' + chalk.bold('%s') +
      ' ' + chalk.gray('%s'),
        ctx.method,
        ctx.originalUrl)

    try {
      await next()
    } catch (err) {
      // log uncaught downstream errors
      log(print, ctx, start, null, err)
      throw err
    }

    // calculate the length of a streaming response
    // by intercepting the stream with a counter.
    // only necessary if a content-length header is currently not set.
    const length = ctx.response.length
    const body = ctx.body
    let counter
    // readable??
    if (length == null && body && body.readable) {
      // 使用流处理
      ctx.body = body
        .pipe(counter = Counter())
        .on('error', ctx.onerror)
    }

    // log when the response is finished or closed,
    // whichever happens first.
    const res = ctx.res

    // 清除监听 同时打印
    const onfinish = done.bind(null, 'finish')
    const onclose = done.bind(null, 'close')

    // 监听
    res.once('finish', onfinish)
    res.once('close', onclose)

    function done (event) {
      res.removeListener('finish', onfinish)
      res.removeListener('close', onclose)
      log(print, ctx, start, counter ? counter.length : length, null, event)
    }
  }
}

/**
 * Log helper. log 工具函数
 */

function log (print, ctx, start, len, err, event) {
  // get the status code of the response
  const status = err
    ? (err.isBoom ? err.output.statusCode : err.status || 500)
    : (ctx.status || 404)

  // set the color of the status code;
  // 根据返回值的首位区别
  const s = status / 100 | 0
  const color = colorCodes.hasOwnProperty(s) ? colorCodes[s] : 0

  // get the human readable response length
  let length
  if (~[204, 205, 304].indexOf(status)) {
    length = ''
  } else if (len == null) {
    length = '-'
  } else {
    length = bytes(len).toLowerCase()
  }

  const upstream = err ? chalk.red('xxx')
    : event === 'close' ? chalk.yellow('-x-')
    : chalk.gray('-->')

  print('  ' + upstream +
    ' ' + chalk.bold('%s') +
    ' ' + chalk.gray('%s') +
    ' ' + chalk[color]('%s') +
    ' ' + chalk.gray('%s') +
    ' ' + chalk.gray('%s'),
      ctx.method,
      ctx.originalUrl,
      status,
      time(start),
      length)
}

/**
 * Show the response time in a human readable format.
 * In milliseconds if less than 10 seconds,
 * in seconds otherwise.
 */

function time (start) {
  const delta = Date.now() - start
  return humanize(delta < 10000
    ? delta + 'ms'
    : Math.round(delta / 1000) + 's')
}
```

[回到顶部](#readme)

### 静态资源

koa-send

> Static file serving middleware.

[仓库地址](https://github.com/koajs/send)

*这个库是给`koa-static`打基础的库，用于找到并返回给浏览器静态文件*

我们只看核心代码，不看配置和排错处理
核心是`node`原生的`fs`，通过`fs.Stat`类获取到文件信息来满足自定义设置头部；格式化`path`获取到文件信息以后，通过`ctx.body = fs.createReadStream(path)`来返回给前台一个文件流

```js
async function send (ctx, path, opts = {}) { 
  // ... 配置处理， 排错处理
  let stats
  try {
    stats = await fs.stat(path)

    // Format the path to serve static file servers
    // and not require a trailing slash for directories,
    // so that you can do both `/directory` and `/directory/`
    if (stats.isDirectory()) {
      if (format && index) {
        path += '/' + index
        stats = await fs.stat(path)
      } else {
        return
      }
    }
  } catch (err) {
    const notfound = ['ENOENT', 'ENAMETOOLONG', 'ENOTDIR']
    if (notfound.includes(err.code)) {
      throw createError(404, err)
    }
    err.status = 500
    throw err
  }

  if (setHeaders) setHeaders(ctx.res, path, stats)

  // 流以及自定义的setHeader
  ctx.set('Content-Length', stats.size)
  if (!ctx.response.get('Last-Modified')) ctx.set('Last-Modified', stats.mtime.toUTCString())
  if (!ctx.response.get('Cache-Control')) {
    const directives = ['max-age=' + (maxage / 1000 | 0)]
    if (immutable) {
      directives.push('immutable')
    }
    ctx.set('Cache-Control', directives.join(','))
  }
  if (!ctx.type) ctx.type = type(path, encodingExt)
  // 核心逻辑就是这里
  ctx.body = fs.createReadStream(path)

  return path

}
```

[回到顶部](#readme)

koa-static

[回到顶部](#readme)

### 路由
koa-router

### 处理请求参数
koa-better-body

### 将中间件转换成koa2可以使用的中间件
koa-convert

### sesion
koa-session

### EJS模板使用
koa-ejs

### mysql
mysql-pro