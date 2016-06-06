
## HTTP 1/2

回到我们之前的 「Hello World」例子, 短短数行即可。
```js
const http = require('http');
const hostname = '127.0.0.1';
const port = 1337;

http.createServer((req, res) => {
  res.writeHead(200, { 'Content-Type': 'text/plain' });
  res.end('Hello World\n');
}).listen(port, hostname, () => {
  console.log(`Server running at http://${hostname}:${port}/`);
});
```

因为 Node.js 把许多细节都已在源码中封装好了，主要代码在 lib/_http_*.js 这些文件中，现在就让我们照着上述代码，看看从一个 HTTP 请求的到来直到响应，Node.js 都为我们在源码层做了些什么。

### Server
在 Node.js 中，若要收到一个 HTTP 请求，首先需要创建一个 http.Server 类的实例，然后监听它的 request 事件。由于 HTTP 协议属于应用层，在下层的传输层通常使用的是 TCP 协议，所以 net.Server 类正是 http.Server 类的父类。

```js
// lib/_http_server.js
// ...

function Server(requestListener) {
  if (!(this instanceof Server)) return new Server(requestListener);
  net.Server.call(this, { allowHalfOpen: true });

  if (requestListener) {
    this.addListener('request', requestListener);
  }

  // ...
  this.addListener('connection', connectionListener);

  this.addListener('clientError', function(err, conn) {
    conn.destroy(err);
  });

  this.timeout = 2 * 60 * 1000;

  this._pendingResponseData = 0;
}
util.inherits(Server, net.Server);
```

`requestListener` 回调函数作为观察者，监听了 `request` 事件， 默认超时时间为2分钟。

而当连接建立时，观察者 connectionListener 处理 `connection` 事件。

这时，则需要一个 HTTP parser 来解析通过 TCP 传输过来的数据：

```js
// lib/_http_server.js
const parsers = common.parsers;
// ...

function connectionListener(socket) {
  // ...
  var parser = parsers.alloc();
  parser.reinitialize(HTTPParser.REQUEST);
  parser.socket = socket;
  socket.parser = parser;
  parser.incoming = null;
  // ...
}
```

#### HTTP Parser

值得一提的是，parser 是从一个“池”中获取的，这个“池”使用了一种叫做 freelist的数据结构。
为了尽可能的对 parser 进行重用，并避免了不断调用构造函数的消耗，且设有数量上限（http 模块中为 1000）。

HTTPParser 的实现目前由 C++绑定实现，具体参见 deps/http_parser 目录。但笔者这边拓展一下：

社区有过对 http_parser 实现性能的争论， 性能上 JS 实现的版本超越 C 的实现。

原因是多方面的：
* 去调了 C++ 绑定层。
* JS 实现，避免了 C 栈和 JS 堆栈的切换和参数拷贝。
* V8 JIT 对热点函数的优化。

即便有上述优势，社区目前还是没有合并，处于pending 状态，结合个人和社区观点：
* 并发请求会导致 garbage collection 频繁，触发GC 停顿。
* 可以作为第三方模块存在。

> pull request: https://github.com/nodejs/node/pull/1457/

这里的 parser 也是基于事件的，很符合 Node.js 的核心思想。
```js
// lib/_http_common.js
// ...
const binding = process.binding('http_parser');
const HTTPParser = binding.HTTPParser;
const FreeList = require('internal/freelist').FreeList;
// ...

var parsers = new FreeList('parsers', 1000, function() {
  var parser = new HTTPParser(HTTPParser.REQUEST);
  // ...
  parser[kOnHeaders] = parserOnHeaders;
  parser[kOnHeadersComplete] = parserOnHeadersComplete; 
  parser[kOnBody] = parserOnBody; 
  parser[kOnMessageComplete] = parserOnMessageComplete;
  parser[kOnExecute] = null; 

  return parser;
});
exports.parsers = parsers;

// lib/_http_server.js
// ...

function connectionListener(socket) {
  parser.onIncoming = parserOnIncoming;
}
```
所以一个完整的 HTTP 请求从接收到完全解析，会挨个经历 parser 上的如下事件监听器：

* parserOnHeaders：不断解析推入的请求头数据。
* parserOnHeadersComplete：请求头解析完毕，构造 header 对象，为请求体创建 http.IncomingMessage 实例。
* parserOnBody：不断解析推入的请求体数据。
* parserOnExecute：请求体解析完毕，检查解析是否报错，若报错，直接触发 clientError 事件。若请求为 CONNECT 方法，或带有 Upgrade 头，则直接触发 connect 或 upgrade 事件。
* parserOnIncoming：处理具体解析完毕的请求。

前面提到的 `request`事件到底是在哪里触发的呢？回到源码

```js
// lib/_http_server.js
// ...

function connectionListener(socket) {
  var outgoing = [];
  var incoming = [];
  // ...
  
  function parserOnIncoming(req, shouldKeepAlive) {
    incoming.push(req);
    // ...
    var res = new ServerResponse(req);
    
    if (socket._httpMessage) {
      outgoing.push(res);
    } else {
      res.assignSocket(socket);
    }
    
    res.on('finish', resOnFinish);
    function resOnFinish() {
      incoming.shift();
      // ...
      var m = outgoing.shift();
      if (m) {
        m.assignSocket(socket);
      }
    }
    // ...
    self.emit('request', req, res);
  }
}
```

我们注意到 2个队列，`incoming`和`outgoing`, 他们用于缓冲 IncomingMessage 实例和对应的 ServerResponse 实例。
通过 `IncomingMessage`实例构建相应的 `ServerResponse` 实例, 并且通过 `res.assignSocket(socket);` ，绑定了
三元组 `<req, res, socket>`。


最后，发送 request 事件，参数为req, res。回到 hello world 中，监听者拿到 req 和 res, 向 response 流中写入HTTP头
和内容发送出去。


### 总结
对象池也是内存池的一种衍生，需要在内存和性能方面折中考量。

上面只是梳理了一个主线，其他异常处理，安全等方面剖析后面的章节会一一解读。


### 参考 
* https://docs.google.com/document/d/1A3cxhZg2aktJeSGt0P-8KrA4WyG1c8LlPomQflaQs8s/edit










