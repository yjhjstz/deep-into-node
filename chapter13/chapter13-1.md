
## 子进程 (child_process)

child_process 是Node的一个十分重要的模块，通过它可以实现创建多进程，以利用单机的多核计算资源。虽然，Nodejs天生是单线程单进程的，但是有了child_process模块，可以在程序中直接创建子进程，并使用主进程和子进程之间实现通信。

### 进程通信
每个进程各自有不同的用户地址空间，任何一个进程的全局变量在另一个进程中都看不到，所以进程之间要交换数据必须通过内核，
在内核中开辟一块缓冲区，进程1把数据从用户空间拷到内核缓冲区，进程2再从内核缓冲区把数据读走，内核提供的这种机制称为进程间通信。


类型 |	无连接 |	可靠  |	流控制	| 优先级
----|---------|------|----------| -----
普通PIPE |N   |	Y	|  Y		| N
命名PIPE| N	| Y     |	Y		| N
消息队列| N	| Y     |	Y		| N
信号量  | N	| Y     |	Y		| Y
共享存储| N	| Y     |	Y		| Y
UNIX流SOCKET	| N	| Y     |	Y	| N
UNIX数据包SOCKET|	Y	|Y	|N		| N


* 注:无连接: 指无需调用某种形式的open,就有发送消息的能力流控制:

Node 中实现 IPC 通信的是管道技术，但只是抽象的称呼，具体细节实现由 libuv提供， 在 windows 下由命名管道（named pipe）实现， *nix 系统则采用 Unix Domain Socket实现。 也就是上表中的最后第二个。

Socket API原本是为网络通讯设计的，但后来在socket的框架上发展出一种IPC机制，就是UNIX Domain Socket。虽然网络socket也可用于同一台主机的进程间通讯（通过loopback地址127.0.0.1），但是UNIX Domain Socket用于IPC更有效率：不需要经过网络协议栈，不需要打包拆包、计算校验和、维护序号和应答等，只是将应用层数据从一个进程拷贝到另一个进程。

> Depending on the platform, unix domain sockets can achieve around 50% more throughput than the TCP/IP loopback (on Linux for instance).

这是因为，IPC机制本质上是可靠的通讯，而网络协议是为不可靠的通讯设计的。UNIX Domain Socket也提供面向流和面向数据包两种API接口，类似于TCP和UDP，但是面向消息的UNIX Domain Socket也是可靠的，消息既不会丢失也不会顺序错乱。


### 创建子进程
* spawn()启动一个子进程来执行命令
* exec()启动一个子进程来执行命令, 带回调参数获知子进程的情况, 可指定进程运行的超时时间
* execFile()启动一个子进程来执行一个可执行文件, 可指定进程运行的超时时间
* fork() 与spawn()类似, 不同在于它创建的node子进程只需指定要执行的js文件模块即可
```js
// don't call this example code
var cp = require('child_process');
cp.spawn('node', ['work.js']);
cp.exec('node work.js', function(err, stdout, stderr) {
  // some code
});
cp.execFile('work.js', function(err, stdout, stderr) {
  // some code
});
cp.fork('./work.js');
```

exec方法会直接调用bash（/bin/sh程序）来解释命令，所以如果有用户输入的参数，exec方法是不安全的。
```js
var path = ";user input";
child_process.exec('ls -l ' + path, function (err, data) {
  console.log(data);
});
```
上面代码表示，在bash环境下，`ls -l; user input`
会直接运行。如果用户输入恶意代码，将会带来安全风险。因此，在有用户输入的情况下，最好不使用exec方法，而是使用execFile方法。



### 建立 IPC 通道
父进程在创建子进程前创建IPC通道并监听, 用环境变量NODE_CHANNEL_FD告诉子进程的IPC的文件描述符。
```js
startup.processChannel = function() {
  // If we were spawned with env NODE_CHANNEL_FD then load that up and
  // start parsing data from that stream.
  if (process.env.NODE_CHANNEL_FD) {
    var fd = parseInt(process.env.NODE_CHANNEL_FD, 10);
    assert(fd >= 0);

    // Make sure it's not accidentally inherited by child processes.
    delete process.env.NODE_CHANNEL_FD;

    var cp = NativeModule.require('child_process');

    // Load tcp_wrap to avoid situation where we might immediately receive
    // a message.
    // FIXME is this really necessary?
    process.binding('tcp_wrap');

    cp._forkChild(fd);
    assert(process.send);
  }
};

```

子进程在启动的过程中连接IPC的FD
```js
exports._forkChild = function(fd) {
  // set process.send()
  var p = new Pipe(true);
  p.open(fd);
  p.unref();
  const control = setupChannel(process, p);
  process.on('newListener', function(name) {
    if (name === 'message' || name === 'disconnect') control.ref();
  });
  process.on('removeListener', function(name) {
    if (name === 'message' || name === 'disconnect') control.unref();
  });
};
```
建立连接后父子进程就可以自由的，全双工的通信了。


### 句柄传递

ChildProcess 类的实例，通过调用 ChildProcess#send(message[, sendHandle[, options]][, callback]) 方法，我们可以实现与子进程的通信，其中的 sendHandle 参数支持传递 net.Server ，net.Socket 等多种句柄，使用它，我们可以很轻松的实现在进程间转发 TCP socket。

send方法可以发送的对象包括如下集中：

- net.Socket对象: TCP套接字
- net.Server对象: TCP服务器
- net.Native: C++层面的TCP套接字和IPC管道
- dgram.Socket: UDP套接字
- dgram.Native: C++层面的UDP套接字

传递的过程：

**主进程**：

- 传递消息和句柄。
- 将消息包装成内部消息，使用 JSON.stringify 序列化为字符串。
- 通过对应的 handleConversion[message.type].send 方法序列化句柄。
- 将序列化后的字符串和句柄发入 IPC channel 。

**子进程**：

- 使用 JSON.parse 反序列化消息字符串为消息对象。
- 触发内部消息事件（internalMessage）监听器。
- 将传递来的句柄使用 handleConversion[message.type].got 方法反序列化为 JavaScript 对象。
- 带着消息对象中的具体消息内容和反序列化后的句柄对象，触发用户级别事件。


### 总结
很多应用比如 redis提供了本地访问的接口，进程通信使用的是 socket 的回环地址。当然它是通用性的考虑，否则要区分本地环境还是网络环境，如果不考虑这点，其实可以用 unix domain socket
代替，以获取更好的相互性能。

> Here you have the results on a single CPU 3.3GHz Linux machine :

类型  |   TCP | UDS |  PIPE
-----| -----| ------| ----
latency | 6us | 2us | 2us 
throughput | 253702 msg/s| 1733874 msg/s | 1682796 msg/s

* UDS: UNIX Domain Socket


### 参考

[1]. https://github.com/rigtorp/ipc-bench

