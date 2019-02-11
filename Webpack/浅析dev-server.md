
## 问题： webpack-dev-server 是怎么跑起来的

[git仓库](https://github.com/webpack/webpack-dev-server)

*才疏学浅，抛砖引玉*


> 核心暴露的是一个`Server`构造函数，而构造函数的内部核心就是一个`express`服务器。通过`express`的各种api，分配路由以及调用中间件

下面在源码里做简要分析，完整代码请去git仓库查看

```js
const fs = require('fs');
const path = require('path');

const ip = require('ip');
const tls = require('tls');
const url = require('url');
const http = require('http');
const https = require('https');
const spdy = require('spdy');
const sockjs = require('sockjs');

const semver = require('semver');

const killable = require('killable');

const del = require('del');
const chokidar = require('chokidar');

const express = require('express');

const compress = require('compression');
const serveIndex = require('serve-index');
const httpProxyMiddleware = require('http-proxy-middleware');
const historyApiFallback = require('connect-history-api-fallback');

const webpack = require('webpack');
const webpackDevMiddleware = require('webpack-dev-middleware');

const createLogger = require('./utils/createLogger');
const createCertificate = require('./utils/createCertificate');

const validateOptions = require('schema-utils');
const schema = require('./options.json');

// 第一段是处理sockjs : sockjs移除了Origin header，但是dev-server有个白名单设置会检查Origin header
// 所以这里有一点处理
{
  const SockjsSession = require('sockjs/lib/transport').Session;
  const decorateConnection = SockjsSession.prototype.decorateConnection;
  SockjsSession.prototype.decorateConnection = function(req) {
    decorateConnection.call(this, req);
    const connection = this.connection;
    if (
      connection.headers &&
      !('origin' in connection.headers) &&
      'origin' in req.headers
    ) {
      connection.headers.origin = req.headers.origin;
    }
  };
}


const STATS = {
  all: false,
  hash: true,
  assets: true,
  warnings: true,
  errors: true,
  errorDetails: false,
};

// 要暴露的核心构造函数
// compiler就是 webpack()返回的核心对象
function Server(compiler, options = {}, _log) {
  
  // 这里挂载了一些属性

  if (this.progress) {
    const progressPlugin = new webpack.ProgressPlugin(
      (percent, msg, addInfo) => {
        percent = Math.floor(percent * 100);

        if (percent === 100) {
          msg = 'Compilation completed';
        }

        if (addInfo) {
          msg = `${msg} (${addInfo})`;
        }

        this.sockWrite(this.sockets, 'progress-update', { percent, msg });
      }
    );

    progressPlugin.apply(compiler);
  }

  // https://webpack.docschina.org/api/compiler-hooks/
  // 这里是加载 Plugin   
  // .tap 是生命周期钩子函数
  const addHooks = (compiler) => {
    const { compile, invalid, done } = compiler.hooks;

    compile.tap('webpack-dev-server', invalidPlugin);
    invalid.tap('webpack-dev-server', invalidPlugin);
    done.tap('webpack-dev-server', (stats) => {
      this._sendStats(this.sockets, stats.toJson(STATS));
      this._stats = stats;
    });
  };
  // 祖册插件
  if (compiler.compilers) {
    compiler.compilers.forEach(addHooks);
  } else {
    addHooks(compiler);
  }

  // app就是Server的核心，其实就是一个初始化的 express服务器
  const app = (this.app = new express());

  // wasm 是 WebAssembly 的简称 。不了解不过多解释
  express.static.mime.types.wasm = 'application/wasm';

  // 所有的请求都要先检查是否在白名单中。这里检查的就是 dev-server里有个allowhost的数组里的域
  app.all('*', (req, res, next) => {
    if (this.checkHost(req.headers)) {
      return next();
    }

    res.send('Invalid Host header');
  });

  const wdmOptions = { logLevel: this.log.options.level };

  // middleware for serving webpack bundle
  this.middleware = webpackDevMiddleware(
    compiler,
    Object.assign({}, options, wdmOptions)
  );
  
  // 下面都湿返回一些文件流
  app.get('/__webpack_dev_server__/live.bundle.js', (req, res) => {
    res.setHeader('Content-Type', 'application/javascript');

    fs.createReadStream(
      path.join(__dirname, '..', 'client', 'live.bundle.js')
    ).pipe(res);
  });

  app.get('/__webpack_dev_server__/sockjs.bundle.js', (req, res) => {
    //...
  });

  app.get('/webpack-dev-server.js', (req, res) => {
    // ...
  });

  app.get('/webpack-dev-server/*', (req, res) => {
    // ...
  });

  // 直接访问这个地址可以返回一个文件列表的流
  app.get('/webpack-dev-server', (req, res) => {
    res.setHeader('Content-Type', 'text/html');

    res.write(
      '<!DOCTYPE html><html><head><meta charset="utf-8"/></head><body>'
    );

    const outputPath = this.middleware.getFilenameFromUrl(
      options.publicPath || '/'
    );

    const filesystem = this.middleware.fileSystem;

    function writeDirectory(baseUrl, basePath) {
      const content = filesystem.readdirSync(basePath);

      res.write('<ul>');

      content.forEach((item) => {
        const p = `${basePath}/${item}`;

        if (filesystem.statSync(p).isFile()) {
          res.write('<li><a href="');
          res.write(baseUrl + item);
          res.write('">');
          res.write(item);
          res.write('</a></li>');

          if (/\.js$/.test(item)) {
            const html = item.substr(0, item.length - 3);

            res.write('<li><a href="');
            res.write(baseUrl + html);
            res.write('">');
            res.write(html);
            res.write('</a> (magic html for ');
            res.write(item);
            res.write(') (<a href="');
            res.write(
              baseUrl.replace(
                // eslint-disable-next-line
                /(^(https?:\/\/[^\/]+)?\/)/,
                '$1webpack-dev-server/'
              ) + html
            );
            res.write('">webpack-dev-server</a>)</li>');
          }
        } else {
          res.write('<li>');
          res.write(item);
          res.write('<br>');

          writeDirectory(`${baseUrl + item}/`, p);

          res.write('</li>');
        }
      });

      res.write('</ul>');
    }

    writeDirectory(options.publicPath || '/', outputPath);

    res.end('</body></html>');
  });


  // process.cwd() 返回nodejs进程的当前工作目录
  let contentBase;

  if (options.contentBase !== undefined) {
    contentBase = options.contentBase;
  } else {
    contentBase = process.cwd();
  }

  // Keep track of websocket proxies for external websocket upgrade.
  const websocketProxies = [];

  // 一些功能的配置
  const features = {
    // 压缩
    compress: () => {
      if (options.compress) {
        app.use(compress());
      }
    },
    // 对proxy的一些处理 
    // 对 对象 函数 数组的写法做了兼容处理，具体可以看仓库里的注释
    proxy: () => {
      if (options.proxy) {
        if (!Array.isArray(options.proxy)) {
          options.proxy = Object.keys(options.proxy).map((context) => {
            let proxyOptions;
            // For backwards compatibility reasons.
            const correctedContext = context
              .replace(/^\*$/, '**')
              .replace(/\/\*$/, '');

            if (typeof options.proxy[context] === 'string') {
              proxyOptions = {
                context: correctedContext,
                target: options.proxy[context],
              };
            } else {
              proxyOptions = Object.assign({}, options.proxy[context]);
              proxyOptions.context = correctedContext;
            }

            proxyOptions.logLevel = proxyOptions.logLevel || 'warn';

            return proxyOptions;
          });
        }

        const getProxyMiddleware = (proxyConfig) => {
          const context = proxyConfig.context || proxyConfig.path;
          if (proxyConfig.target) {
            // http-proxy-middleware 是代理的核心node module
            return httpProxyMiddleware(context, proxyConfig);
          }
        };
        
        options.proxy.forEach((proxyConfigOrCallback) => {
          let proxyConfig;
          let proxyMiddleware;

          if (typeof proxyConfigOrCallback === 'function') {
            proxyConfig = proxyConfigOrCallback();
          } else {
            proxyConfig = proxyConfigOrCallback;
          }

          proxyMiddleware = getProxyMiddleware(proxyConfig);

          if (proxyConfig.ws) {
            websocketProxies.push(proxyMiddleware);
          }

          app.use((req, res, next) => {
            if (typeof proxyConfigOrCallback === 'function') {
              const newProxyConfig = proxyConfigOrCallback();

              if (newProxyConfig !== proxyConfig) {
                proxyConfig = newProxyConfig;
                proxyMiddleware = getProxyMiddleware(proxyConfig);
              }
            }


            // bypass 必须是一个函数。 可以用来跳过某些请求，使它不被代理。
            const bypass = typeof proxyConfig.bypass === 'function';

            const bypassUrl =
              (bypass && proxyConfig.bypass(req, res, proxyConfig)) || false;

            // 跳过 直接执行下一个中间件
            if (bypassUrl) {
              req.url = bypassUrl;

              next();
            } else if (proxyMiddleware) {
              return proxyMiddleware(req, res, next);
            } else {
              next();
            }
          });
        });
      }
    },
    // 可以把404响应改成指定index.html的配置
    historyApiFallback: () => {
      if (options.historyApiFallback) {
        const fallback =
          typeof options.historyApiFallback === 'object'
            ? options.historyApiFallback
            : null;
        // Fall back to /index.html if nothing else matches.
        app.use(historyApiFallback(fallback));
      }
    },
    // 开发服务器的静态资源目录，用的比较少，这里不解析
    // 简单说明一点，不能用http 或者https 做远程静态资源解析，这个功能下个版本会被去掉，可以用proxy来替代
    contentBaseFiles: () => {
      // .. 大体还是使用app.get() 通过配置的参数拼一些地址，具体看仓库
    },
    contentBaseIndex: () => {
      
    },
    // 设置监听本地的文件改变  不支持远程
    watchContentBase: () => {
      // 调用的this._watch()
    },
    // 所有中间件执行前的钩子函数
    before: () => {
      if (typeof options.before === 'function') {
        options.before(app, this);
      }
    },
    middleware: () => {
      app.use(this.middleware);
    },
    // 所有中间件执行后的钩子函数
    after: () => {
      if (typeof options.after === 'function') {
        options.after(app, this);
      }
    },
    headers: () => {
      app.all('*', this.setContentHeaders.bind(this));
    },
    magicHtml: () => {
      app.get('*', this.serveMagicHtml.bind(this));
    },
    // setup已经废弃
  };

  // default 和 options 的一些合并
  const defaultFeatures = ['setup', 'before', 'headers', 'middleware'];

  if (options.proxy) {
    defaultFeatures.push('proxy', 'middleware');
  }

  if (contentBase !== false) {
    defaultFeatures.push('contentBaseFiles');
  }

  if (options.watchContentBase) {
    defaultFeatures.push('watchContentBase');
  }

  if (options.historyApiFallback) {
    defaultFeatures.push('historyApiFallback', 'middleware');

    if (contentBase !== false) {
      defaultFeatures.push('contentBaseFiles');
    }
  }

  defaultFeatures.push('magicHtml');

  if (contentBase !== false) {
    defaultFeatures.push('contentBaseIndex');
  }
  // compress is placed last and uses unshift so that it will be the first middleware used
  if (options.compress) {
    defaultFeatures.unshift('compress');
  }

  if (options.after) {
    defaultFeatures.push('after');
  }

  (options.features || defaultFeatures).forEach((feature) => {
    features[feature]();
  });

  // 服务器使用https
  if (options.https) {
    // for keep supporting CLI parameters
    if (typeof options.https === 'boolean') {
      options.https = {
        ca: options.ca,
        pfx: options.pfx,
        key: options.key,
        cert: options.cert,
        passphrase: options.pfxPassphrase,
        requestCert: options.requestCert || false,
      };
    }

    for (const property of ['ca', 'pfx', 'key', 'cert']) {
      const value = options.https[property];
      const isBuffer = value instanceof Buffer;

      if (value && !isBuffer && fs.lstatSync(value).isFile()) {
        options.https[property] = fs.readFileSync(path.resolve(value));
      }
    }

    let fakeCert;

    if (!options.https.key || !options.https.cert) {
      // Use a self-signed certificate if no certificate was configured.
      // Cycle certs every 24 hours
      const certPath = path.join(__dirname, '../ssl/server.pem');
      // 看路径是否存在  主要是检查 https的 自定义签名，挣输之类
      let certExists = fs.existsSync(certPath);

      if (certExists) {
        const certTtl = 1000 * 60 * 60 * 24;
        const certStat = fs.statSync(certPath);

        const now = new Date();

        // cert is more than 30 days old, kill it with fire
        if ((now - certStat.ctime) / certTtl > 30) {
          this.log.info('SSL Certificate is more than 30 days old. Removing.');

          del.sync([certPath], { force: true });

          certExists = false;
        }
      }
      // webpack-dev-server默认自带一个证书
      if (!certExists) {
        this.log.info('Generating SSL Certificate');

        const attrs = [{ name: 'commonName', value: 'localhost' }];

        const pems = createCertificate(attrs);

        fs.writeFileSync(certPath, pems.private + pems.cert, {
          encoding: 'utf-8',
        });
      }

      fakeCert = fs.readFileSync(certPath);
    }

    options.https.key = options.https.key || fakeCert;
    options.https.cert = options.https.cert || fakeCert;

    if (!options.https.spdy) {
      options.https.spdy = {
        protocols: ['h2', 'http/1.1'],
      };
    }

// 这里就是服务器跑起来的原因
    if (semver.gte(process.version, '10.0.0')) {
      this.listeningApp = https.createServer(options.https, app);
    } else {
      this.listeningApp = spdy.createServer(options.https, app);
    }
  } else {
    
    this.listeningApp = http.createServer(app);
  }

  // 用来追踪已经中断这个socket的包
  killable(this.listeningApp);


  websocketProxies.forEach(function(wsProxy) {
    this.listeningApp.on('upgrade', wsProxy.upgrade);
  }, this);
}

// dev-server的use其实也是用的 express的use
Server.prototype.use = function() {
  // eslint-disable-next-line
  this.app.use.apply(this.app, arguments);
};

// 同样用的express的setHeader
Server.prototype.setContentHeaders = function(req, res, next) {
  if (this.headers) {
    // eslint-disable-next-line
    for (const name in this.headers) {
      // eslint-disable-line
      res.setHeader(name, this.headers[name]);
    }
  }

  next();
};

// 都是对dev-server的host相关选项进行检查    里面专门对 localhost return了true。所以本地开发总能用localhost
Server.prototype.checkHost = function(headers) {
  return this.checkHeaders(headers, 'host');
};

Server.prototype.checkOrigin = function(headers) {
  return this.checkHeaders(headers, 'origin');
};

Server.prototype.checkHeaders = function(headers, headerToCheck) {
  // allow user to opt-out this security check, at own risk
  if (this.disableHostCheck) {
    return true;
  }

  if (!headerToCheck) headerToCheck = 'host';
  // get the Host header and extract hostname
  // we don't care about port not matching
  const hostHeader = headers[headerToCheck];

  if (!hostHeader) {
    return false;
  }

  // use the node url-parser to retrieve the hostname from the host-header.
  const hostname = url.parse(
    // if hostHeader doesn't have scheme, add // for parsing.
    /^(.+:)?\/\//.test(hostHeader) ? hostHeader : `//${hostHeader}`,
    false,
    true
  ).hostname;
  // always allow requests with explicit IPv4 or IPv6-address.
  // A note on IPv6 addresses:
  // hostHeader will always contain the brackets denoting
  // an IPv6-address in URLs,
  // these are removed from the hostname in url.parse(),
  // so we have the pure IPv6-address in hostname.
  if (ip.isV4Format(hostname) || ip.isV6Format(hostname)) {
    return true;
  }
  // always allow localhost host, for convience
  if (hostname === 'localhost') {
    return true;
  }
  // allow if hostname is in allowedHosts
  if (this.allowedHosts && this.allowedHosts.length) {
    for (let hostIdx = 0; hostIdx < this.allowedHosts.length; hostIdx++) {
      const allowedHost = this.allowedHosts[hostIdx];

      if (allowedHost === hostname) return true;

      // support "." as a subdomain wildcard
      // e.g. ".example.com" will allow "example.com", "www.example.com", "subdomain.example.com", etc
      if (allowedHost[0] === '.') {
        // "example.com"
        if (hostname === allowedHost.substring(1)) {
          return true;
        }
        // "*.example.com"
        if (hostname.endsWith(allowedHost)) {
          return true;
        }
      }
    }
  }

  // allow hostname of listening adress
  if (hostname === this.hostname) {
    return true;
  }

  // also allow public hostname if provided
  if (typeof this.publicHost === 'string') {
    const idxPublic = this.publicHost.indexOf(':');

    const publicHostname =
      idxPublic >= 0 ? this.publicHost.substr(0, idxPublic) : this.publicHost;

    if (hostname === publicHostname) {
      return true;
    }
  }

  // disallow
  return false;
};

