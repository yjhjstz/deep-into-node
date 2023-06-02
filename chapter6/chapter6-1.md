## Buffer

### 
In Node.js, the Buffer class is a core library that is released with the Node kernel. The Buffer library brings a way to store raw data in Node.js, allowing Node.js to handle binary data. Whenever it is necessary to move data in I/O operations in Node.js, the Buffer library can be used. The raw data is stored in an instance of the Buffer class. A Buffer is similar to an integer array, but it corresponds to a block of raw memory outside of the V8 heap.

Conversion between Buffer and JavaScript string objects requires an explicit call to an encoding method to complete. Here are several different string encodings:

- 'ascii' - Only for 7-bit ASCII characters. This encoding method is very fast and discards high-bit data.
- 'utf8' - Multi-byte encoded Unicode characters. Many web pages and other file formats use UTF-8.
- 'ucs2' - Two bytes, Unicode characters encoded in little-endian byte order. It can only encode characters within the BMP (Basic Multilingual Plane, U+0000 - U+FFFF) range.
- 'base64' - Base64 string encoding.
- 'binary' - A way to convert raw binary data to a string, using only the first 8 bits of each character. This encoding method is deprecated and Buffer objects should be used instead.
- 'hex' - Each byte is encoded in binary.


When creating a Buffer, it is important to keep in mind the following rules:
- Buffers are copies of memory, not shared memory.
- Buffers occupy memory as an array, not as a byte array. For example, `new Uint32Array(new Buffer([1,2,3,4]))` creates four Uint32Array members with values of `[1,2,3,4]`, not `[0x1020304]` or `[0x4030201]`.





### Slab Allocation
In the lib/buffer.js module, there is a private variable called "pool" that points to the current 8K slab:


```js
Buffer.poolSize = 8 * 1024;
var pool;

function allocPool() {
  pool = new SlowBuffer(Buffer.poolSize);
  pool.used = 0;
}
```
SlowBuffer is exported in src/node_buffer.cc. When a user calls new Buffer, if the requested space is greater than 8K, Node.js will directly call SlowBuffer. If it is less than 8K, the new Buffer will be created on top of the current slab:

- The parent member variable of the newly created Buffer will point to this slab,
- The offset variable points to the offset in this slab:

```js
if (!pool || pool.length - pool.used < this.length) allocPool();
this.parent = pool;
this.offset = pool.used;
pool.used += this.length;
```

PS: In lib/_tls_legacy.js, `SlabBuffer` creates a 10MB slab.

```js
function alignPool() {
  // Ensure aligned slices
  if (poolOffset & 0x7) {
   poolOffset |= 0x7;
   poolOffset++;
  }
}
```
Here is an 8-byte memory alignment processing.
* If data storage is not aligned according to platform requirements, it will result in efficiency losses in access. For example, a 32-bit Intel processor accesses (including read and write) memory data through a bus. Each bus cycle accesses 32-bit memory data starting from an even address, and memory data is stored in bytes. If a 32-bit data is not stored at a memory address that is a multiple of 4 bytes, the processor needs two bus cycles to access it, which obviously reduces access efficiency.

* Node.js is a cross-platform language, and there are also many third-party C++ addons. Avoiding damage to the use of third-party modules, such as directIO, requires memory alignment.

* Compatible with node.js v0.10

> 详细：https://github.com/nodejs/node/pull/2487

### Shallow copy
Buffer is more like a C language array that can do pointer operations. For example, you can directly modify a byte at a certain position using [index].
It should be noted that the Buffer#slice method does not return a new Buffer, but returns a reference to a certain range of values in the original Buffer.
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

The above is an example provided by the official API. `buf2` is a reference to the first 3 bytes of `buf1`, and modifying `buf2` is equivalent to acting on `buf1`.


### Deep copy
If you want to copy a Buffer, you must first create a new Buffer and copy the data from the original Buffer using the .copy method.

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

By deep copying, `buf2` intercepts part of the content of `buf1`, and subsequent modifications to `buf2` will not affect `buf1`, and the contents of the two are independent and not shared.

One thing to note: deep copying is a CPU and memory-consuming operation, so please know what you are doing.


### Memory fragmentation

Dynamic allocation will inevitably cause memory fragmentation. So what is memory fragmentation?
Memory fragmentation describes all unavailable free memory in a system. These fragments cannot be used because the allocation algorithm responsible for dynamic memory allocation makes these free memory unusable.

The above slab allocation has obvious memory fragmentation, that is, 8KB of memory is not fully used, and there is a certain waste. The general slab implementation wastes about 1/2 of the space.

Of course, there are more efficient and memory-saving memory management allocations, such as tcmalloc, but they must also bear a certain management cost. Node.js does not blindly persist in this regard, but achieves a balance between performance and space usage.

### zero fill

Node.js introduced the command line option `--zero-fill-buffers` in v5.10.0, which forces the allocated memory to be filled with 0 when requesting `Buffer`.

Why introduce this feature?

- To prevent uninitialized memory in your code.
- To prevent other code from accessing data that was written to the Buffer before, which can cause security vulnerabilities, as shown below.

```js
✗ node -p "new Buffer(1024).toString('ascii')"
`7(@ P
xn?_k7x0x0' @#k
              :ArrayBuffer kh
                             &7;?m@bFn?_ @`` n?0'h2R'Lq083~C[e;@string k (R!~!H3kl
 ```

The implementation is distinguished between using `malloc()` or `calloc()` to allocate memory by using the `--zero-fill-buffers` command line option.

Of course, `calloc()` is still much worse in performance, so the community has opened an option instead of enabling it by default.

* Here are benchmark results for allocating a 1mb Buffer:




 xxx | v5.4.0	| v4.2.3 |	v0.12.9	| v0.10.41
 --- | --------  | -------| ---------|---------
new Buffer | 41,515 ops/sec ±3.00% | 43,670 ops/sec ±1.86% | 53,969 ops/sec ±1.41% | 147,350 ops/sec ±1.82% 
new Buffer (zero-filled) | 5,041 ops/sec ±2.00% | 4,293 ops/sec ±1.79% | 7,953 ops/sec ±0.55% | 8,167 ops/sec ±2.38% 



> For detail：https://github.com/nodejs/node/issues/4660 

### Summary
Buffer is a typical module that combines Javascript and C++. The performance-related parts are implemented in C++, while the non-performance-related parts are implemented in Javascript.

Node loads Buffer into memory and puts it into the global object when the process starts, so there is no need to require it.

Buffer memory allocation: The memory allocation of Buffer objects is not in the V8 heap memory, but in the C++ layer of Node.

### Reference
* https://nodesource.com/blog/nsolid-deepdive-into-security-policies-zero-fill-buffer-allocations/

