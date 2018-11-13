
# koa-body

> 常用的用来解析`POST`请求数据的中间件，如表单、文件上传等

### koa-bodyparser

这里先分析一小段`koa-bodyparser`的原理，[仓库地址](https://github.com/koajs/bodyparser)

**结论（仅供参考）：`bodyparser`的解析原理其实就是根据`request`里的`Content-type`关键字来调用不同的`parse`方法，常用的在默认配置项里，当然也可以通过`extendType`增加自定义格式。比如说`POST`请求里一个常见的`multipart/form-data`格式，用`bodyparser`显然就解析不出来**

*这里有一个概念是`rawbody`，找了半天也没找到。最后通过[npm仓库地址](https://www.npmjs.com/package/raw-body)找到了一段解释。*

> Gets the entire buffer of a stream either as a Buffer or a string. Validates the stream's length against an expected length and maximum limit. Ideal for parsing request bodies.


```js
// 作者也说了，这个是基于co-body
var parse = require('co-body');
var copy = require('copy-to');

function checkEnable(types, type) {
  return types.includes(type);
}


module.exports = function (opts) {

  opts = opts || {};
  var detectJSON = opts.detectJSON;
  var onerror = opts.onerror;

  // 如果不传的话就不会解析'text'
  // 这里是关键 解析就是通过request header的 Content-type 来分别解析
  var enableTypes = opts.enableTypes || ['json', 'form'];
  // 判断配置项里是否包含 'form , json, text'
  // 由这个决定解析的类型
  var enableForm = checkEnable(enableTypes, 'form');
  var enableJson = checkEnable(enableTypes, 'json');
  var enableText = checkEnable(enableTypes, 'text');
  // 可以自定义 检测JSON格式 和 错误捕捉
  opts.detectJSON = undefined;
  opts.onerror = undefined;

  
  // 这个是co-body的一个配置项，当设置为true的时候，co-body会返回一个包含两个属性的对象
  // { parsed: /* parsed value */, raw: /* raw body */} }
  opts.returnRawBody = true;

  // default json types
  var jsonTypes = [
    'application/json',
    'application/json-patch+json',
    'application/vnd.api+json',
    'application/csp-report',
  ];

  // default form types
  var formTypes = [
    'application/x-www-form-urlencoded',
  ];

  // default text types
  var textTypes = [
    'text/plain',
  ];

  var jsonOpts = formatOptions(opts, 'json');
  var formOpts = formatOptions(opts, 'form');
  var textOpts = formatOptions(opts, 'text');

  // 把后一个数字的每一项都推进前一项
  // 可以增加自定义的解析范围
  var extendTypes = opts.extendTypes || {};

  extendType(jsonTypes, extendTypes.json);
  extendType(formTypes, extendTypes.form);
  extendType(textTypes, extendTypes.text);

  // ...
};

function formatOptions(opts, type) {
  var res = {};
  copy(opts).to(res);
  res.limit = opts[type + 'Limit'];
  return res;
}

function extendType(original, extend) {
  if (extend) {
    if (!Array.isArray(extend)) {
      extend = [extend];
    }
    extend.forEach(function (extend) {
      original.push(extend);
    });
  }
}

```

核心部分

```js
module.exports = function (opts) {
  // ... 配置项

  // 知道koa中是app.use(bodyParse())用法，ctx是 koa包装的context实例，next是gen.next
  return async function bodyParser(ctx, next) {
    if (ctx.request.body !== undefined) return await next();
    // 可以选择的禁用body-parser
    if (ctx.disableBodyParser) return await next();
    try {
      // 核心，根据上面enableForm，enableJson，enableText变量使用co-body解析
      const res = await parseBody(ctx);
      // 给ctx.request赋body和rawbody值
      ctx.request.body = 'parsed' in res ? res.parsed : {};
      if (ctx.request.rawBody === undefined) ctx.request.rawBody = res.raw;
    } catch (err) {
      if (onerror) {
        onerror(err, ctx);
      } else {
        throw err;
      }
    }
    // 执行下一个中间件
    await next();
  };

  async function parseBody(ctx) {
    if (enableJson && ((detectJSON && detectJSON(ctx)) || ctx.request.is(jsonTypes))) {
      return await parse.json(ctx, jsonOpts);
    }
    if (enableForm && ctx.request.is(formTypes)) {
      return await parse.form(ctx, formOpts);
    }
    if (enableText && ctx.request.is(textTypes)) {
      return await parse.text(ctx, textOpts) || '';
    }
    return {};
  }
}
```



放上源码

```js

'use strict';

/**
 * Module dependencies.
 */

var parse = require('co-body');
var copy = require('copy-to');

/**
 * @param [Object] opts
 *   - {String} jsonLimit default '1mb'
 *   - {String} formLimit default '56kb'
 *   - {string} encoding default 'utf-8'
 *   - {Object} extendTypes
 */

module.exports = function (opts) {
  opts = opts || {};
  var detectJSON = opts.detectJSON;
  var onerror = opts.onerror;

  var enableTypes = opts.enableTypes || ['json', 'form'];
  var enableForm = checkEnable(enableTypes, 'form');
  var enableJson = checkEnable(enableTypes, 'json');
  var enableText = checkEnable(enableTypes, 'text');

  opts.detectJSON = undefined;
  opts.onerror = undefined;

  // force co-body return raw body
  opts.returnRawBody = true;

  // default json types
  var jsonTypes = [
    'application/json',
    'application/json-patch+json',
    'application/vnd.api+json',
    'application/csp-report',
  ];

  // default form types
  var formTypes = [
    'application/x-www-form-urlencoded',
  ];

  // default text types
  var textTypes = [
    'text/plain',
  ];

  var jsonOpts = formatOptions(opts, 'json');
  var formOpts = formatOptions(opts, 'form');
  var textOpts = formatOptions(opts, 'text');

  var extendTypes = opts.extendTypes || {};

  extendType(jsonTypes, extendTypes.json);
  extendType(formTypes, extendTypes.form);
  extendType(textTypes, extendTypes.text);

  return async function bodyParser(ctx, next) {
    if (ctx.request.body !== undefined) return await next();
    if (ctx.disableBodyParser) return await next();
    try {
      const res = await parseBody(ctx);
      ctx.request.body = 'parsed' in res ? res.parsed : {};
      if (ctx.request.rawBody === undefined) ctx.request.rawBody = res.raw;
    } catch (err) {
      if (onerror) {
        onerror(err, ctx);
      } else {
        throw err;
      }
    }
    await next();
  };

  async function parseBody(ctx) {
    if (enableJson && ((detectJSON && detectJSON(ctx)) || ctx.request.is(jsonTypes))) {
      return await parse.json(ctx, jsonOpts);
    }
    if (enableForm && ctx.request.is(formTypes)) {
      return await parse.form(ctx, formOpts);
    }
    if (enableText && ctx.request.is(textTypes)) {
      return await parse.text(ctx, textOpts) || '';
    }
    return {};
  }
};

function formatOptions(opts, type) {
  var res = {};
  copy(opts).to(res);
  res.limit = opts[type + 'Limit'];
  return res;
}

function extendType(original, extend) {
  if (extend) {
    if (!Array.isArray(extend)) {
      extend = [extend];
    }
    extend.forEach(function (extend) {
      original.push(extend);
    });
  }
}

function checkEnable(types, type) {
  return types.includes(type);
}

```