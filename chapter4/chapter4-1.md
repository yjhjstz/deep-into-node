
## cluster

### 背景
众所周知，Node.js中的JavaScript代码执行在单线程中，非常脆弱，一旦出现了未捕获的异常，那么整个应用就会崩溃。这在许多场景下，尤其是web应用中，是无法忍受的。通常的解决方案，便是使用Node.js中自带的cluster模块，以master-worker模式启动多个应用实例。然而大家在享受cluster模块带来的福祉的同时，不少人也开始好奇：

为什么我的应用代码中明明有app.listen(port);，但cluter模块在多次fork这份代码时，却没有报端口已被占用？
Master是如何将接收的请求传递至worker中进行处理然后响应的？
让我们从Node.js项目的lib/cluster.js中的代码里，来一勘究竟。


### 问题一
为了得到这个问题的解答，我们先从worker进程的初始化看起，master进程在fork工作进程时，会为其附上环境变量NODE_UNIQUE_ID，是一个从零开始的递增数：

```js
// lib/cluster.js
// ...

function createWorkerProcess(id, env) {
  // ...
  workerEnv.NODE_UNIQUE_ID = '' + id;

  // ...
  return fork(cluster.settings.exec, cluster.settings.args, {
    env: workerEnv,
    silent: cluster.settings.silent,
    execArgv: execArgv,
    gid: cluster.settings.gid,
    uid: cluster.settings.uid
  });
}
```
随后Node.js在初始化时，会根据该环境变量，来判断该进程是否为cluster模块fork出的工作进程，若是，则执行workerInit()函数来初始化环境，否则执行masterInit()函数。

在workerInit()函数中，定义了cluster._getServer方法，这个方法在任何net.Server实例的listen方法中，会被调用：

```js
// lib/net.js
// ...

function listen(self, address, port, addressType, backlog, fd, exclusive) {
  exclusive = !!exclusive;

  if (!cluster) cluster = require('cluster');

  if (cluster.isMaster || exclusive) {
    self._listen2(address, port, addressType, backlog, fd);
    return;
  }

  cluster._getServer(self, {
    address: address,
    port: port,
    addressType: addressType,
    fd: fd,
    flags: 0
  }, cb);

  function cb(err, handle) {
    // ...

    self._handle = handle;
    self._listen2(address, port, addressType, backlog, fd);
  }
}
```
你可能已经猜到，问题一的答案，就在这个cluster._getServer函数的代码中。它主要干了两件事：
* 向master进程注册该worker，若master进程是第一次接收到监听此端口/描述符下的worker，则起一个内部TCP服务器，来承担监听该端口/描述符的职责，随后在master中记录下该worker。
* Hack掉worker进程中的net.Server实例的listen方法里监听端口/描述符的部分，使其不再承担该职责。

对于第一件事，由于master在接收，传递请求给worker时，会符合一定的负载均衡规则（在非Windows平台下默认为轮询），这些逻辑被封装在RoundRobinHandle类中。故，初始化内部TCP服务器等操作也在此处：
```js
// lib/cluster.js
// ...

function RoundRobinHandle(key, address, port, addressType, backlog, fd) {
  // ...
  this.handles = [];
  this.handle = null;
  this.server = net.createServer(assert.fail);

  if (fd >= 0)
    this.server.listen({ fd: fd });
  else if (port >= 0)
    this.server.listen(port, address);
  else
    this.server.listen(address);  // UNIX socket path.

  /// ...
}
```
对于第二件事，由于net.Server实例的listen方法，最终会调用自身_handle属性下listen方法来完成监听动作，故在代码中修改之：
```js
// lib/cluster.js
// ...

function rr(message, cb) {
  // ...
  // 此处的listen函数不再做任何监听动作
  function listen(backlog) {
    return 0;
  }

  function close() {
    // ...
  }
  function ref() {}
  function unref() {}

  var handle = {
    close: close,
    listen: listen,
    ref: ref,
    unref: unref,
  };
  // ...
  handles[key] = handle;
  cb(0, handle); // 传入这个cb中的handle将会被赋值给net.Server实例中的_handle属性
}

// lib/net.js
// ...
function listen(self, address, port, addressType, backlog, fd, exclusive) {
  // ...

  if (cluster.isMaster || exclusive) {
    self._listen2(address, port, addressType, backlog, fd);
    return; // 仅在worker环境下改变
  }
    
  cluster._getServer(self, {
    address: address,
    port: port,
    addressType: addressType,
    fd: fd,
    flags: 0
  }, cb);

  function cb(err, handle) {
    // ...
    self._handle = handle; 
    // ...
  }
}
```

至此，第一个问题便已豁然开朗了，总结下：
* 端口仅由master进程中的内部TCP服务器监听了一次。
* 不会出现端口被重复监听报错，是由于，worker进程中，最后执行监听端口操作的方法，已被cluster模块主动覆盖。


### 问题二

解决了问题一，问题二的解决就明朗轻松许多了。通过问题一我们已得知，监听端口的是master进程中创建的内部TCP服务器，所以第二个问题的解决，着手点就是该内部TCP服务器接手连接时，执行的操作。Cluster模块的做法是，监听该内部TCP服务器的connection事件，在监听器函数里，有负载均衡地挑选出一个worker，向其发送newconn内部消息（消息体对象中包含cmd: 'NODE_CLUSTER'属性）以及一个客户端句柄（即connection事件处理函数的第二个参数），相关代码如下：
```js
// lib/cluster.js
// ...

function RoundRobinHandle(key, address, port, addressType, backlog, fd) {
  // ...
  this.server = net.createServer(assert.fail);
  // ...

  var self = this;
  this.server.once('listening', function() {
    // ...
    self.handle.onconnection = self.distribute.bind(self);
  });
}

RoundRobinHandle.prototype.distribute = function(err, handle) {
  this.handles.push(handle);
  var worker = this.free.shift();
  if (worker) this.handoff(worker);
};

RoundRobinHandle.prototype.handoff = function(worker) {
  // ...
  var message = { act: 'newconn', key: this.key };
  var self = this;
  sendHelper(worker.process, message, handle, function(reply) {
    // ...
  });
};
```
Worker进程在接收到了newconn内部消息后，根据传递过来的句柄，调用实际的业务逻辑处理并返回：
```js
// lib/cluster.js
// ...

// 该方法会在Node.js初始化时由 src/node.js 调用
cluster._setupWorker = function() {
  // ...
  process.on('internalMessage', internal(worker, onmessage));

  // ...
  function onmessage(message, handle) {
    if (message.act === 'newconn')
      onconnection(message, handle);
    // ...
  }
};

function onconnection(message, handle) {
  // ...
  var accepted = server !== undefined;
  // ...
  if (accepted) server.onconnection(0, handle);
}
```
至此，问题二也得到了解决，也总结一下：

* 所有请求先同一经过内部TCP服务器。
* 在内部TCP服务器的请求处理逻辑中，有负载均衡地挑选出一个worker进程，将其发送一个newconn内部消息，随消息发送客户端句柄。
* Worker进程接收到此内部消息，根据客户端句柄创建net.Socket实例，执行具体业务逻辑，返回。

### 总结