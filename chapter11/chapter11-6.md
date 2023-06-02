
## FAQ 


### Synchronous require()

```js
// Native extension for .js
Module._extensions['.js'] = function(module, filename) {
  var content = fs.readFileSync(filename, 'utf8');
  module._compile(internalModule.stripBOM(content), filename);
};
```
Many people may wonder why synchronous is used instead of asynchronous implementation?

The reason why synchronous is used is because it is required by the CommonJS module specification followed by Node.js. Specifically,

At that time, the CommonJS community had a lot of controversy over this, which led to the split of AMD, which insisted on asynchronous, from CommonJS.

CommonJS modules are loaded and executed synchronously, AMD modules are loaded and executed asynchronously, and CMD (Sea.js) modules are loaded asynchronously and executed synchronously. The module system of ES6 finally chose asynchronous loading and synchronous execution. That is, Sea.js's behavior is closest to ES6 modules. However, Sea.js does this at a cost-it needs to scan the code to extract dependencies, so it is not a purely runtime module system like CommonJS/AMD.

Note that Sea.js was developed after 2010, and CMD was proposed later. At that time, Node.js had only two choices: CommonJS and AMD. Even if there were equivalent proposals for CMD at that time, from the perspective of performance, Node.js was unlikely to choose a class CMD scheme that requires static analysis overhead. Considering that Node.js modules come from the local file system, Node.js finally chose the seemingly simpler CommonJS module specification, until today.


* From the perspective of module specifications, synchronous acquisition of dependencies is the first choice for almost all module mechanisms, and it is in line with the intuition of developers laid by countless languages.

* From the perspective of the characteristics of the module itself, the conclusion is that the benefits of using asynchronous require are small, and it is not friendly to developers at the same time.

### fs.realpath cache
Nowadays, the implementation of `realpath` has become very concise, directly calling the system call realpath.
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

You may have another question, where did the path cache that improves performance go? Isn't caching an important means of improving performance?

The community's modifications can be seen at https://github.com/nodejs/node/pull/3594,
>   fs: optimize realpath using uv_fs_realpath()
    
>   Remove realpath() and realpathSync() cache.
>   Use the native uv_fs_realpath() which is faster
>   then the JS implementation by a few orders of magnitude

Removing the cache actually improves performance. The author's commit submission is also very clear: the native uv_fs_realpath implementation is much better than the js layer implementation, but the specific reason is not mentioned.


I have mentioned the basic principles and rough implementation of the file system before. The VFS introduces a mechanism of high-speed disk cache, which belongs to a software mechanism that allows the kernel to save some information that originally exists on the disk in RAM, so that further access to these data can be quickly performed without having to slow down. Access to the disk itself.
High-speed disk cache can be roughly divided into the following three types:
* Directory item high-speed cache-mainly stores directory item objects that describe file system path names
* Index node high-speed cache-mainly stores index node objects that describe disk index nodes
* Page high-speed cache-mainly stores complete data page objects. Each page contains data that belongs to a certain file. At the same time, all file read and write operations depend on the page high-speed cache. It is the main disk high-speed cache used by the Linux kernel.

The high performance of the native implementation of `readpath` is due to the directory item high-speed cache, which has its own elimination mechanism and maintains efficient access to itself. In fact, the caching mechanism still exists, but it has been moved down to the VFS file system level.


### Stream reading
The fs module of nodejs does not provide a copy method, but we can easily implement one, for example:
```js
var source = fs.readFileSync('/path/to/source', {encoding: 'utf8'});
fs.writeFileSync('/path/to/dest', source);
```
This method reads the entire file content into memory and then writes it to the file. For small text files, this is not a big problem. But for large binary files, such as audio and video files, which are often several GB in size, if this method is used, it is easy to make the memory "out of stock". Specifically, it is 1GB for 32-bit systems and 2GB for 64-bit systems.

The ideal method should read a part and write a part. No matter how large the file is, as long as time permits, it will be processed. Here we need to use the concept of stream.

The above file copy can be easily implemented as follows:
```js
// The pipe automatically calls data, end and other events
fs.createReadStream('/path/to/source').pipe(fs.createWriteStream('/path/to/dest'));
```
The source file automatically flows to the target file through the pipeline.



