## Application Building

### Creating a TCP Server
Here is a simple example of creating a TCP server socket in NodeJS, with relevant explanations in the code comments.
```js
var net = require('net');

var HOST = '127.0.0.1';
var PORT = 6969;

// Create a TCP server instance, and call the listen function to start listening on the specified port
// The callback function passed to net.createServer() will be the event handler for the "connection" event
// In each "connection" event, the socket object received by this callback function is unique
var server = net.createServer();
server.listen(PORT, HOST);
console.log('Server listening on ' +
    server.address().address + ':' + server.address().port);

server.on('connection', function(sock) {
    console.log('CONNECTED: ' +
         sock.remoteAddress +':'+ sock.remotePort);
});
```

First, let's take a look at `net.createServer`, which returns an instance of `Server`, as follows.
```js
1075 function Server(options, connectionListener) {
1076   if (!(this instanceof Server))
1077     return new Server(options, connectionListener);
1078 
1079   EventEmitter.call(this);
1080 
1081   var self = this;
1082 
1083   if (typeof options === 'function') {
1084     connectionListener = options;
1085     options = {};
1086     self.on('connection', connectionListener);
1087   } else {
1088     options = options || {};
1089 
1090     if (typeof connectionListener === 'function') {
1091       self.on('connection', connectionListener);
1092     }
1093   }
1094 
1095   this._connections = 0;
1096   // ...
1111 
1112   this._handle = null;
1113   this._usingSlaves = false;
1114   this._slaves = [];
1115   this._unref = false;
1116 
1117   this.allowHalfOpen = options.allowHalfOpen || false;
1118   this.pauseOnConnect = !!options.pauseOnConnect;
1119 }
1120 util.inherits(Server, EventEmitter);
```

`Server` inherits from `EventEmitter`. If a callback function is passed, L1086 and L1091 bind the passed function as a listener to the `connection` event, and then listen. Let's take a look at the callback processing when a connection arrives as a server.
```js
1400 function onconnection(err, clientHandle) {
1401   var handle = this;
1402   var self = handle.owner;
1403 
1404   debug('onconnection');
1405 
1406   if (err) {
1407     self.emit('error', errnoException(err, 'accept'));
1408     return;
1409   }
1410 
1411   if (self.maxConnections && self._connections >= self.maxConnections) {
1412     clientHandle.close();
1413     return;
1414   }
1415 
1416   var socket = new Socket({
1417     handle: clientHandle,
1418     allowHalfOpen: self.allowHalfOpen,
1419     pauseOnCreate: self.pauseOnConnect
1420   });
1421   socket.readable = socket.writable = true;
1422 
1423 
1424   self._connections++;
1425   socket.server = self;
1426   socket._server = self;
1427   // ...
1431   self.emit('connection', socket);
1432 }
```
This function is called back by `TCPWrap::OnConnection`, `tcp_wrap->MakeCallback(env->onconnection_string(), ARRAY_SIZE(argv), argv);`, where the first parameter indicates the status and the second parameter is the connection handle.

L1416-L1421, create a JS-level Socket based on the passed handle, and send a connection event to the observer at L1431.

In the example of the TCP server above, the server listens for the `connection` event and customizes the user processing logic.

### Creating a TCP Client

Now let's create a TCP client that connects to the server just created, sends a message to the server, and closes the connection after receiving feedback from the server. The following code describes this process.
```js
var net = require('net');

var HOST = '127.0.0.1';
var PORT = 6969;

var client = new net.Socket();
client.connect(PORT, HOST, function() {

    console.log('CONNECTED TO: ' + HOST + ':' + PORT);
    // Send data to the server immediately after establishing a connection, and the server will receive this data
    client.write('I am Chuck Norris!');

});

// Add a "data" event processing function to the client
// Data is the data sent back by the server
client.on('data', function(data) {

    console.log('DATA: ' + data);
    // Completely close the connection
    client.destroy();

});

// Add a "close" event processing function to the client
client.on('close', function() {
    console.log('Connection closed');
});
```

After creating the `Socket` object, the client initiates a connection to the server. Before the actual connection, a DNS query is required (not required if an IP is provided), and `lookupAndConnect` is called, followed by `function connect(self, address, port, addressType, localAddress, localPort)` to initiate the connection.

We notice the five-tuple: `<remoteAddress, remotePort, addressType, localAddress, localPort>`, which uniquely identifies a network connection.

After establishing a full-duplex Socket, the user program can listen for the `data` event to obtain data.

###
