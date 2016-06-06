## HTTP 2/2

http 模块提供了两个函数 http.request 和 http.get，功能是作为客户端向 HTTP服务器发起请求。

这通常来实现自己的爬虫程序， 笔者自己写的一个爬取知乎的一个例子：https://github.com/yjhjstz/iZhihu


### GET 例子
```js
const http = require("http")
http.get('http://www.baidu.com', (res) => {
  console.log(`Got response: ${res.statusCode}`);
  // consume response body
  res.resume();
}).on('error', (e) => {
  console.log(`Got error: ${e.message}`);
});
```
上面的程序会返回一个200的状态码！

### HTTP Client
Node.js 中，http.get 通过创建一个 `ClientRequest`的对象，建立与服务端的连接通信。
```js
// lib/_http_client.js
function ClientRequest(options, cb) {
  var self = this;
  OutgoingMessage.call(self);

  // ...
  const defaultPort = options.defaultPort ||
                      self.agent && self.agent.defaultPort;

  var port = options.port = options.port || defaultPort || 80;
  var host = options.host = options.hostname || options.host || 'localhost';

  if (options.setHost === undefined) {
    var setHost = true;
  }

  self.socketPath = options.socketPath;

  var method = self.method = (options.method || 'GET').toUpperCase();
  if (!common._checkIsHttpToken(method)) {
    throw new TypeError('Method must be a valid HTTP token');
  }
  self.path = options.path || '/';
  if (cb) {
    self.once('response', cb);
  }

  // ...

  var called = false;
  if (self.socketPath) {
    // ...
  } else if (self.agent) {
    // ...
  } else {
    // No agent, default to Connection:close.
    self._last = true;
    self.shouldKeepAlive = false;
    if (typeof options.createConnection === 'function') {
      const newSocket = options.createConnection(options, oncreate);
      if (newSocket && !called) {
        called = true;
        self.onSocket(newSocket);
      } else {
        return;
      }
    } else {
      debug('CLIENT use net.createConnection', options);
      self.onSocket(net.createConnection(options));
    }
  }

  function oncreate(err, socket) {
    // ...
  }

  self._deferToConnect(null, null, function() {
    self._flush();
    self = null;
  });
}

util.inherits(ClientRequest, OutgoingMessage);
```
callback 通过 `self.once('response', cb);`, 监听了 response 事件。之后如果没有设置代理服务，则默认使用
net 模块创建与服务器的连接。那么 response 事件是哪里发送的呢？

下面我们看到比较重要的 `onSocket` 函数。
```js
ClientRequest.prototype.onSocket = function(socket) {
  process.nextTick(onSocketNT, this, socket);
};

function onSocketNT(req, socket) {
  if (req.aborted) {
    // If we were aborted while waiting for a socket, skip the whole thing.
    socket.emit('free');
  } else {
    tickOnSocket(req, socket);
  }
}
```

这边 onSocket 必须是一个异步函数，大家可以仔细体会下！ 同时 onSocketNT 会异常做了处理，当请求失败时，则发送 free 事件。
否则来到 `tickOnSocket`。

```js
function tickOnSocket(req, socket) {
  var parser = parsers.alloc();
  req.socket = socket;
  req.connection = socket;
  parser.reinitialize(HTTPParser.RESPONSE);
  parser.socket = socket;
  parser.incoming = null;
  parser.outgoing = req;
  req.parser = parser;

  socket.parser = parser;
  socket._httpMessage = req;

  // Setup "drain" propagation.
  httpSocketSetup(socket);

  // Propagate headers limit from request object to parser
  if (typeof req.maxHeadersCount === 'number') {
    parser.maxHeaderPairs = req.maxHeadersCount << 1;
  } else {
    // Set default value because parser may be reused from FreeList
    parser.maxHeaderPairs = 2000;
  }

  parser.onIncoming = parserOnIncomingClient;
  socket.removeListener('error', freeSocketErrorListener);
  socket.on('error', socketErrorListener);
  socket.on('data', socketOnData);
  socket.on('end', socketOnEnd);
  socket.on('close', socketCloseListener);
  req.emit('socket', socket);
}
```

同 HTTP Server 类似，从池中申请一个解析器，用于解析 HTTP 协议， 到这一步说明连接已经建立，所以重新设置 error 事件的回调。

同时设置 数据回调等，然后发送 `socket`事件, 来到 `parserOnIncomingClient`.

```js
// client
function parserOnIncomingClient(res, shouldKeepAlive) {
  var socket = this.socket;
  var req = socket._httpMessage;


  // propagate "domain" setting...
  if (req.domain && !res.domain) {
    debug('setting "res.domain"');
    res.domain = req.domain;
  }

  debug('AGENT incoming response!');

  if (req.res) {
    // We already have a response object, this means the server
    // sent a double response.
    socket.destroy();
    return;
  }
  req.res = res;

  var isHeadResponse = req.method === 'HEAD';
 
  // ...
  req.res = res;
  res.req = req;

  // add our listener first, so that we guarantee socket cleanup
  res.on('end', responseOnEnd);
  var handled = req.emit('response', res);

  // If the user did not listen for the 'response' event, then they
  // can't possibly read the data, so we ._dump() it into the void
  // so that the socket doesn't hang there in a paused state.
  if (!handled)
    res._dump();

  return isHeadResponse;
}
```

在这里发送 response 事件，参数对象 `res` 上也挂上了 `req` 对象。这样 req 和 res 就相互引用。

用户的 callback 终于得到回调。

### 总结
上面只是梳理了一个http client 主线， 实际我们很少使用该模块，而是使用第三方的 npm 包，比如
* urllib (轻量级)
* request


### 参考


