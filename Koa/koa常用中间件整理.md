# Readme

## 不定期收集一些常用中间件和简单解析在下


[cors](#跨域)

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

### 静态资源
koa-static

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