// 等于app实例的listen
// http://nodejs.cn/api/net.html#net_server_listen
// 最后一个就是监听器
Server.prototype.listen = function(port, hostname, fn) {
  this.hostname = hostname;

  const returnValue = this.listeningApp.listen(port, hostname, (err) => {
    const socket = sockjs.createServer({
      // Use provided up-to-date sockjs-client
      sockjs_url: '/__webpack_dev_server__/sockjs.bundle.js',
      // Limit useless logs
      log: (severity, line) => {
        if (severity === 'error') {
          this.log.error(line);
        } else {
          this.log.debug(line);
        }
      },
    });

    socket.on('connection', (connection) => {
      if (!connection) {
        return;
      }

      if (
        !this.checkHost(connection.headers) ||
        !this.checkOrigin(connection.headers)
      ) {
        this.sockWrite([connection], 'error', 'Invalid Host/Origin header');

        connection.close();

        return;
      }

      this.sockets.push(connection);

      connection.on('close', () => {
        const idx = this.sockets.indexOf(connection);

        if (idx >= 0) {
          this.sockets.splice(idx, 1);
        }
      });

      if (this.hot) {
        this.sockWrite([connection], 'hot');
      }

      if (this.progress) {
        this.sockWrite([connection], 'progress', this.progress);
      }

      if (this.clientOverlay) {
        this.sockWrite([connection], 'overlay', this.clientOverlay);
      }

      if (this.clientLogLevel) {
        this.sockWrite([connection], 'log-level', this.clientLogLevel);
      }

      if (!this._stats) {
        return;
      }

      this._sendStats([connection], this._stats.toJson(STATS), true);
    });

    socket.installHandlers(this.listeningApp, {
      prefix: this.sockPath,
    });

    if (fn) {
      fn.call(this.listeningApp, err);
    }
  });

  return returnValue;
};

