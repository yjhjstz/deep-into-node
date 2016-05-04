
## Buffer


### 
在Node.js中，Buffer类是随Node内核一起发布的核心库。Buffer库为Node.js带来了一种存储原始数据的方法，可以让Nodejs处理二进制数据，每当需要在Nodejs中处理I/O操作中移动的数据时，就有可能使用Buffer库。原始数据存储在 Buffer 类的实例中。一个 Buffer 类似于一个整数数组，但它对应于 V8 堆内存之外的一块原始内存。

Buffer 和 Javascript 字符串对象之间的转换需要显式地调用编码方法来完成。以下是几种不同的字符串编码：

- ‘ascii’ – 仅用于 7 位 ASCII 字符。这种编码方法非常快，并且会丢弃高位数据。
- ‘utf8’ – 多字节编码的 Unicode 字符。许多网页和其他文件格式使用 UTF-8。
- ‘ucs2’ – 两个字节，以小尾字节序(little-endian)编码的 Unicode 字符。它只能对 BMP（基本多文种平面，U+0000 – U+FFFF） 范围内的字符编码。
- ‘base64’ – Base64 字符串编码。
- ‘binary’ – 一种将原始二进制数据转换成字符串的编码方式，仅使用每个字符的前 8 位。这种编码方法已经过时，应当尽可能地使用 Buffer 对象。
- 'hex' - 每个字节都采用 2 进制编码。

在Buffer中创建一个数组，需要注意以下规则：
- Buffer 是内存拷贝，而不是内存共享。

- Buffer 占用内存被解释为一个数组，而不是字节数组。比如，new Uint32Array(new Buffer([1,2,3,4])) 创建了4个 Uint32Array，它的成员为 [1,2,3,4] ,而不是[0x1020304] 或 [0x4030201]。



### 
在 lib/buffer.js 模块中，有个模块私有变量 pool， 它指向当前的一个8K 的slab ：

```js
Buffer.poolSize = 8 * 1024;
var pool;

function allocPool() {
  pool = new SlowBuffer(Buffer.poolSize);
  pool.used = 0;
}
```
SlowBuffer 为 src/node_buffer.cc 导出，当用户调用new Buffer时 ，如果你要申请的空间大于8K，node 会直接调用SlowBuffer ，如果小于8K ，新的Buffer 会建立在当前slab 之上：

- 新创建的Buffer的 parent成员变量会指向这个slab ，
- offset 变量指向在这个slab 中的偏移：
```js
if (!pool || pool.length - pool.used < this.length) allocPool();
this.parent = pool;
this.offset = pool.used;
pool.used += this.length;
```




