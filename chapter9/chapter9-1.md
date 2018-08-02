
## 网络 (Net)

### 网络模型
ISO制定的OSI参考模型的过于庞大、复杂招致了许多批评。与此对照，由技术人员自己开发的TCP/
IP协议栈获得了更为广泛的应用。如图所示，是TCP/IP参考模型和OSI参考模型的对比示意图。

![NET](http://img.blog.csdn.net/20160324203556764)

### UDP vs TCP
* TCP(Transmission Control Protocol)：传输控制协议
* UDP(User Datagram Protocol)：用户数据报协议

主要在于连接性(Connectivity)、可靠性(Reliability)、有序性(Ordering)、有界性(Boundary)、拥塞控制(Congestion or Flow control)、传输速度(Speed)、量级(Heavy/Light weight)、头部大小(Header size)等差异。

#### 主要差异：
* TCP是面向连接(Connection oriented)的协议，UDP是无连接(Connection less)协议；
  * TCP用三次握手建立连接：1) Client向server发送SYN；2) Server接收到SYN，回复Client一个SYN-ACK；3)Client接收到SYN_ACK，回复Server一个ACK。到此，连接建成。UDP发送数据前不需要建立连接。
  ![](http://p.blog.csdn.net/images/p_blog_csdn_net/ruimingde/492389/o_tcp三次握手.bmp)


* TCP可靠，UDP不可靠；
  * TCP丢包会自动重传，UDP不会。


* TCP有序，UDP无序；
  * 消息在传输过程中可能会乱序，后发送的消息可能会先到达，TCP会对其进行重排序，UDP不会。

从程序实现的角度来看，可以用下图来进行描述。

![](http://img.my.csdn.net/uploads/201303/15/1363304870_3150.jpg)

从上图也能清晰的看出，TCP通信需要服务器端侦听listen、接收客户端连接请求accept，等待客户端connect建立连接后才能进行数据包的收发（recv/send）工作。而UDP则服务器和客户端的概念不明显，服务器端即接收端需要绑定端口，等待客户端的数据的到来。后续便可以进行数据的收发（recvfrom/sendto）工作。

### Socket 抽象
Socket 是对 TCP/IP 协议族的一种封装，是应用层与TCP/IP协议族通信的中间软件抽象层。它把复杂的TCP/IP协议族隐藏在Socket接口后面，对用户来说，一组简单的接口就是全部，让Socket去组织数据，以符合指定的协议。

Socket 还可以认为是一种网络间不同计算机上的进程通信的一种方法，利用三元组（ip地址，协议，端口）就可以唯一标识网络中的进程，网络中的进程通信可以利用这个标志与其它进程进行交互。

Socket 起源于 Unix ，Unix/Linux 基本哲学之一就是“一切皆文件”，都可以用“打开(open) –> 读写(write/read) –> 关闭(close)”模式来进行操作。因此 Socket 也被处理为一种特殊的文件。

#### C++层绑定
TCP的绑定导出：
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
首先 `Socket` 是一个全双工的 Stream，所以继承了 Duplex。通过 `createHandle` 创建套接字并赋值到
`this._handle`上。

同时监听 `finish`, `_socketEnd`事件， 



### 粘包
> 一般所谓的TCP粘包是在一次接收数据不能完全地体现一个完整的消息数据。TCP通讯为何存在粘包呢？主要原因是TCP是以流的方式来处理数据，再加上网络上MTU的值往往小于在应用处理的消息数据，所以就会引发一次接收的数据无法满足消息的需要，导致粘包的存在。处理粘包的唯一方法就是制定应用层的数据通讯协议，通过协议来规范现有接收的数据是否满足消息数据的需要。

#### 情况分析

TCP粘包通常在流传输中出现，UDP则不会出现粘包，因为UDP有消息边界。使用TCP协议发送数据段需要等待缓冲区满了才将数据发送出去，当满的时候有可能不是一条消息而是几条消息存在于缓冲区内，为了优化性能（Nagle算法），TCP会将这几个小数据包合并为一个大的数据包，造成粘包；另外在接收数据端，如果没能及时接收缓冲区的包，也会造成缓冲区多包合并接收，这也是粘包。

#### 解决办法

* 自定义应用层协议；
* 不使用Nagle算法, 使用提供的 API：`socket.setNoDelay`。


### UDP
#### 组播
* https://en.wikipedia.org/wiki/Multicast#IP_multicast%EF%BC%89%EF%BC%8C%E5%85%B7

#### UDP Socket
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

UDP 继承了 `EventEmitter`, 同样也支持 IPV4和 IPV6协议， 由`type`区分，
`this._reuseAddr` 标识是否要使用选项：`SO_REUSEADDR`。

SO_REUSEADDR允许完全重复的捆绑：当一个IP地址和端口绑定到某个套接口上时，还允许此IP地址和端口捆绑到另一个套接口上。一般来说，这个特性仅在支持多播的系统上才有，而且只对UDP套接口而言（TCP不支持多播）。

### 总结
从笔者的经验看，尽量不要尝试去使用 `UDP`，除非你知道丢包了对于应用是没有影响的，否则排查网络丢包会使人崩溃的！


### 参考
* https://en.wikipedia.org/wiki/Nagle's_algorithm
