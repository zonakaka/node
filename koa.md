### 一、 koa是什么
------------
koa是一个精简的node框架，它主要做了以下事情：
* 基于node原生req和res为request和response对象赋能，并基于它们封装成一个context对象。
* 基于async/await（generator）的中间件洋葱模型机制。
  
### 二、初读koa源码
----------
koa源码比较简单，主要由四个文件构成<br/>
```
── lib <br/>
   ├── application.js  <br/>
   ├── context.js       <br/>
   ├── request.js                    <br/>
   └── response.js                    <br/>
```

这4个文件其实也对应了koa的4个对象：<br/>
```
── lib <br/>
   ├── new Koa()  <br/>
   ├── ctx       <br/>
   ├── ctx.req                   <br/>
   └── ctx.response                   <br/>
```

接下来我们分别看下各个文件的源码。

### 2.1 application.js
-----------
部分不必要代码省略不表
```javascript
const onFinished = require('on-finished');
const response = require('./response');
const compose = require('koa-compose');
const context = require('./context');
const request = require('./request');
const Emitter = require('events');
const http = require('http');
const convert = require('koa-convert');

/**
 * Expose `Application` class.
 * Inherits from `Emitter.prototype`.
 */
module.exports = class Application extends Emitter {
  /**
   * Initialize a new `Application`.
   */
  constructor(options) {
    this.middleware = [];
    this.context = Object.create(context);
    this.request = Object.create(request);
    this.response = Object.create(response);
  }
  /**
   * Shorthand for:
   *    http.createServer(app.callback()).listen(...)
   */
  listen(...args) {
    debug('listen');
    const server = http.createServer(this.callback());
    // http.createServer((req, res) => {...})
    return server.listen(...args);
  }
  /**
   * Use the given middleware `fn`.
   * Old-style middleware will be converted.
   */

  use(fn) {
    if (isGeneratorFunction(fn)) {
      fn = convert(fn);
    }
    this.middleware.push(fn);
    return this;
  }

  /**
   * Return a request handler callback
   * for node's native http server.
   */

  callback() {
    const fn = compose(this.middleware);
    const handleRequest = (req, res) => {
      const ctx = this.createContext(req, res);
      return this.handleRequest(ctx, fn);
    };

    return handleRequest;
  }

  /**
   * Handle request in callback.
   */

  handleRequest(ctx, fnMiddleware) {
    const res = ctx.res;
    res.statusCode = 404;
    const onerror = err => ctx.onerror(err);
    const handleResponse = () => respond(ctx);
    onFinished(res, onerror);
    return fnMiddleware(ctx).then(handleResponse).catch(onerror);
  }

  /**
   * Initialize a new context.
   *
   * @api private
   */

  createContext(req, res) {
    const context = Object.create(this.context);
    const request = context.request = Object.create(this.request);
    const response = context.response = Object.create(this.response);
    return context;
  }

/**
 * Response helper.
 */
function respond(ctx) {
  let body = ctx.body;
  // body: [json, string, buffer, stream]
  res.end(body);
}
```
### 2.2 context.js
-----------
### 2.3 request.js
-----------
### 2.4 response.js
-----------
