## FAQ 


### 同步 require()

```js
// Native extension for .js
Module._extensions['.js'] = function(module, filename) {
  var content = fs.readFileSync(filename, 'utf8');
  module._compile(internalModule.stripBOM(content), filename);
};
```
大家可能都有疑问：为什么会选择使用同步而不用异步实现呢？

之所以同步是 Node.js 所遵循的 CommonJS 的模块规范要求的, 具体来说

在当年，CommonJS 社区对此就有很多争议，导致了坚持异步的 AMD 从 CommonJS 中分裂出来。

CommonJS 模块是同步加载和同步执行，AMD 模块是异步加载和异步执行，CMD（Sea.js）模块是异步加载和同步执行。ES6 的模块体系最后选择的是异步加载和同步执行。也就是 Sea.js 的行为是最接近 ES6 模块的。不过 Sea.js 这样做是需要付出代价的——需要扫描代码提取依赖，所以它不像 CommonJS/AMD 是纯运行时的模块系统。

注意 Sea.js 是 2010年之后开发的，提出 CMD 更晚。Node.js 当年（2009年）只有 CommonJS 和 AMD 两个选择。就算当时已经有 CMD 的等价提案，从性能角度出发，Node.js 不太可能选择需要静态分析开销的 类 CMD 方案。考虑到 Node.js 的模块是来自于本地文件系统，最后 Node.js 选择了看上去更简单的 CommonJS 模块规范，直到今天。


* 从模块规范的角度来看，依赖的同步获取是几乎所有模块机制的首选，是符合由无数的语言奠定的开发者的直觉。

* 从模块本身的特性来说的，结论就是使用异步的require收益很小，同时对开发者并不友好。

### fs.realpath 缓存
如今的 `realpath`的实现变得非常简洁, 直接调用系统调用realpath。
```js
fs.realpath = function realpath(path, options, callback) {
  if (!options) {
    options = {};
  } else if (typeof options === 'function') {
    callback = options;
    options = {};
  } else if (typeof options === 'string') {
    options = {encoding: options};
  } else if (typeof options !== 'object') {
    throw new TypeError('"options" must be a string or an object');
  }
  callback = makeCallback(callback);
  if (!nullCheck(path, callback))
    return;
  var req = new FSReqWrap();
  req.oncomplete = callback;
  binding.realpath(pathModule._makeLong(path), options.encoding, req);
  return;
};
```

大家可能又有疑问了， 原本提升性能的路径缓存去哪里了，不是说缓存都是提升性能的重要手段吗？

社区的修改可以在 https://github.com/nodejs/node/pull/3594 看到，
>   fs: optimize realpath using uv_fs_realpath()
    
>   Remove realpath() and realpathSync() cache.
>   Use the native uv_fs_realpath() which is faster
>   then the JS implementation by a few orders of magnitude

去掉了缓存反而提升了性能， 作者的 commit 提交也写的非常清楚：native uv_fs_realpath 实现要大大优于js层的实现，
但并没有说具体原因。


前面我已经提到过了文件系统的基本原理和大致实现，VFS中引入了高速磁盘缓存的机制，这属于一种软件机制，允许内核将原本存在磁盘上的某些信息保存在RAM中，以便对这些数据的进一步访问能快速进行，而不必慢速访问磁盘本身。
高速磁盘缓存可大致分为以下三种：
* 目录项高速缓存——主要存放的是描述文件系统路径名的目录项对象
* 索引节点高速缓存——主要存放的是描述磁盘索引节点的索引节点对象
* 页高速缓存——主要存放的是完整的数据页对象，每个页所包含的数据一定属于某个文件，同时，所有的文件读写操作都依赖于页高速缓存。其是Linux内核所使用的主要磁盘高速缓存。

`readpath` 的 native 实现的高性能得益于目录项高速缓存，有自身的淘汰机制，保持自身的高效的访问。其实缓存机制依然存在，只是下移到 VFS文件系统层面了。


### 流式读
nodejs的fs模块并没有提供一个copy的方法，但我们可以很容易的实现一个，比如：
```js
var source = fs.readFileSync('/path/to/source', {encoding: 'utf8'});
fs.writeFileSync('/path/to/dest', source);
```
这种方式是把文件内容全部读入内存，然后再写入文件，对于小型的文本文件，这没有多大问题。但是对于体积较大的二进制文件，比如音频、视频文件，动辄几个GB大小，如果使用这种方法，很容易使内存“爆仓”。具体的说，对于32位系统是1GB，64位是2GB。

理想的方法应该是读一部分，写一部分，不管文件有多大，只要时间允许，总会处理完成，这里就需要用到流的概念。

上面的文件复制可以简单实现一下：
```js
// pipe自动调用了data,end等事件
fs.createReadStream('/path/to/source').pipe(fs.createWriteStream('/path/to/dest'));
```
源文件通过管道自动流向了目标文件。


### 总结
- 不要迷信异步， 使用时评估同步和异步的开销，包括复杂度和性能。

- 缓存策略需要综合考虑，这离不开对系统的了解(更多的涉猎)，重复缓存只会带来没必要的开销。

- 大文件的操作，使用流式操作。


### 参考
* https://www.zhihu.com/question/38041375


