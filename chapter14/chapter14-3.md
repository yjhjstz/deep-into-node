### Introduction
`node-profiler` is another product of the alinode team that can help you analyze the performance of JavaScript code offline, presenting the performance details of Google V8 in front of you, and optimizing it. For online tuning, please use `alinode`.

### Download and Install
It is recommended to install the tool `tnvm`, which supports the installation and switching of node, alinode, and profiler.
```
wget -O- https://raw.githubusercontent.com/aliyun-node/tnvm/master/install.sh | bash
```
After the installation is complete, you need to add tnvm as a command-line program. Depending on the platform, it may be `~/.bashrc`, `~/.profile`, or `~/.zshrc`.
```shell
 tnvm install profiler-v0.12.6
 tnvm use profiler-v0.12.6
```


### Usage Example
```js
var http = require('http');
http.createServer(function (req, res) {
  res.writeHead(200);
  res.end('hello world!');
}).listen(1334);
```

```shell
$ node-profiler server.js
start agent            
webkit-devtools-agent: A proxy got connected.
webkit-devtools-agent: Waiting for commands...
webkit-devtools-agent: Websockets service started on 0.0.0.0:9999  <==启动成功
```
If the following appears:
```shell
Error: listen EADDRINUSE           <== 可能是由于端口被占用
```

After successful startup, manually open the URL (http://alinode.aliyun.com/profiler/inspector.html?host=localhost:9999&page=0) with Chrome (recommended).
The following interface appears:
![](https://cloud.githubusercontent.com/assets/3832082/8587127/7b54f88c-262a-11e5-9298-3a49c2b71d7c.jpg)

By default, **Collect JavaSript CPU Profile** is selected, click **Start**.

You can use the pressure test script to implement pressure testing on the service to ensure more results:
```shell
$ wrk http://localhost:1334/ # Here we use wrk, other tools such as ab can also be used
```
Click **Stop** to get the following results:
![](https://cloud.githubusercontent.com/assets/3832082/8587247/8dc33cbc-262b-11e5-8a10-59c8f9de280e.jpg)

More information about functions during runtime can be seen.

### UI Meaning
UI Column | Meaning
----   | ----
Self | exclusive time
Total | inclusive time
# Hidden Classes  | Number of hidden classes
Bailout | The last reason for deoptimization extracted in v8
Function | Function name script : line

**Red indicates that the function has not been optimized, and light green indicates that the function has been optimized by V8.**

### Principle Introduction
- Based on the built-in sampling collector of V8;
- Fixed sampling frequency, default 1ms, configurable;
- The main thread is paused, the function call stack is sampled, and the time is counted;
- It is necessary to ensure that the sampling is long enough (preheating).

#### Explanation of two concepts
- exclusive time: exclusive time
- inclusive time: inclusive time

```js
function foo() { 
   bar();
}
function bar() {
          <==Sampling point
}
foo();
```

In this example, the sampling point is in `bar()`, so the time consumed by `bar` is called **exclusive time**, and `foo` calls `bar`, so the time consumed by `foo` includes the time of `bar`, called **inclusive time**.


### Precautions
* This tool currently only supports X64 platforms (Linux, Mac).
* Do not deploy it online. If you need online tuning, please use [alinode](alinode.aliyun.com).

