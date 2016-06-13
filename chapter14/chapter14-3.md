### 楔子
`node-profiler` 作为 alinode 团队的另一款产品能够帮助您线下深入分析 javascript 代码的性能，将Google V8的性能细节展现在您的面前，优化而知其所以然。线上调优请使用 `alinode`。

### 下载安装
推荐安装工具`tnvm`，支持 node, alinode, profiler 的安装切换。
```
wget -O- https://raw.githubusercontent.com/aliyun-node/tnvm/master/install.sh | bash
```
完成安装后，您需要将tnvm添加为命令行程序。根据平台的不同，可能是~/.bashrc，~/.profile或~/.zshrc.
```shell
 tnvm install profiler-v0.12.6
 tnvm use profiler-v0.12.6
```


### 使用示例
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
如出现如下：
```shell
Error: listen EADDRINUSE           <== 可能是由于端口被占用
```

成功启动后，则用chrome(推荐)手动打开url (http://alinode.aliyun.com/profiler/inspector.html?host=localhost:9999&page=0)
出现如下界面：
![](https://cloud.githubusercontent.com/assets/3832082/8587127/7b54f88c-262a-11e5-9298-3a49c2b71d7c.jpg)

默认**Collect JavaSript CPU Profile**，单击**Start**。

可以采用压测脚本实现对服务进行压力测试，保证更多的结果：
```shell
$ wrk http://localhost:1334/ # 这里使用wrk，也可以使用其他工具，如ab
```
点击**Stop**，得到如下图的结果：
![](https://cloud.githubusercontent.com/assets/3832082/8587247/8dc33cbc-262b-11e5-8a10-59c8f9de280e.jpg)

可以看到更多关于函数在运行时的信息。

### UI含义
UI 栏目 | 示意
----   | ----
Self | exclusive time
Total | inclusive time
# Hidden Classes  | 隐藏类个数
Bailout | v8中提取的最后一次去优化原因
Function | 函数名称 script : line

**红色表示函数未被优化， 淡绿色表示函数被V8优化过。**

### 原理介绍
- 基于V8内置采样收集器；
- 固定采样频率，默认1ms, 可配置；
- 会暂停主线程，采样函数call stack，统计时间；
- 需保证采样足够长的时间（预热）。

#### 解释下两个概念
- exclusive time :独占时间
- inclusive time :包含时间

```js
function foo() { 
   bar();
}
function bar() {
          <==采样点
}
foo();
```

这个例子，采样点在bar()，那么bar所消耗的时间叫作 **exclusive time**，而foo调用了bar, foo所消耗的时间包括了bar的时间，叫作 **inclusive time** .


### 注意事项
* 该工具目前只支持 X64 平台（Linux, Mac)。
* 切勿部署到线上，如需线上调优请使用 [alinode](alinode.aliyun.com)。