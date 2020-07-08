### 一、 koa是什么
------------
koa是一个精简的node框架，它主要做了以下事情：
* 基于node原生req和res为request和response对象赋能，并基于它们封装成一个context对象。
* 基于async/await（generator）的中间件洋葱模型机制。
  
### 1.1 一个koa小栗子
-------
```javascript
const Koa = require('koa');
const app = new Koa();

app.use(async ctx => {
  ctx.body = 'Hello World';
});

app.listen(3000);
```

### 二、初读koa源码
----------
koa源码比较简单，主要由四个文件构成<br/>
```
── lib <br/>
   ├── application.js  
   ├── context.js     
   ├── request.js                   
   └── response.js                 
```

这4个文件其实也对应了koa的4个对象：<br/>
```
── lib <br/>
   ├── new Koa()  
   ├── ctx       
   ├── ctx.req                   
   └── ctx.response                   
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
    // Object.create(xxx)方法创建一个新对象，使用现有的对象来提供新创建的对象的__proto__。即以xxx为原型创建一个新对象，实现对xxx的继承。
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
    //  通过convert 将generator升级到async/await
    if (isGeneratorFunction(fn)) {
      fn = convert(fn);
    }
    // 将中间件存到this.middleware数组当中
    this.middleware.push(fn);
    return this;
  }

  /**
   * Return a request handler callback
   * for node's native http server.
   */

  callback() {
    //   利用koa-compose实现中间件的洋葱模型,细节在第三章讲解
      const fn = compose(this.middleware);
        // 返回 (req, res) = {}这样结构的回调函数
    const handleRequest = (req, res) => {
        // 利用this.createContext封装代理req、res一些属性及方法的context（context Plus）。
        const ctx = this.createContext(req, res);
        // 调用app实例上的handleRequest
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
    // 错误处理
    const onerror = err => ctx.onerror(err);
    // 处理响应内容
    const handleResponse = () => respond(ctx);
    onFinished(res, onerror);
    // 执行中间件并处理响应内容，对错误进行捕获。
    return fnMiddleware(ctx).then(handleResponse).catch(onerror);
  }

  /**
   * Initialize a new context.
   *
   * @api private
   */

  createContext(req, res) {
    //  将request、response挂载到context上
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
```javascript
const util = require('util');
const createError = require('http-errors');
const httpAssert = require('http-assert');
const delegate = require('delegates');

const proto = module.exports = {
  // 省略了一些不甚重要的函数
  onerror(err) {
    // 触发application实例的error事件
    this.app.emit('error', err, this);
  },
};

/*
 在application.createContext函数中，
 被创建的context对象会挂载基于request.js实现的request对象和基于response.js实现的response对象。
 下面2个delegate的作用是让context对象代理request和response的部分属性和方法
*/
delegate(proto, 'response')
  .method('attachment')
  ...
  .access('status')
  ...
  .getter('writable')
  ...;

delegate(proto, 'request')
  .method('acceptsLanguages')
  ...
  .access('querystring')
  ...
  .getter('origin')
  ...;
```
-----------
### 2.3 request.js
-----------
```javacript
module.exports = {
  
  // 在application.js的createContext函数中，会把node原生的req作为request对象(即request.js封装的对象)的属性
  // request对象会基于req封装很多便利的属性和方法
  get header() {
    return this.req.headers;
  },

  set header(val) {
    this.req.headers = val;
  },

  // 省略了大量类似的工具属性和方法
};
```
### 2.4 response.js
-----------
```javascript
module.exports = {
  // 在application.js的createContext函数中，会把node原生的res作为response对象（即response.js封装的对象）的属性
  // response对象与request对象类似，基于res封装了一系列便利的属性和方法
  get body() {
    return this._body;
  },

  set body(val) {
    //   支持string/json/buffer/Stream类型....
  },
 
```
### 三、 koa-compose源码
-----------
```javascript
function compose (middleware) {

  return function (context, next) {
    // last called middleware #
    let index = -1
    // 第一次调用
    return dispatch(0)
    function dispatch (i) {
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
下面我们结合一个栗子来分析koa-compose源码
```javascript
const Koa = require('koa');
const app = new Koa()
app.use(async(ctx, next) => {
    console.log('middleware1 start')
    await next()
    console.log('middleware1 end')
})
app.use(async(ctx,next) => {
    console.log('middleware2 start' )
    await next()
    console.log('middleware2 end')
})
app.use(async ctx => ctx.body = 'hello world');

app.listen(3000)
```
  `dipsatch(0)`即为执行如下代码
```javascript
    Promise.resolve(async(ctx, next) => {
    console.log('middleware1 start')
    await next()
    console.log('middleware1 end')
}(context, dispatch.bind(null, i + 1))
```
* 第一步输出`middleware1 start`。
* 执行`await next()`，即`dispatch(1)`同样的，`dispatch(1)`同`dispatch(0)`类似，执行如下代码。
  
  ```javascript
  Promise.resolve(async(ctx, next) => {
    console.log('middleware2 start' )
    await next()
    console.log('middleware2 end')
  }(context, dispatch.bind(null, i + 1))
  ```
  * 第一步输出`middleware2 start`
  * 调用`await next()`,即`dispatch(2)`...

看到这里，我们应该大致明白了还有第三个，第四个...第n个中间件时，执行的顺序。

### 参考
> [Object.create](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Object/create)
> 
> [koa 官网](https://koa.bootcss.com/)
> 
> [koa源码指南](https://developers.weixin.qq.com/community/develop/article/doc/0000e4c9290bc069f3380e7645b813)
