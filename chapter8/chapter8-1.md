
## Stream 流

在Node stream的接口中，包含可读流和可写流。有些流是即可读又可写。


### 可读流

* Files
 - fs.createReadStream(path, [options])
* HTTP (Server)
 - http.ServerRequest
* HTTP (Client)
  -http.ClientResponse
* TCP
  - net.Socket
* Child process
 - child.stdout
 - child.stderr
* Process
 - process.stdin

可读流可以触发的事件有：

* Event
 - 'data'
 - 'end'
 - 'error'


### 可写流

* Files
 - fs.createWriteStream(path, [options])
* HTTP (Server)
 - http.ServerResponse
* HTTP (Client)
 - http.ClientRequest
* TCP
 - net.Socket
* Child process
 - child.stdin
* Process
 - process.stdout
 - process.stderr

可写流可以触发的事件包括：
* Event
 - 'drain'
 - 'error'