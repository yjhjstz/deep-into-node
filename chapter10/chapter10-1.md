
## HTTP 1/2

Back to our previous "Hello World" example, it only takes a few lines of code.
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

Because Node.js has encapsulated many details in the source code, the main code is in the files lib/_http_*.js. Now let's follow the above code and see what Node.js has done for us at the source code level from the arrival of an HTTP request to the response.

### Server
In Node.js, to receive an HTTP request, you first need to create an instance of the http.Server class and then listen for its request event. Since the HTTP protocol belongs to the application layer, the lower transport layer usually uses the TCP protocol, so the net.Server class is the parent class of the http.Server class.

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

The `requestListener` callback function acts as an observer and listens for the `request` event, with a default timeout of 2 minutes.

When the connection is established, the observer `connectionListener` handles the `connection` event.

At this point, an HTTP parser is needed to parse the data transmitted through TCP:

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

It is worth mentioning that the parser is obtained from a "pool", which uses a data structure called freelist. In order to reuse the parser as much as possible and avoid the cost of constantly calling the constructor, and there is a limit on the number (1000 in the http module).

The implementation of HTTPParser is currently implemented by C++ binding. For details, see the deps/http_parser directory. But here I will expand a bit:

There has been a debate in the community about the performance of the http_parser implementation, and the JS implementation of performance surpasses the C implementation.

The reasons are many:
* Removed the C++ binding layer.
* JS implementation avoids switching between C stack and JS stack and parameter copying.
* V8 JIT optimizes hot functions.

Even with these advantages, the community has not yet merged and is in a pending state. Combining personal and community views:
* Concurrent requests will cause frequent garbage collection and trigger GC pauses.
* Can exist as a third-party module.

> pull request: https://github.com/nodejs/node/pull/1457/

The parser here is also event-based, which is in line with Node.js's core ideas.
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
So a complete HTTP request from receiving to complete parsing will go through the event listeners on the parser one by one:

* parserOnHeaders: Continuously parse the incoming request header data.
* parserOnHeadersComplete: The request header is parsed, the header object is constructed, and the http.IncomingMessage instance is created for the request body.
* parserOnBody: Continuously parse the incoming request body data.
* parserOnExecute: After the request body is parsed, check whether the parsing reports an error. If an error is reported, the clientError event is triggered directly. If the request is a CONNECT method or has an Upgrade header, the connect or upgrade event is triggered directly.
* parserOnIncoming: Handle the specific parsed request.

Where is the `request` event triggered? Let's go back to the source code.


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

We notice two queues, `incoming` and `outgoing`, which are used to buffer IncomingMessage instances and their corresponding ServerResponse instances.
By constructing the corresponding `ServerResponse` instance through the `IncomingMessage` instance, and binding the triple `<req, res, socket>` through `res.assignSocket(socket);`.

Finally, the `request` event is sent with `req` and `res` as parameters. In the "Hello World" example, the listener receives `req` and `res`, writes the HTTP header and content to the response stream, and sends it out.

### Summary
Object pools are a derivative of memory pools and require a trade-off between memory and performance.

The above only outlines a main line, and other aspects such as exception handling and security will be analyzed in subsequent chapters.

### Reference
* https://docs.google.com/document/d/1A3cxhZg2aktJeSGt0P-8KrA4WyG1c8LlPomQflaQs8s/edit
```












