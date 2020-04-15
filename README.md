# node
## co 源码解读
### 使用场景
异步概念： js是单线程，所谓单线程就是代码按着从上到下的顺序去执行。但是在单线程中如果存在耗时时间非常长的计算，线程将会发生阻塞，无法向下继续执行。为了解决这类问题。js针对这种场景设计了异步---即发送了一个请求后不会立即得结果，代码可以继续往下执行。待结果返回后通过回调的方式去处理。但是，当有相互依赖的异步调用存在时，会产生异步嵌套的问题。我们可以通过generator、async/await、promise等方法解决异步嵌套问题。

#generator
```
//通过function *(){}的方式定义一个迭代器
function *gen(){
  //yield 用来暂停迭代器，并将执行权交给func1()
   yield fun1()
   yield func2()
}
function fun1(){
  setTimeout(()=>return 1)
}
function fun2(){
  setTimeout(()=>return 2)
}
//创建一个迭代器
var g = gen()
//g.next()在上次暂停的地方继续执行。并将执行权交还给g即当前生成器
g.next() //{value:1,done:false}
g.next() // {value: 2, done:false}
g.next() //{value:undefined: done: true}
```
yield命令像一个切换器一样，控制执行权的切换。
generator遇到yield就暂停，将执行权交给yield 指定的对象，对象执行完毕后将执行权返回，由generator.next恢复执行，generator.next会在在上次暂停的地方继续执行。
generator的优点就是代码写法像同步写法，但是需要手动去控制流程的执行，非常不方便。

那么如何实现generator的自动回调呢？
```
var fn = thunk(origin)
function thunk(fn) {
  return function(){
    var args = Array.prototype.slice.call(arguments)
    return function(callback) {
       args.push(callback)
      return fn.apply(this,args )
    }
  }
}
function run(fn){
  var gen = fn()
  function next(){
    var result = gen.next()
    if (result.done) return
    result.value(next)
  }
}
function *origin(){
    var a = yield fun1()
    var b = yield fun2()
    return a+b 
}
function fun1(){
    setTimeout(()=>1)
}
function fun2(a){
    setTimeout(()=>a+1)
}
```

***
### 源码解读
1. 核心代码主要是这个co函数。实现了迭代器的自动迭代并将迭代器对象封装成Promise。
```
function co(gen) {
  var ctx = this;
  var args = slice.call(arguments, 1);
  return new Promise(function(resolve, reject) {
    // 若为迭代器第一次调用不会返回任何结果
    if (typeof gen === 'function') gen = gen.apply(ctx, args);
    //若是普通函数则包装为Promise并返回，不再向下执行。
    if (!gen || typeof gen.next !== 'function') return resolve(gen);
    // 重点，手动调用onFulfilled
    onFulfilled();
    //这个是重点
    function onFulfilled(res) {
    }
    //onRejected暂不讲了，和onFulfilled差不多
    function onRejected(err) {
    }
    //这个是重点
    function next(ret) {
    }
  });
}

```
2. onFulfilled代码解析
```
 /**
     * @param {Mixed} res
     * @return {Promise}
     * @api private
     */

    function onFulfilled(res) {
      var ret;
      try {
        调用迭代器gen的next方法，ret接收返回值
        ret = gen.next(res);
      } catch (e) {
        return reject(e);
      }
      //传递ret给co自定义的next方法。
      next(ret);
      return null;
    }
```
3. next 代码解析
```
function next(ret) {
    // 判断迭代器是否迭代结束，结束则调用resolve将ret.value传递下去（这也是递归的出口）
    if (ret.done) return resolve(ret.value);
    //toPromise方法将对象封装成Promise
    var value = toPromise.call(ctx, ret.value);
    //递归调用onFulfilled、onRejected，实现迭代器next的自调用。
    if (value && isPromise(value)) return value.then(onFulfilled, onRejected);
    return onRejected(new TypeError('You may only yield a function, promise, generator, array, or object, '
      + 'but the following object was passed: "' + String(ret.value) + '"'));
  }
```
* next()做了两件事：
  * 一个是对象的Promise化。
  * 一个是通过`value.then(onFulfilled)`将当前的中间值传递给then的第一个对象，即为onFulfilled。
  * onFulfilled在当前值的基础上继续调用gen.next()和next()方法。由onFulfilled->next()->onFulfilled实现了迭代器的自动执行。
  * 当迭代器遍历结束时，其结果格式应该为{value:undefined, done:true}，即`if (ret.done) return resolve(ret.value);` 此时遍历结束。

