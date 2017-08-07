
## cluster


### 背景
众所周知，Node.js是单线程的，一个单独的Node.js进程无法充分利用多核。Node.js从v0.8开始，新增cluster模块，让Node.js开发Web服务时，很方便的做到充分利用多核机器。
 
充分利用多核的思路是：使用多个进程处理业务。cluster模块封装了创建子进程、进程间通信、服务负载均衡。有两类进程，master进程和worker进程，master进程是主控进程，它负责启动worker进程，worker是子进程、干活的进程。

### 竞争模型 

最初的 Node.js 多进程模型就是这样实现的，master 进程创建 socket，绑定到某个地址以及端口后，自身不调用 listen 来监听连接以及 accept 连接，而是将该 socket 的 fd 传递到 fork 出来的 worker 进程，worker 接收到 fd 后再调用 listen，accept 新的连接。但实际一个新到来的连接最终只能被某一个 worker 进程 accpet 再做处理，至于是哪个 worker 能够 accept 到，开发者完全无法预知以及干预。这势必就导致了当一个新连接到来时，多个 worker 进程会产生竞争，最终由胜出的 worker 获取连接。

相信到这里大家也应该知道这种多进程模型比较明显的问题了

* 多个进程之间会竞争 accpet 一个连接，产生惊群现象，效率比较低。
* 由于无法控制一个新的连接由哪个进程来处理，必然导致各 worker 进程之间的负载非常不均衡。



### round-robin (轮询)

上面的多进程模型存在诸多问题，于是就出现了基于round-robin的另一种模型。
主要思路是master进程创建socket，绑定好地址以及端口后再进行监听。该socket的fd不传递到各个worker进程，当master进程获取到新的连接时，再决定将accept到的客户端socket fd传递给指定的worker处理。我这里使用了指定, 所以如何传递以及传递给哪个worker完全是可控的，round-robin只是其中的某种算法而已，当然可以换成其他的。

Master是如何将接收的请求传递至worker中进行处理然后响应的？

Cluster 模块通过监听该内部TCP服务器的connection事件，在监听器函数里，有负载均衡地挑选出一个worker，向其发送newconn内部消息（消息体对象中包含cmd: 'NODE_CLUSTER'属性）以及一个客户端句柄（即connection事件处理函数的第二个参数），相关代码如下：
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
至此，也总结一下：

* 所有请求先同一经过内部TCP服务器。
* 在内部TCP服务器的请求处理逻辑中，有负载均衡地挑选出一个worker进程，将其发送一个newconn内部消息，随消息发送客户端句柄。
* Worker进程接收到此内部消息，根据客户端句柄创建net.Socket实例，执行具体业务逻辑，返回。

### listen 端口复用
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
你可能已经猜到，答案就在这个cluster._getServer函数的代码中。它主要干了两件事：
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

至此，总结下：
* 端口仅由master进程中的内部TCP服务器监听了一次。
* 不会出现端口被重复监听报错，是由于，worker进程中，最后执行监听端口操作的方法，已被cluster模块主动覆盖。



### 总结


### 参考