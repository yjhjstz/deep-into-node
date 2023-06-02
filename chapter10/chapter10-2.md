## HTTP 2/2

The http module provides two functions, http.request and http.get, which are used as clients to send requests to HTTP servers.

This is usually used to implement web crawlers. Here is an example of a web crawler that I wrote to crawl Zhihu: https://github.com/yjhjstz/iZhihu


### GET Example
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
The above program will return a status code of 200!

### HTTP Client
In Node.js, http.get creates a `ClientRequest` object to establish a connection with the server.

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
The callback listens for the response event. If no proxy service is set, the net module is used to create a connection with the server. But where is the response event sent?

Let's take a look at the important `onSocket` function.
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

The onSocket function must be an asynchronous function. If the request fails, the free event is sent. Otherwise, it goes to `tickOnSocket`.

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

Like the HTTP server, a parser is requested from the pool to parse the HTTP protocol. At this point, the connection has been established, so the error event callback is reset.

Data callback is also set, and the `socket` event is sent to `parserOnIncomingClient`.

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

// Send the response event here, and the `req` object is also attached to the `res` object. This way, `req` and `res` reference each other.

The user's callback is finally called.

### Summary
The above only outlines the main line of the http client. In practice, we rarely use this module, but instead use third-party npm packages, such as:
* urllib (lightweight)
* request


### Reference
```




