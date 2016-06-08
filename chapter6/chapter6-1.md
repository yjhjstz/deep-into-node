
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



### slab 分配
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

PS: 在 lib/_tls_legacy.js 中， `SlabBuffer` 创建了一个 10MB 的 slab。

```js
function alignPool() {
  // Ensure aligned slices
  if (poolOffset & 0x7) {
   poolOffset |= 0x7;
   poolOffset++;
  }
}
```
这里做了8字节的内存对齐处理。
* 如果不按照平台要求对数据存放进行对齐，会带来存取效率上的损失。比如32位的Intel处理器通过总线访问(包括读和写)内存数据。每个总线周期从偶地址开始访问32位内存数据，内存数据以字节为单位存放。如果一个32位的数据没有存放在4字节整除的内存地址处，那么处理器就需要2个总线周期对其进行访问，显然访问效率下降很多。 

* Node.js 是一个跨平台的语言，第三方的C++ addon 也是非常多，避免破坏了第三方模块的使用，比如 directIO 就必须要内存对齐。

* 兼容 node.js v0.10

> 详细：https://github.com/nodejs/node/pull/2487

### 浅拷贝
Buffer更像是可以做指针操作的C语言数组。例如，可以用[index]方式直接修改某个位置的字节。
需要注意的是：Buffer#slice 方法， 不是返回一个新的Buffer， 而是返回对原 Buffer 某个区间数值的引用。
```js
const buf1 = Buffer.allocUnsafe(26);

for (var i = 0 ; i < 26 ; i++) {
  buf1[i] = i + 97; // 97 is ASCII a
}

const buf2 = buf1.slice(0, 3);
buf2.toString('ascii', 0, buf2.length);
  // Returns: 'abc'
buf1[0] = 33;
buf2.toString('ascii', 0, buf2.length);
  // Returns : '!bc'

```

上面是官方 API 提供的例子， `buf2`是对 `buf1`前3个字节的引用，对 `buf2`的修改就相当于作用在 `buf1`上。


### 深拷贝
如果想要拷贝一份Buffer，得首先创建一个新的Buffer，并通过.copy方法把原Buffer中的数据复制过去。

```js
const buf1 = Buffer.allocUnsafe(26);
const buf2 = Buffer.allocUnsafe(26).fill('!');

for (let i = 0 ; i < 26 ; i++) {
  buf1[i] = i + 97; // 97 is ASCII a
}

buf1.copy(buf2, 8, 16, 20);
console.log(buf2.toString('ascii', 0, 25));
  // Prints: !!!!!!!!qrst!!!!!!!!!!!!!
```

通过深拷贝的方式，`buf2` 截取了 `buf1` 的部分内容，之后对 `buf2`的修改并不会作用于 `buf1`, 两者内容独立不共享。

需要注意的事：深拷贝是一种消耗 CPU 和内存的操作，请知道自己在做什么。


### 内存碎片

动态分配将不可避免会产生内存碎片的问题，那么什么是内存碎片？
内存碎片即“碎片的内存”描述一个系统中所有不可用的空闲内存，这些碎片之所以不能被使用，是因为负责动态分配内存的分配算法使得这些空闲的内存无法使用。

上述的 slab 分配，存在明显的内存碎片，即 8KB 的内存并没有完全被使用，存在一定的浪费。通用的slab实现，会浪费约1/2的空间。

当然存在更高效，更省内存的内存管理分配，比如 tcmalloc, 但也必须承受一定的管理代价。node.js 在这方面并没有一味的执着于此，而是达到一种性能与空间使用的平衡。

### zero fill

Node.js 在v5.10.0 加入了命令行选项 `--zero-fill-buffers`, 强制在申请 `Buffer`时用0填充分配的内存。

为什么要引入这个特性呢？

- 防止你代码中本该初始化的地方没有初始化；
- 防止其他代码访问到你之前写入 Buffer 的数据， 这边存在安全隐患, 如下
```js
✗ node -p "new Buffer(1024).toString('ascii')"
`7(@ P
xn?_k7x0x0' @#k
              :ArrayBuffer kh
                             &7;?m@bFn?_ @`` n?0'h2R'Lq083~C[e;@string k (R!~!H3kl
 ```

代码实现上则是通过 `--zero-fill-buffers` 区分申请内存是用 `malloc()`或者 `calloc()`。

当然性能上 `calloc()` 还是差很多，所以社区开放了一个选项，而不是默认开启。


* here are benchmark results for allocating a 1mb Buffer:



 xxx | v5.4.0	| v4.2.3 |	v0.12.9	| v0.10.41
 --- | --------  | -------| ---------|---------
new Buffer | 41,515 ops/sec ±3.00% | 43,670 ops/sec ±1.86% | 53,969 ops/sec ±1.41% | 147,350 ops/sec ±1.82% 
new Buffer (zero-filled) | 5,041 ops/sec ±2.00% | 4,293 ops/sec ±1.79% | 7,953 ops/sec ±0.55% | 8,167 ops/sec ±2.38% 



> 具体了解：https://github.com/nodejs/node/issues/4660 

### 总结
Buffer是一个典型的Javascript和C++结合的模块，性能相关部分用C++实现，非性能相关部分用javascript实现。

Node在进程启动时Buffer就已经加装进入内存，并将其放入全局对象，因此无需require。


Buffer内存分配，Buffer对象的内存分配不是在V8的堆内存中，在Node的C++层面实现内存的申请。


### 参考

* https://nodesource.com/blog/nsolid-deepdive-into-security-policies-zero-fill-buffer-allocations/

