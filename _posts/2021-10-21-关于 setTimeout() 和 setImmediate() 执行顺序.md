# 关于 setTimeout() 和 setImmediate() 执行顺序

### 从一段示例代码引发的思考

官方文档给出的一段示例代码及预期可能的结果：

>```js
>// timeout_vs_immediate.js
>// this script is not within an I/O cycle (i.e. the main module)
>setTimeout(() => {
>  console.log('timeout');
>}, 0);
>
>setImmediate(() => {
>  console.log('immediate');
>});
>```

>因为受进程性能的约束，两个定时器的回调函数执行顺序是不确定的：
>
>```shell
>$ node timeout_vs_immediate.js
>timeout
>immediate
>
>$ node timeout_vs_immediate.js
>immediate
>timeout
>```

根据文档描述的事件循环，这段位于主模块的代码在注册好两个定时函数的回调函数后，会进入事件循环。然后先进入 `Timers` 阶段处理 `setTimeout()` 的回调后，才会进入 `Check` 阶段处理 `setImmediate()` 的回调函数。

但是以上分析和预期结果并不一致，而官方文档说是和进程的性能有关，于是有了下面的猜想和实验。

### `setImmediate()` 对比 `setTimeout()`

Ref: [The Node.js Event Loop, Timers, and process.nextTick()](https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick/)

`setImmediate()` 和 `setTimeout()` 是相似的，但根据调用它们的时间而以不同的方式表现。

- `setImmediate()`旨在在当前 **poll** 阶段完成后执行脚本。
- `setTimeout()` 安排脚本在经过以毫秒为单位的最小阈值后运行。

### 从 `setTimeout()` 入手

Ref: [node](https://github.com/nodejs/node)

进入 `setTimeout()` 源码实现部分：

> ```js
> // node/lib/timers.js 
> function setTimeout(callback, after, arg1, arg2, arg3) {
>  validateCallback(callback);
> 
>  let i, args;
>  switch (arguments.length) {
>    // fast cases
>    case 1:
>    case 2:
>      break;
>    case 3:
>      args = [arg1];
>      break;
>    case 4:
>      args = [arg1, arg2];
>      break;
>    default:
>      args = [arg1, arg2, arg3];
>      for (i = 5; i < arguments.length; i++) {
>        // Extend array dynamically, makes .apply run much faster in v6.0.0
>        args[i - 2] = arguments[i];
>      }
>      break;
>  }
> 
>  const timeout = new Timeout(callback, after, args, false, true);
>  insert(timeout, timeout._idleTimeout);
> 
>  return timeout;
> }
> ```

> ```js
> // node/lib/internal/timers.js
> function Timeout(callback, after, args, isRepeat, isRefed) {
>  after *= 1; // Coalesce to number or NaN
>  if (!(after >= 1 && after <= TIMEOUT_MAX)) {
>    if (after > TIMEOUT_MAX) {
>      process.emitWarning(`${after} does not fit into` +
>                          ' a 32-bit signed integer.' +
>                          '\nTimeout duration was set to 1.',
>                          'TimeoutOverflowWarning');
>    }
>    after = 1; // Schedule on next tick, follows browser behavior
>  }
> 
>  this._idleTimeout = after;
>  this._idlePrev = this;
>  this._idleNext = this;
>  this._idleStart = null;
>  // This must be set to null first to avoid function tracking
>  // on the hidden class, revisit in V8 versions after 6.2
>  this._onTimeout = null;
>  this._onTimeout = callback;
>  this._timerArgs = args;
>  this._repeat = isRepeat ? after : null;
>  this._destroyed = false;
> 
>  if (isRefed)
>    incRefCount();
>  this[kRefed] = isRefed;
>  this[kHasPrimitive] = false;
> 
>  initAsyncResource(this, 'Timeout');
> }
> ```

从源码实现可以发现， `setTimeout(fn, timeout)` 的 `timeout` 参数的下限是 **1**，即 `setTimeout(fn, 0)` 和 `setTimeout(fn, 1)` 其实是等价的，也就是说，至少在 **1ms** 后，`setTimeout()` 设置的定时器才会被判定为超时。

由这条线索可以进行一个猜想：如果程序从注册 `setTimeout()` 定时器回调到进入事件循环 `Timers` 阶段所花费的时间不超过 **1ms**，那么就这个定时器的回调会在下一次的事件循环中才会（超时）被执行。在这种情况下，`setImmediate()` 的回调先于定时器的回调在本轮事件循环中执行。



### 验证猜想

假设以上的猜想正确，只要**延后程序进入事件循环的时间（至少 1ms ）**，就能使这段位于主模块的代码一定会先执行 `setTimeout()` 定时器的回调函数，然后再执行 `setImmediate()` 的回调函数。

测试代码及结果：

>```js
>// this script is not within an I/O cycle (i.e. the main module)
>setTimeout(() => {
>console.log('timeout');
>}, 0);
>
>setImmediate(() => {
>console.log('immediate');
>});
>
>console.log('now "timeout" should be first surely'); // waste some time
>```
>多次运行结果：
>
>```shell
>% node index.js
>now "timeout" should be first surely
>timeout
>immediate
>
>% node index.js
>now "timeout" should be first surely
>timeout
>immediate
>
>% node index.js
>now "timeout" should be first surely
>timeout
>immediate
>```

多次运行结果和预期一致。

