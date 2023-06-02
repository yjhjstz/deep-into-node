
## Stream

Streams have been a part of the Unix programming environment since the early days, and over the past few decades they have proven to be a reliable way to break a large system into small, composable parts that work together perfectly. In Unix, we use the `|` symbol to implement streams. In Node, the built-in [stream module](http://nodejs.org/docs/latest/api/stream.html) has been used by multiple core modules and can also be used by user-defined modules. Like Unix, the basic operator of the stream module in Node is called `.pipe()`, and you can also use a backpressure mechanism to deal with objects that consume data slowly.

### Why Use Streams

In Node, I/O is asynchronous, so interacting with the disk and network involves passing callback functions. You may have written code like this before:

```js
	var http = require('http');
	var fs = require('fs');

	var server = http.createServer(function (req, res) {
	    fs.readFile(__dirname + 'data.txt', function (err, data) {
	        res.end(data);
	    });
	});
	server.listen(8000);
```

The above code is not problematic, but each time a request is made, we read the entire `data.txt` file into memory and then return the result to the client. Think about it, if the `data.txt` file is very large, when responding to a large number of concurrent user requests, the program may consume a lot of memory, which may cause slow user connections.

Secondly, the above code may cause a very bad user experience, because users need to wait for the program to read the file content into memory before receiving any content.

Fortunately, `(req, res)` parameters are stream objects, which means we can use a better way to achieve the above requirements:

```js
	var http = require('http');
	var fs = require('fs');

	var server = http.createServer(function (req, res) {
	    var stream = fs.createReadStream(__dirname + '/data.txt');
	    stream.pipe(res);
	});
	server.listen(8000);
```

Here, the `.pipe()` method will automatically help us listen for the `data` and `end` events. The above code is not only concise, but also sends each small piece of data in the `data.txt` file to the client continuously.

In addition, using the `.pipe()` method has other benefits, such as automatically controlling the backend pressure so that node can put as little cache as possible in memory when the client connection is slow.

Want to compress data? We can use the corresponding stream module to complete this task!

```js
	var http = require('http');
	var fs = require('fs');
	var oppressor = require('oppressor');

	var server = http.createServer(function (req, res) {
	    var stream = fs.createReadStream(__dirname + '/data.txt');
	    stream.pipe(oppressor(req)).pipe(res);
	});
	server.listen(8000);
```

With the above code, we successfully compressed the data sent to the browser using gzip. We just used an oppressor module to handle this.

Once you learn to use the stream API, you can assemble these stream modules like building Lego blocks or connecting water pipes, and you may never use modules without stream APIs to get and push data again.

### Stream Module Basics

Nodejs provides a total of 4 streams at the bottom, Readable stream, Writable stream, Duplex stream, and Transform stream.

Usage scenario |	Class |	Method to be rewritten
-------| ------| -------------
Read only |	Readable| 	_read
Write only	| Writable	| _write
Duplex	| Duplex |	_read, _write
Operate on written data and then read the result|	Transform|	_transform, _flush


#### pipe

Regardless of which stream, the `.pipe()` method is used to implement input and output.

The `.pipe()` function is very simple. It only accepts a source `src` and outputs the data to a writable stream `dst`:

	src.pipe(dst)

`.pipe(dst)` will return `dst`, so you can chain multiple streams:

	a.pipe(b).pipe(c).pipe(d)

The above code can also be equivalent to:

	a.pipe(b);
	b.pipe(c);
	c.pipe(d);
	
This is very similar to writing stream code in Unix:

	a | b | c | d

Except that you are writing in Node instead of in the shell!

### Readable Stream

A Readable stream can produce data, and you can send this data to a writable, transform, or duplex stream by calling the `pipe()` method:

	readableStream.pipe(dst)

#### Creating a Readable Stream

Now let's create a readable stream!

	var Readable = require('stream').Readable;

	var rs = new Readable;
	rs.push('beep ');
	rs.push('boop\n');
	rs.push(null);

	rs.pipe(process.stdout);

The effect of running the code is as follows:

	$ node read0.js
	beep boop

In the above code, the role of `rs.push(null)` is to tell `rs` that the output data should end.

One thing to note is that we have already pushed the content into the readable stream `rs` before outputting the data to `process.stdout`, but all the data is still writable.

This is because when you use `.push()` to push data into a readable stream, the data will be stored in a cache until another thing consumes the data.

However, in most cases, what we want is for the data to be generated only when it is needed, in order to avoid a large amount of cached data.

```
	var Readable = require('stream').Readable;
	var rs = Readable();

	var c = 97;
	rs._read = function () {
	    rs.push(String.fromCharCode(c++));
	    if (c > 'z'.charCodeAt(0)) rs.push(null);
	};

	rs.pipe(process.stdout);
```
The output of the code is as follows:

	$ node read1.js
	abcdefghijklmnopqrstuvwxyz

In the above code, the role of `rs.push(null)` is to tell `rs` that the output data should end.

One thing to note is that we have already pushed the content into the readable stream `rs` before outputting the data to `process.stdout`, but all the data is still writable.

This is because when you use `.push()` to push data into a readable stream, the data will be stored in a cache until another thing consumes the data.

`_read` function can also receive a `size` parameter to indicate how many bits of data the consumer wants to read, but this parameter is optional.

It should be noted that you can use `util.inherit()` to inherit a Readable stream.

To illustrate that the `_read` function is only called when the data consumer appears, we can simply modify the above code:

	var Readable = require('stream').Readable;
	var rs = Readable();

	var c = 97 - 1;

	rs._read = function () {
	    if (c >= 'z'.charCodeAt(0)) return rs.push(null);

	    setTimeout(function () {
	        rs.push(String.fromCharCode(++c));
	    }, 100);
	};

	rs.pipe(process.stdout);

	process.on('exit', function () {
	    console.error('\n_read() called ' + (c - 97) + ' times');
	});
	process.stdout.on('error', process.exit);
	
When running the above code, we can find that if we only request 5 bits of data, `_read` will only run 5 times:

	$ node read2.js | head -c5
	abcde
	_read() called 5 times
	
In the above code, `setTimeout` is very important because the operating system needs to spend some time sending the program end signal.

In addition, the `process.stdout.on('error',fn)` handler is also important because when `head` is no longer interested in our program output, the operating system will send a `SIGPIPE` signal to our process, and `process.stdout` will capture an `EPIPE` error at this time.

These complex parts are necessary in interactions with the operating system, but they are optional if you are interacting directly with streams in Node.

If you create a readable stream and want to push any value into it, make sure you specify the objectMode parameter when creating the stream, `Readable({ objectMode: true })`.

#### Consuming a Readable Stream

Most of the time, it is easy to pipe a readable stream directly into another type of stream or a stream created using through or concat-stream. But sometimes we also need to consume a readable stream directly.

	process.stdin.on('readable', function () {
	    var buf = process.stdin.read();
	    console.dir(buf);
	});
	
The output of the code is as follows:

	$ (echo abc; sleep 1; echo def; sleep 1; echo ghi) | node consume0.js 
	<Buffer 61 62 63 0a>
	<Buffer 64 65 66 0a>
	<Buffer 67 68 69 0a>
	null

When data is available, the `readable` event will be triggered, and you can call the `.read()` method to get this data from the cache.

When the stream ends, `.read()` will return `null` because there are no more bytes available for us to get.

You can also tell the `.read()` method to return `n` bytes of data. Although this method is available for all streams in the core objects, it is not available for object streams.


Here's an example where we specify that we want to read 3 bytes of data each time:

	process.stdin.on('readable', function () {
	    var buf = process.stdin.read(3);
	    console.dir(buf);
	});

When we run the above code, we will get incomplete data:

	$ (echo abc; sleep 1; echo def; sleep 1; echo ghi) | node consume1.js 
	<Buffer 61 62 63>
	<Buffer 0a 64 65>
	<Buffer 66 0a 67>

This is because the extra data is left in the internal buffer. Therefore, we need to tell Node that we are still interested in the remaining data. We can use `.read(0)` to accomplish this:

	process.stdin.on('readable', function () {
	    var buf = process.stdin.read(3);
	    console.dir(buf);
	    process.stdin.read(0);
	});

Now our code works as expected!

	$ (echo abc; sleep 1; echo def; sleep 1; echo ghi) | node consume2.js 
	<Buffer 61 62 63>
	<Buffer 0a 64 65>
	<Buffer 66 0a 67>
	<Buffer 68 69 0a>

We can also use the `.unshift()` method to put back the extra data.

Using the `unshift()` method can avoid unnecessary buffer copying. In the following code, we will create a readable parser that splits on newlines:

	var offset = 0;

	process.stdin.on('readable', function () {
	    var buf = process.stdin.read();
	    if (!buf) return;
	    for (; offset < buf.length; offset++) {
	        if (buf[offset] === 0x0a) {
	            console.dir(buf.slice(0, offset).toString());
	            buf = buf.slice(offset + 1);
	            offset = 0;
	            process.stdin.unshift(buf);
	            return;
	        }
	    }
	    process.stdin.unshift(buf);
	});

The output of the code is as follows:

	$ tail -n +50000 /usr/share/dict/american-english | head -n10 | node lines.js 
	'hearties'
	'heartiest'
	'heartily'
	'heartiness'
	'heartiness\'s'
	'heartland'
	'heartland\'s'
	'heartlands'
	'heartless'
	'heartlessly'

Of course, there are already many modules like `split` that can help you accomplish this, so you don't need to write one yourself.


### Writable Stream

A writable stream is a stream that can only be written to, not read from:

	src.pipe(writableStream)

#### Creating a Writable Stream

You only need to define a `._write(chunk, enc, next)` function to release data from a readable stream:

	var Writable = require('stream').Writable;
	var ws = Writable();
	ws._write = function (chunk, enc, next) {
	    console.dir(chunk);
	    next();
	};

	process.stdin.pipe(ws);

The output of the code is as follows:

	$ (echo beep; sleep 1; echo boop) | node write0.js 
	<Buffer 62 65 65 70 0a>
	<Buffer 62 6f 6f 70 0a>

The first parameter, `chunk`, represents the data written in.

The second parameter, `enc`, represents the encoding string, but you can only write a string when `opts.decodeString` is `false`.

The third parameter, `next(err)`, is a callback function. You can use this callback function to tell the data consumer that more data can be written. You can optionally pass an error object `error`, which will trigger an `emit` event on the stream entity.

In the process of transferring data from a readable stream to a writable stream, the data will be automatically converted to a `Buffer` object, unless you specify the `decodeStrings` parameter as `false` when creating the writable stream, `Writable({decodeStrings: false})`.

If you need to pass an object, you need to specify the `objectMode` parameter as `true`, `Writable({ objectMode: true })`.

#### Writing to a Writable Stream

If you need to write to a writable stream, just call `.write(data)`.

	process.stdout.write('beep boop\n');

To tell a writable stream that you have finished writing, just call the `.end()` method. You can also use `.end(data)` to write some more data before ending.

	var fs = require('fs');
	var ws = fs.createWriteStream('message.txt');

	ws.write('beep ');

	setTimeout(function () {
	    ws.end('boop\n');
	}, 1000);

The output of the code is as follows:

	$ node writing1.js 
	$ cat message.txt
	beep boop

If you specify the `highWaterMark` parameter when creating the writable stream, calling the `.write()` method will return false when there is no more data to write.

If you want to wait for the cache situation, you can listen for the `drain` event.



### Duplex Stream

A duplex stream is a stream that can be both read from and written to, full duplex. As shown in the figure:
![](http://3.bp.blogspot.com/-hWPHqV9RJlM/VnrEyChmtnI/AAAAAAAABpQ/uTnbBCU87ek/s1600/duplex.PNG)

In terms of code implementation:
```js
const Readable = require('_stream_readable');
const Writable = require('_stream_writable');

util.inherits(Duplex, Readable);

var keys = Object.keys(Writable.prototype);
for (var v = 0; v < keys.length; v++) {
  var method = keys[v];
  if (!Duplex.prototype[method])
    Duplex.prototype[method] = Writable.prototype[method];
}
```
`Duplex` first inherits `Readable`, because javascript does not have the multiple inheritance feature of C++, so
traverse the prototype methods of `Writable` and assign them to the prototype of `Duplex`.

### Transform Stream
Transform streams are duplex streams that transform input into output. They implement both the Readable and Writable interfaces.

The transform stream can be imagined as the middle part of a stream, which can be read and written, but does not store data. It is only responsible for processing the data that passes through it.

Node's transform streams include:
* zlib streams
* crypto streams

### Summary
The advantage of stream processing: dividing functions and combining them through pipelines.

### Reference
https://github.com/substack/stream-handbook


