
## 异常处理与domain

### 异步异常捕获
由于node的回调异步特性，无法通过try catch来捕捉所有的异常：
```js
try {
  process.nextTick(function () {
    foo.bar();
  });
} catch (err) {
  //can not catch it
}
```
如果try catch能够捕获所有的异常，这样我们可以在代码出现一些非预期的错误时，能够记录下错误的同时，友好的给调用者返回一个500错误。可惜，try catch无法捕获异步中的异常。所以我们能做的只能是：
```js
app.get('/index', function (req, res) {
  // 业务逻辑  
});

process.on('uncaughtException', function (err) {
  logger.error(err);
});
```

这个时候，虽然我们可以记录下这个错误的日志，且进程也不会异常退出，但是我们是没有办法对发现错误的请求友好返回的，因为异常处理只返回给我们一个冷冰冰的 `error`, 脱离了上下文，我们只能够让它超时返回。

### domain
在node v0.8+版本的时候，发布了一个模块domain。这个模块做的就是try catch所无法做到的：捕捉异步回调中出现的异常。 于是乎，我们上面那个无奈的例子好像有了解决的方案：
```js
var domain = require('domain');

//引入一个domain的中间件，将每一个请求都包裹在一个独立的domain中
//domain来处理异常
app.use(function (req,res, next) {
  var d = domain.create();
  //监听domain的错误事件
  d.on('error', function (err) {
    logger.error(err);
    res.statusCode = 500;
    res.json({sucess:false, messag: '服务器异常'});
    d.dispose();
  });

  d.add(req);
  d.add(res);
  d.run(next);
});
```
`domain`虽然捕捉到了异常，但是还是由于异常而导致的堆栈丢失会导致内存泄漏，所以出现这种情况的时候还是需要重启这个进程的。

### Domain剖析
Domain 自身其实也是 Event 模块一个典型的应用， 它通过事件的方式来传递捕获的错误。

```js
inherits(Domain, EventEmitter);

function Domain() {
  EventEmitter.call(this);

  this.members = [];
}
```


 

### 总结
domain很强大，但它只能捕获在其作用域范围内的异常。对于非预期的异常产生的时候， 我们最好让当前请求超时，然后让这个进程停止服务，之后重新启动。

但始终 Domain 在异常处理上有各种不完美，目前该模块处于即将废除阶段，取代他的可能是另一种机制。

详细讨论见：
> https://github.com/nodejs/node/issues/66

### 参考
* http://node.alibaba-inc.com/post/async-error-handle-and-domain.html?spm=0.0.0.0.7r8vQ2

