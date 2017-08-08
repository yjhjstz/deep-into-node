## 应用构建


### 创建TCP服务端
下面是一个在NodeJS中创建TCP服务端套接字的简单例子，相关说明见代码注释。
```js
var net = require('net');

var HOST = '127.0.0.1';
var PORT = 6969;

// 创建一个TCP服务器实例，调用listen函数开始监听指定端口
// 传入net.createServer()的回调函数将作为”connection“事件的处理函数
// 在每一个“connection”事件中，该回调函数接收到的socket对象是唯一的
var server = net.createServer();
server.listen(PORT, HOST);
console.log('Server listening on ' +
    server.address().address + ':' + server.address().port);

server.on('connection', function(sock) {
    console.log('CONNECTED: ' +
         sock.remoteAddress +':'+ sock.remotePort);
});
```

首先我们来看下 `net.createServer`, 它返回一个 `Server`的实例，如下。
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

`Server` 继承了 `EventEmitter`, 如果传入 callback 函数，  L1086，L1091 则把传入的函数作为监听者
绑定到 `connnection`事件上， 然后 listen 。我们看看作为 server 端连接到来的回调处理。
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
此函数由 `TCPWrap::OnConnection`回调，`tcp_wrap->MakeCallback(env->onconnection_string(), ARRAY_SIZE(argv), argv);`， 第一个参数标识状态，第二个参数为连接的句柄。

L1416-L1421, 根据传过来的句柄，创建 JS 层面的 Socket。并在 L1431 向观察者发送 connection 事件。


上面 TCP 服务端的例子，server 监听connection 事件，自定义用户处理逻辑。



### 创建TCP客户端

现在让我们创建一个TCP客户端连接到刚创建的服务器上，该客户端向服务器发送一串消息，并在得到服务器的反馈后关闭连接。下面的代码描述了这一过程。
```js
var net = require('net');

var HOST = '127.0.0.1';
var PORT = 6969;

var client = new net.Socket();
client.connect(PORT, HOST, function() {

    console.log('CONNECTED TO: ' + HOST + ':' + PORT);
    // 建立连接后立即向服务器发送数据，服务器将收到这些数据 
    client.write('I am Chuck Norris!');

});

// 为客户端添加“data”事件处理函数
// data是服务器发回的数据
client.on('data', function(data) {

    console.log('DATA: ' + data);
    // 完全关闭连接
    client.destroy();

});

// 为客户端添加“close”事件处理函数
client.on('close', function() {
    console.log('Connection closed');
});
```

创建 `Socket`对象后，client 端向server端发起连接，在真正的连接之前，需要进行 DNS 查询（提供 IP 的不用），
调用 `lookupAndConnect`, 之后才是调用 `function connect(self, address, port, addressType, localAddress, localPort) `发起连接。

我们注意到五元组: `<remoteAddress, remotePort, addressType, localAddress, localPort>`, 他们唯一的标识了一个网络连接。

建立起全双工的 Socket 后，用户程序就可以监听 「data」事件，获取数据了。



### 总结



### 参考