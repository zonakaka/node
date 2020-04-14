# node
## co 源码解读
### 使用场景
co是基于generator实现的流式控制器，适用于nodejs和浏览器。它结合了promise共同解决异步嵌套问题。可以看做是```async/await```
前身。

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
一个是对象的Promise化。
一个是通过`value.then(onFulfilled)`将当前的中间值传递给then的第一个对象，即为onFulfilled。onFulfilled在当前值的基础上继续调用gen.next()和next()方法。
从而实现了递归，当迭代器遍历结束时，其结果格式应该为{value:undefined, done:true},即`if (ret.done) return resolve(ret.value);`，此时遍历结束。

