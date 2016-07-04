
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

另外， domain 为了支持深层次的嵌套， 提供了 `Domain#enter` 和 `Domain#exit` 的 API。
先来看 `enter`的实现，
```js
Domain.prototype.enter = function() {
  if (this._disposed) return;

  // note that this might be a no-op, but we still need
  // to push it onto the stack so that we can pop it later.
  exports.active = process.domain = this;
  stack.push(this);
  _domain_flag[0] = stack.length;
};
```
设置当前活跃的 `domain`, 并且为了便于回溯，将当前的 `domain` 加入到队列的后面，更新栈的深度。

再看 `exit`实现，
```js
Domain.prototype.exit = function() {
  // skip disposed domains, as usual, but also don't do anything if this
  // domain is not on the stack.
  var index = stack.lastIndexOf(this);
  if (this._disposed || index === -1) return;

  // exit all domains until this one.
  stack.splice(index);
  _domain_flag[0] = stack.length;

  exports.active = stack[stack.length - 1];
  process.domain = exports.active;
};
```
相反的， 退出当前的 `domain`, 更新长度，设置当前活跃的 `domain`。

读者可能好奇，我并没有显式地调用 `enter`, `exit`, 而只是简单的创建了一个 domain, 怎么会达到这种效果？

读者可以看看 `AsyncWrap::MakeCallback()`, 每次C++ --> JS, 都会检查 domain, 如果使用，则会显式地调用他们。
其他地方读者可以自行寻找。

为了解决不在当前作用域的异常处理， Domain 也提供 `Domain#add` 和 `Domain#remove` 来增加 `emitter` 或者
`Timer`。


回到事件的根本， 什么时候触发domain的error事件？
```js
process._fatalException = function(er) {
      var caught;

      if (process.domain && process.domain._errorHandler)
        caught = process.domain._errorHandler(er) || caught;

      if (!caught)
        caught = process.emit('uncaughtException', er);

      // If someone handled it, then great.  otherwise, die in C++ land
      // since that means that we'll exit the process, emit the 'exit' event
      if (!caught) {
        try {
          if (!process._exiting) {
            process._exiting = true;
            process.emit('exit', 1);
          }
        } catch (er) {
          // nothing to be done about it at this point.
        }

      // if we handled an error, then make sure any ticks get processed
      } else {
        NativeModule.require('timers').setImmediate(process._tickCallback);
      }

      return caught;
    };
```

如果当前 process 使用了 domain, 也是就 `process.domain` 不为空，就调用 `_errorHandler` 来处理，
当前也存在没有处理的情况，职责链来到 process， process 则触发 `uncaughtException` 事件。
 

### 总结
domain很强大，但它只能捕获在其作用域范围内的异常。对于非预期的异常产生的时候， 我们最好让当前请求超时，然后让这个进程停止服务，之后重新启动。

但始终 Domain 在异常处理上有各种不完美，目前该模块处于即将废除阶段，取代他的可能是另一种机制。

详细讨论见：
> https://github.com/nodejs/node/issues/66

### 参考
* http://node.alibaba-inc.com/post/async-error-handle-and-domain.html?spm=0.0.0.0.7r8vQ2

