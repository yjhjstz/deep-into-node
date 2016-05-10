## Node.js Docker

### docker vs host
```js
var http = require('http');
    http.createServer(function (req, res) {
    res.writeHead(200, {'Content-Type': 'text/plain'});
    res.end('Hello World\n');
}).listen(1337); 
console.log('Server running at http://0.0.0.0:1337/');
```
针对我们最关心的性能问题，用 `node-v4.2.3` 做了对比测试：
`docker` vs `host` node 。结论是性能损失在1%~4%之间，依据网络，业务代码因子而不定，数据如下：
-  外部网络环境

```shell
docker node
root@ubuntu-512mb-nyc3-01:~/wrk# ./wrk http://192.241.209.*:1337
Running 10s test @ http://192.241.209.*:1337
  2 threads and 10 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency    76.01ms    1.24ms  88.79ms   83.68%
    Req/Sec    65.51     16.12   101.00     76.77%
  1305 requests in 10.04s, 198.81KB read
Requests/sec:    129.99
Transfer/sec:     19.80KB
host node
root@ubuntu-512mb-nyc3-01:~/wrk# ./wrk http://192.241.209.*:1337 
Running 10s test @ http://192.241.209.*:1337
  2 threads and 10 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency    75.85ms    1.87ms  87.10ms   61.29%
    Req/Sec    65.66     12.35   101.00     57.07%
  1307 requests in 10.03s, 199.11KB read
Requests/sec:    130.27
Transfer/sec:     19.85KB  
```
- 内部网络环境

```shell
** host node**
[root@centos7-x64 statusbar]# wrk  http://localhost:1337
Running 10s test @ http://localhost:1337
  2 threads and 10 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     1.51ms    1.34ms  39.81ms   98.03%
    Req/Sec     3.46k   821.79     4.29k    76.50%
  68971 requests in 10.05s, 10.26MB read
Requests/sec:   6862.84
Transfer/sec:      1.02MB 

docker node `--net=host`模式
[root@centos7-x64 ~]# wrk http://localhost:1337
Running 10s test @ http://127.0.0.1:1337
  2 threads and 10 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     1.49ms  287.14us   3.81ms   92.99%
    Req/Sec     3.30k   424.64     3.60k    91.00%
  65866 requests in 10.03s, 9.80MB read
Requests/sec:   6564.82
Transfer/sec:      0.98MB
 
```
总的来说，对于正常 web 应用，走外部网络连接，性能损失很小，却极大的方便了我们的开发运维。

### 部署应用
> 什么是 MEAN 架构？
> MEAN 表示 Mongodb / ExpressJS / AngularJS /NodeJS，是目前流行的网站应用开发组合，涵盖前端至后台。由于这些框架用的语言都是 Javascript，所以又戏称 Javascript Fullstack。

本例子中，我们将尝试部署一个 MEAN 架构的 NodeJS 应用。
```shell
目录结构
├── docker-node-full/
│   ├── start.sh
│   ├── Dockerfile
```

### 安装 Docker
Ubuntu的系统，使用 apt 安装：
```sh
$ sudo apt-get install -y docker-engine
```


### 创建步骤
* 创建文件夹
```shell
mkdir ~/docker-node-full && cd $_
```

* 创建 Dockerfile 配置文件

```shell
# 设置基础镜像
FROM ubuntu:14.10

# 安装 NodeJS 和 npm
RUN apt-get install -y nodejs npm

# 由于 apt-get 下载的 Node 实际上是 nodejs，所以要创建一个 node 的快捷方式
RUN ln -s /usr/bin/nodejs /usr/bin/node

# 安装 Git
RUN apt-get install -y git

# 安装 Mongodb（来自官方教程）
RUN apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv 7F0CEB10
RUN echo 'deb http://downloads-distro.mongodb.org/repo/ubuntu-upstart dist 10gen' | tee /etc/apt/sources.list.d/mongodb.list
RUN apt-get update
RUN apt-get install -y mongodb-org

# 设置工作目录
WORKDIR /srv/full

# 清空已存在的文件（如果有）
RUN rm -rf /srv/full

# 通过 Git 下载准备好的 MEAN 架构的网站代码
RUN git clone https://github.com/chuyik/fullstack-demo-dist.git .

# 安装 NodeJS 依赖库
RUN npm install --production

# 创建 mongodb 数据文件夹
RUN mkdir -p /data/db

# 暴露端口（分别是 NodeJS 应用和 Mongodb）
EXPOSE 5566 27017

# 设置 NodeJS 应用环境变量
ENV NODE_ENV=production PORT=5566

# 添加启动脚本
ADD start.sh /tmp/
RUN chmod +x /tmp/start.sh

# 设置启动时默认运行命令
CMD ["bash", "/tmp/start.sh"]
```

* 创建 start.sh 启动脚本

```shell
# 后台启动 Mongodb
mongod --fork --logpath=/var/log/mongo.log --logappend

# 运行 NodeJS 应用
npm start
```

#### 构建镜像

```shell
# 通过该命令，按照 Dockerfile 所配置的信息构建出镜像
  docker build --rm -t node-full .

# 检查镜像是否创建成功
  docker images
```


#### 运行镜像

```shell
# 运行刚刚创建的镜像
# -p 设置端口，格式为「主机端口:容器端口」
  docker run -p 5566:5566 node-full
```


#### 访问应用
可以用浏览器访问 http://localhost:5566, 或运行 curl -s http://localhost:5566。



### 保存 Mongodb 数据文件
由于 Mongodb 服务运行在 Docker 容器 (container) 中，所以数据也在里面，但这并不利于数据管理和保存。因此，可以通过一些方法，将 Mongodb 数据文件保存在容器的外头。


#### 磁盘映射
这个是最简单的方式，在 docker run 命令当中，就有磁盘映射的参数 -v。
```shell
# -v 磁盘映射，格式为「主机目录:容器目录」
docker run -p 5566:5566 -v /var/mongodata:/data/db node-full
```
但这个命令在 Mac 和 Windows 中执行失败，因为 boot2docker 的虚拟机不支持。
所以，可以将数据保存在 boot2docker 内，并设置共享文件夹便于 Mac 或 Windows 访问。


