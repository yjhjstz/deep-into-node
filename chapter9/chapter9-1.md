
## Net

### Network Model
The OSI reference model developed by ISO has been criticized for being too large and complex. In contrast, the TCP/IP protocol stack developed by technical personnel themselves has been more widely used. The following figure shows a comparison diagram of the TCP/IP reference model and the OSI reference model.

![NET](http://img.blog.csdn.net/20160324203556764)

### UDP vs TCP
* TCP (Transmission Control Protocol): Transmission Control Protocol
* UDP (User Datagram Protocol): User Datagram Protocol

The main differences are in connectivity, reliability, ordering, boundary, congestion or flow control, transmission speed, weight, and header size.

#### Main differences:
* TCP is a connection-oriented protocol, while UDP is a connectionless protocol;
  * TCP uses a three-way handshake to establish a connection: 1) The client sends a SYN to the server; 2) The server receives the SYN and replies to the client with a SYN-ACK; 3) The client receives the SYN-ACK and replies to the server with an ACK. At this point, the connection is established. UDP does not need to establish a connection before sending data.
  ![](http://p.blog.csdn.net/images/p_blog_csdn_net/ruimingde/492389/o_tcp三次握手.bmp)


* TCP is reliable, while UDP is unreliable;
  * TCP will automatically retransmit lost packets, while UDP will not.


* TCP is ordered, while UDP is unordered;
  * Messages may be out of order during transmission, and later messages may arrive first. TCP will reorder them, while UDP will not.

From the perspective of program implementation, the following figure can be used to describe it.

![](http://img.my.csdn.net/uploads/201303/15/1363304870_3150.jpg)

From the above figure, it can be seen clearly that TCP communication requires the server-side listening to listen, receiving the client connection request to accept, waiting for the client to connect to establish a connection before receiving and sending data packets (recv/send). UDP does not have a clear concept of server and client. The server-side receiving end needs to bind the port and wait for the client's data to arrive. Subsequently, data reception and transmission (recvfrom/sendto) can be performed.

### Socket abstraction
Socket is a encapsulation of the TCP/IP protocol family, which is a middle software abstraction layer for communication between the application layer and the TCP/IP protocol family. It hides the complex TCP/IP protocol family behind the Socket interface. For users, a set of simple interfaces is everything, and Socket organizes data to conform to the specified protocol.

Socket can also be regarded as a method of inter-process communication between processes on different computers in the network. The triplet (IP address, protocol, port) can uniquely identify the process in the network, and the process communication in the network can use this mark to interact with other processes.

Socket originated from Unix, and one of the basic philosophies of Unix/Linux is "everything is a file". All operations can be performed using the "open (open) -> read/write (write/read) -> close (close)" mode. Therefore, Socket is also treated as a special file.

#### C++ binding
TCP binding export:
```c++
void TCPWrap::Initialize(Local<Object> target,
                         Local<Value> unused,
                         Local<Context> context) {
  Environment* env = Environment::GetCurrent(context);

  Local<FunctionTemplate> t = env->NewFunctionTemplate(New);
  t->SetClassName(FIXED_ONE_BYTE_STRING(env->isolate(), "TCP"));
  t->InstanceTemplate()->SetInternalFieldCount(1);

  // Init properties
  t->InstanceTemplate()->Set(String::NewFromUtf8(env->isolate(), "reading"),
                             Boolean::New(env->isolate(), false));
  t->InstanceTemplate()->Set(String::NewFromUtf8(env->isolate(), "owner"),
                             Null(env->isolate()));
  t->InstanceTemplate()->Set(String::NewFromUtf8(env->isolate(), "onread"),
                             Null(env->isolate()));
  t->InstanceTemplate()->Set(String::NewFromUtf8(env->isolate(),
                                                 "onconnection"),
                             Null(env->isolate()));


  env->SetProtoMethod(t, "close", HandleWrap::Close);

  env->SetProtoMethod(t, "ref", HandleWrap::Ref);
  env->SetProtoMethod(t, "unref", HandleWrap::Unref);

  StreamWrap::AddMethods(env, t, StreamBase::kFlagHasWritev);

  env->SetProtoMethod(t, "open", Open);
  env->SetProtoMethod(t, "bind", Bind);
  env->SetProtoMethod(t, "listen", Listen);
  env->SetProtoMethod(t, "connect", Connect);
  env->SetProtoMethod(t, "bind6", Bind6);
  env->SetProtoMethod(t, "connect6", Connect6);
  env->SetProtoMethod(t, "getsockname",
                      GetSockOrPeerName<TCPWrap, uv_tcp_getsockname>);
  env->SetProtoMethod(t, "getpeername",
                      GetSockOrPeerName<TCPWrap, uv_tcp_getpeername>);
  env->SetProtoMethod(t, "setNoDelay", SetNoDelay);
  env->SetProtoMethod(t, "setKeepAlive", SetKeepAlive);

#ifdef _WIN32
  env->SetProtoMethod(t, "setSimultaneousAccepts", SetSimultaneousAccepts);
#endif

  target->Set(FIXED_ONE_BYTE_STRING(env->isolate(), "TCP"), t->GetFunction());
  env->set_tcp_constructor_template(t);

  // Create FunctionTemplate for TCPConnectWrap.
  Local<FunctionTemplate> cwt =
      FunctionTemplate::New(env->isolate(), NewTCPConnectWrap);
  cwt->InstanceTemplate()->SetInternalFieldCount(1);
  cwt->SetClassName(FIXED_ONE_BYTE_STRING(env->isolate(), "TCPConnectWrap"));
  target->Set(FIXED_ONE_BYTE_STRING(env->isolate(), "TCPConnectWrap"),
              cwt->GetFunction());
}
```
`TCPWrap`导出了 TCP 类，TCPConnectWrap 类，并且我们看到对 IPV6协议族的支持：`bind6`, `connect6`。

#### TCP Socket
Node.js 的 Net模块也对 TCP socket 进行了抽象封装：
```js
function Socket(options) {
  if (!(this instanceof Socket)) return new Socket(options);

  this._connecting = false;
  this._hadError = false;
  this._handle = null;
  this._parent = null;
  this._host = null;

  if (typeof options === 'number')
    options = { fd: options }; // Legacy interface.
  else if (options === undefined)
    options = {};

  stream.Duplex.call(this, options);

  if (options.handle) {
    this._handle = options.handle; // private
  } else if (options.fd !== undefined) {
    this._handle = createHandle(options.fd);
    this._handle.open(options.fd);
    if ((options.fd == 1 || options.fd == 2) &&
        (this._handle instanceof Pipe) &&
        process.platform === 'win32') {
      // Make stdout and stderr blocking on Windows
      var err = this._handle.setBlocking(true);
      if (err)
        throw errnoException(err, 'setBlocking');
    }
    this.readable = options.readable !== false;
    this.writable = options.writable !== false;
  } else {
    // these will be set once there is a connection
    this.readable = this.writable = false;
  }

  // shut down the socket when we're finished with it.
  this.on('finish', onSocketFinish);
  this.on('_socketEnd', onSocketEnd);

  initSocketHandle(this);

  // ...
}
util.inherits(Socket, stream.Duplex);
```
The `Socket` class is a full-duplex stream that inherits from `Duplex`. It creates a socket using `createHandle` and assigns it to `this._handle`.

It also listens for the `finish` and `_socketEnd` events.

### Packet Concatenation
Packet concatenation typically occurs during stream transmission, as TCP is a stream-based protocol. Additionally, network MTU values are often smaller than the message data being processed by the application, which can cause a single data reception to not fully represent a complete message. The only way to handle packet concatenation is to define an application-layer data communication protocol that specifies whether the received data meets the needs of the message data.

#### Analysis
TCP packet concatenation usually occurs during stream transmission, while UDP does not have packet concatenation because UDP has message boundaries. When sending a data segment using the TCP protocol, the data segment needs to wait for the buffer to be full before sending the data. When the buffer is full, it may not be a single message, but several messages that exist in the buffer. In order to optimize performance (Nagle's algorithm), TCP combines these small data packets into a large data packet, causing packet concatenation. In addition, if the receiving end of the data cannot receive the packets in the buffer in time, it will also cause multiple packets in the buffer to be combined and received, which is also packet concatenation.

#### Solutions
* Define a custom application-layer protocol;
* Do not use the Nagle algorithm, use the provided API: `socket.setNoDelay`.

#### UDP
##### Multicast
* https://en.wikipedia.org/wiki/Multicast#IP_multicast

##### UDP Socket
```js
function Socket(type, listener) {
  EventEmitter.call(this);

  if (typeof type === 'object') {
    var options = type;
    type = options.type;
  }

  var handle = newHandle(type);
  handle.owner = this;

  this._handle = handle;
  this._receiving = false;
  this._bindState = BIND_STATE_UNBOUND;
  this.type = type;
  this.fd = null; // compatibility hack

  // If true - UV_UDP_REUSEADDR flag will be set
  this._reuseAddr = options && options.reuseAddr;

  if (typeof listener === 'function')
    this.on('message', listener);
}
util.inherits(Socket, EventEmitter);
```

UDP inherits from `EventEmitter` and also supports IPV4 and IPV6 protocols, distinguished by `type`. `this._reuseAddr` indicates whether to use the `SO_REUSEADDR` option.

SO_REUSEADDR allows for completely duplicate binding: when an IP address and port are bound to a socket, this IP address and port can also be bound to another socket. Generally, this feature is only available on systems that support multicast, and only for UDP sockets (TCP does not support multicast).

### Summary
From my experience, try not to use `UDP` unless you know that packet loss has no impact on the application, otherwise troubleshooting network packet loss will drive you crazy!

### References
* https://en.wikipedia.org/wiki/Nagle's_algorithm