Server.prototype.close = function(cb) {
  this.sockets.forEach((socket) => {
    socket.close();
  });

  this.sockets = [];

  this.contentBaseWatchers.forEach((watcher) => {
    watcher.close();
  });

  this.contentBaseWatchers = [];
  // 这里的kill是使用killable包装了一下的api
  // https://www.npmjs.com/package/killable
  // node中 Server类本身没有kill方法 只有close方法
  this.listeningApp.kill(() => {
    this.middleware.close(cb);
  });
};

Server.prototype.sockWrite = function(sockets, type, data) {
  sockets.forEach((socket) => {
    socket.write(JSON.stringify({ type, data }));
  });
};

Server.prototype.serveMagicHtml = function(req, res, next) {
  const _path = req.path;

  try {
    const isFile = this.middleware.fileSystem
      .statSync(this.middleware.getFilenameFromUrl(`${_path}.js`))
      .isFile();

    if (!isFile) {
      return next();
    }
    // Serve a page that executes the javascript
    res.write(
      '<!DOCTYPE html><html><head><meta charset="utf-8"/></head><body><script type="text/javascript" charset="utf-8" src="'
    );
    res.write(_path);
    res.write('.js');
    res.write(req._parsedUrl.search || '');

    res.end('"></script></body></html>');
  } catch (err) {
    return next();
  }
};

