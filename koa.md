

### 一、 koa是什么
-------------------
koa是一个精简的node框架，它主要做了以下事情：
* 基于node原生req和res为request和response对象赋能，并基于它们封装成一个context对象。
* 基于async/await（generator）的中间件洋葱模型机制。


``` javascript

'use strict';
module.exports = class Application extends Emitter {
  /**
   * Initialize a new `Application`.
   *
   * @api public
   */

  constructor(options) {
    super();
    
    this.middleware = [];
    this.context = Object.create(context);
    this.request = Object.create(request);
    this.response = Object.create(response);
  }

  // http.createServer(app.callback()).listen(...)的简写
  listen(...args) {
    debug('listen');
    const server = http.createServer(this.callback());
    return server.listen(...args);
  }


  // 以数组的形式保存通过 app.use(....)引入的中间件
  use(fn) {
    this.middleware.push(fn);
    return this;
  }

  
  // http.createServer()的回调
  callback() {
    const fn = compose(this.middleware);


    const handleRequest = (req, res) => {
    // 创建context
      const ctx = this.createContext(req, res);
      // 发送请求
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
    // 发送请求，中间件包装处理后的对象作为请求对象
    return fnMiddleware(ctx).then(handleResponse).catch(onerror);
  }

  // 初始化context
  createContext(req, res) {
    const context = Object.create(this.context);
    const request = context.request = Object.create(this.request);
    const response = context.response = Object.create(this.response);
    context.app = request.app = response.app = this;
    context.req = request.req = response.req = req;
    context.res = request.res = response.res = res;
    request.ctx = response.ctx = context;
    request.response = response;
    response.request = request;
    context.originalUrl = request.originalUrl = req.url;
    context.state = {};
    return context;
  }

  

/**
 * Response helper.
 */

function respond(ctx) {
  // allow bypassing koa
  if (false === ctx.respond) return;

  if (!ctx.writable) return;

  const res = ctx.res;
  let body = ctx.body;
  const code = ctx.status;
  //根据不同的body内容及类型对body进行处理...省略细节代码
  
  
  res.end(body);
}
module.exports.HttpError = HttpError;

```
