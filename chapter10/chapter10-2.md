## HTTP 2/2

http 模块提供了两个函数 http.request 和 http.get，功能是作为客户端向 HTTP服务器发起请求。

这通常来实现自己的爬虫程序， 笔者自己写的一个爬取知乎的一个例子：https://github.com/yjhjstz/iZhihu


### GET 例子
```js
const http = require("http")
http.get('http://www.baidu.com', (res) => {
  console.log(`Got response: ${res.statusCode}`);
  // consume response body
  res.resume();
}).on('error', (e) => {
  console.log(`Got error: ${e.message}`);
});
```
上面的程序会返回一个200的状态码！

### HTTP Client


### 总结


### 参考