// send stats to a socket or multiple sockets
Server.prototype._sendStats = function(sockets, stats, force) {
  if (
    !force &&
    stats &&
    (!stats.errors || stats.errors.length === 0) &&
    stats.assets &&
    stats.assets.every((asset) => !asset.emitted)
  ) {
    return this.sockWrite(sockets, 'still-ok');
  }

  this.sockWrite(sockets, 'hash', stats.hash);

  if (stats.errors.length > 0) {
    this.sockWrite(sockets, 'errors', stats.errors);
  } else if (stats.warnings.length > 0) {
    this.sockWrite(sockets, 'warnings', stats.warnings);
  } else {
    this.sockWrite(sockets, 'ok');
  }
};

Server.prototype._watch = function(watchPath) {
  // duplicate the same massaging of options that watchpack performs
  // https://github.com/webpack/watchpack/blob/master/lib/DirectoryWatcher.js#L49
  // this isn't an elegant solution, but we'll improve it in the future
  const usePolling = this.watchOptions.poll ? true : undefined;
  const interval =
    typeof this.watchOptions.poll === 'number'
      ? this.watchOptions.poll
      : undefined;

  const options = {
    ignoreInitial: true,
    persistent: true,
    followSymlinks: false,
    depth: 0,
    atomic: false,
    alwaysStat: true,
    ignorePermissionErrors: true,
    ignored: this.watchOptions.ignored,
    usePolling,
    interval,
  };

  const watcher = chokidar.watch(watchPath, options);

  watcher.on('change', () => {
    this.sockWrite(this.sockets, 'content-changed');
  });

  this.contentBaseWatchers.push(watcher);
};

Server.prototype.invalidate = function() {
  if (this.middleware) {
    this.middleware.invalidate();
  }
};

// Export this logic,
// so that other implementations,
// like task-runners can use it
Server.addDevServerEntrypoints = require('./utils/addEntries');

module.exports = Server;

```