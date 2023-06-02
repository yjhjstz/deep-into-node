## cluster


### Background
As we all know, Node.js is single-threaded, and a single Node.js process cannot fully utilize multiple cores. Since v0.8, Node.js has added the cluster module, which makes it easy to fully utilize multi-core machines when developing web services with Node.js.

The idea of fully utilizing multiple cores is to use multiple processes to handle business. The cluster module encapsulates the creation of child processes, inter-process communication, and service load balancing. There are two types of processes, the master process and the worker process. The master process is the main control process, responsible for starting the worker process, and the worker is the child process and the working process.

### Competition Model

The initial Node.js multi-process model was implemented in this way. The master process creates a socket, binds it to a certain address and port, and does not call listen to listen for connections and accept connections, but passes the fd of the socket to the worker process forked out. The worker receives the fd and then calls listen to accept new connections. However, in reality, a new connection can only be accepted and processed by a certain worker process, and which worker can accept it cannot be predicted or intervened by the developer. This inevitably leads to competition between multiple worker processes when a new connection arrives, and the winner will get the connection.

I believe that everyone should know the obvious problems with this multi-process model

* Multiple processes compete to accept a connection, resulting in a low efficiency.
* Since it is impossible to control which process handles a new connection, the load between worker processes is very unbalanced.

### Round-robin

The above multi-process model has many problems, so another model based on round-robin appeared. The main idea is that the master process creates a socket, binds the address and port, and then listens. The fd of the socket is not passed to each worker process. When the master process receives a new connection, it decides which worker to pass the accepted client socket fd to. I used the specified method here, so how to pass and which worker to pass to is completely controllable. Round-robin is just one of the algorithms, of course, it can be replaced with others.

How does the master pass the received request to the worker for processing and then respond?

The Cluster module listens to the connection event of the internal TCP server. In the listener function, it selects a worker with load balancing and sends a newconn internal message (the message body object contains the cmd: 'NODE_CLUSTER' attribute) and a client handle (that is, the second parameter of the connection event processing function) to it. The relevant code is as follows:
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
After the worker process receives the newconn internal message, it creates a net.Socket instance based on the passed handle, executes the specific business logic, and returns:
```js
// lib/cluster.js
// ...

// This method will be called by src/node.js when Node.js is initialized
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
So far, let's summarize:

* All requests first go through the internal TCP server.
* In the request processing logic of the internal TCP server, a worker process is selected with load balancing, and a newconn internal message is sent to it, along with the client handle.
* The worker process receives this internal message, creates a net.Socket instance based on the client handle, executes the specific business logic, and returns.

### Listen Port Reuse
To get the answer to this problem, let's start with the initialization of the worker process. When the master process forks a working process, it attaches the environment variable NODE_UNIQUE_ID to it, which is an incrementing number starting from zero:

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
Then, when Node.js is initialized, it judges whether the process is a working process forked by the cluster module based on this environment variable. If so, it executes the workerInit() function to initialize the environment, otherwise it executes the masterInit() function.

In the workerInit() function, the cluster._getServer method is defined. This method will be called in the listen method of any net.Server instance:

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

You may have guessed that the answer is in the code of the cluster._getServer function. It mainly does two things:
* Register the worker with the master process. If the master process receives a worker listening on this port/descriptor for the first time, it starts an internal TCP server to handle the listening on this port/descriptor, and then records the worker in the master process.
* Hack the part of the listen method of the net.Server instance in the worker process that listens on the port/descriptor, so that it no longer handles this responsibility.

For the first thing, since the master process receives and passes requests to workers according to certain load balancing rules (default to round-robin on non-Windows platforms), these logics are encapsulated in the RoundRobinHandle class. Therefore, the initialization of the internal TCP server and other operations are also performed here:

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
For the second thing, since the listen method of the net.Server instance ultimately calls the listen method under its own _handle property to complete the listening action, it is modified in the code as follows:
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

Summary:
* The port is only listened to once by the internal TCP server in the master process.
* There will be no error of port being listened to repeatedly, because the method that ultimately performs the listen operation in the worker process has been actively overridden by the cluster module.




