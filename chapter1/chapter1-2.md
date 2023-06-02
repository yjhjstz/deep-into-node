
### Starting with "Hello World"

Let's start with a piece of code that is familiar to everyone, https://nodejs.org/en/about/. Like learning any language, a simple "Hello World" program can give you an intuitive understanding of node.js.

```js
const http = require('http');
const hostname = '127.0.0.1';
const port = 1337;

http.createServer((req, res) => {
  res.writeHead(200, { 'Content-Type': 'text/plain' });
  res.end('Hello World\n');
}).listen(port, hostname, () => {
  console.log(`Server running at http://${hostname}:${port}/`);
});
```

Let's take a look at how many core modules are involved in the first line of code, and start our journey of analyzing node.js source code.

The first line: `const http = require('http');` involves two modules, the module and http modules.

Main code
```js
http.createServer((req, res) => {
  res.writeHead(200, { 'Content-Type': 'text/plain' });
  res.end('Hello World\n');
}).listen(port, hostname, () => {
  ...
});
```

- First, let's understand the inheritance relationship of the HTTP Server, which is helpful for a better understanding of the code.

![http](270064-edbf9b53812f0433.png)

This also involves the event and net modules.

The last sentence:
```js
console.log(`Server running at http://${hostname}:${port}/`);
```
Here, the console module is used, but it is not obtained through `require`. This brings us to the global object, the top-level object of Node.js. I'll leave it as a teaser for now, and I'll explain it in detail in the global section later.

If you want to view some debugging logs of node, you can set the NODE_DEBUG environment variable, for example:
```shell
NODE_DEBUG=HTTP,STREAM,MODULE,NET node http.js
```
View V8 logs
```shell
node --trace http.js
```

### Summary
A simple hello world program involves multiple modules, but behind it is the wisdom of the Node community, a high-level abstraction of web services and asynchronous IO. As the saying goes, simplicity is the ultimate sophistication!

Let's start the journey of Node.js source code with a famous quote from Linus Torvalds.

> Talk is cheap, show me the code.


