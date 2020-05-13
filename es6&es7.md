## es7
- 装饰器
  
  本质上是
  ```javascript
    Object.defineProperty(target, value, descriptor)
   ```
   target即为装饰器所要修饰的对象，即下方紧邻的函数，可以是类、类的属性或方法等。
   value为接收的参数，可选
   descriptor即为target的配置信息，可以获取`value/configurable/writable...ect`。可以通过操作该对象对target进行功能的包装和扩展
   参考 
   - [Object.defineProperty](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Object/defineProperty)
   -  [装饰器是什么](http://www.liuhaihua.cn/archives/115548.html)
