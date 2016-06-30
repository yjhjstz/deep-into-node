
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
## 总结


## 参考
* http://node.alibaba-inc.com/post/async-error-handle-and-domain.html?spm=0.0.0.0.7r8vQ2
