
## Event
> Node.js uses an event-driven, non-blocking I/O model that makes it lightweight and efficient.

Node.js emphasizes its event-driven, non-blocking I/O model, which makes it lightweight and efficient. This event-driven model is used extensively in many core modules of Node.js, making it one of the most important modules in the entire Node.js ecosystem.



### Related Source Code
- [lib/events.js](https://github.com/nodejs/node/blob/v6.0.0/lib/events.js)

### Observer Pattern

![](https://upload.wikimedia.org/wikipedia/commons/thumb/8/8d/Observer.svg/854px-Observer.svg.png)

The above diagram is a UML class diagram.

The observer pattern is a design pattern in which an object, called the subject, maintains a list of its dependents, called observers, and notifies them automatically of any state changes, usually by calling one of their methods. 

When a subject needs to notify observers about something interesting happening, it broadcasts a notification to the observers (which can include specific data related to the topic of the notification).

The deeper motivation for using the observer pattern is to avoid tight coupling between objects when we need to maintain consistency between related objects. For example, an object can notify another object without knowing anything about that object.



### Event.js Implementation
EventEmitter allows us to register one or more functions as listeners, which are called when a specific event is triggered. As shown in the following diagram:
![](https://github.com/yjhjstz/deep-into-node/blob/master/chapter7/2016-05-09%2014.13.19.png)
#### Listeners Storage
The implementation logic of the observer design pattern is generally similar, with a map-like structure that stores the corresponding relationship between the listening event and the callback function.

```js
// This constructor is used to store event handlers. Instantiating this is
// faster than explicitly calling `Object.create(null)` to get a "clean" empty
// object (tested with v8 v4.9).
function EventHandlers() {}
EventHandlers.prototype = Object.create(null);
EventEmitter.init = function() {
  ...
  if (!this._events || this._events === Object.getPrototypeOf(this)._events) {
    this._events = new EventHandlers();
    this._eventsCount = 0;
  }

  this._maxListeners = this._maxListeners || undefined;
};
```
In the EventEmitter class, events and their corresponding listeners are stored as key-value pairs. You may wonder why creating a simple key-value pair is so complicated, and why not just use `this._events = {};`.

Indeed, the initial implementation in the community was like this, but with the upgrade of V8 and the increasing support for ES6, the implementation method is to use an empty constructor and pre-set the prototype of this constructor to null.

Through jsperf comparison of the two implementations, we found that this implementation is twice as fast as the simple implementation!



#### Add Event Listener
addListener: Add event listener, on: Alias of addListener, they are actually the same.

```js
210 function _addListener(target, type, listener, prepend) {
211   var m;
212   var events;
213   var existing;
214 
215   if (typeof listener !== 'function')
216     throw new TypeError('"listener" argument must be a function');
217 
218   events = target._events;
219   if (!events) {
220     events = target._events = new EventHandlers();
221     target._eventsCount = 0;
222   } else {
223     ...
234   }
235 
236   if (!existing) {
237     // Optimize the case of one listener. Don't need the extra array object.
238     existing = events[type] = listener;
239     ++target._eventsCount;
240   } else {
241     if (typeof existing === 'function') {
242       // Adding the second element, need to change to array.
243       existing = events[type] = prepend ? [listener, existing] :
244                                           [existing, listener];
245     } else {
246       // If we've already got an array, just append.
247       if (prepend) {
248         existing.unshift(listener);
249       } else {
250         existing.push(listener);
251       }
252     }
253 
254     // Check for listener leak
255     ...
264   }
265 
266   return target;
267 }

```

When using in complex scenarios, there may be a need for callback order. L250, the default is to add the listener to the end of the event listener array. L247-L248, the `prepend` flag indicates whether to add to the front of the event array.

> Learn more at https://github.com/nodejs/node/pull/6032
#### Removing Event Listeners
In the implementation of the EventEmitter#removeListener API, we need to remove an element from the stored listener array. Our first thought is to use the Array#splice API, i.e. arr.splice(i, 1). However, this API provides too much functionality, supporting the removal of a custom number of elements and the addition of custom elements to the array. Therefore, the source code chooses to implement the minimum usable one:

```js
function spliceOne(list, index) {
  for (var i = index, k = i + 1, n = list.length; k < n; i += 1, k += 1)
    list[i] = list[k];
  list.pop();
}
```
The performance is 1.5 times faster than the native call.



#### Event Triggering
When an event is triggered, the number of arguments that the listener has is arbitrary.

```js
136 EventEmitter.prototype.emit = function emit(type) {
137   var er, handler, len, args, i, events, domain;
138   var needDomainExit = false;
139   var doError = (type === 'error');
140 
141   events = this._events;
142   ...
169 
170   handler = events[type];
171 
172   if (!handler)
173     return false;
174   ...
180   var isFn = typeof handler === 'function';
181   len = arguments.length;
182   switch (len) {
183     // fast cases
184     case 1:
185       emitNone(handler, isFn, this);
186       break;
187     case 2:
188       emitOne(handler, isFn, this, arguments[1]);
189       break;
190     case 3:
191       emitTwo(handler, isFn, this, arguments[1], arguments[2]);
192       break;
193     case 4:
194       emitThree(handler, isFn, this, arguments[1], arguments[2], arguments[3]);
195       break;
196     // slower
197     default:
198       args = new Array(len - 1);
199       for (i = 1; i < len; i++)
200         args[i - 1] = arguments[i];
201       emitMany(handler, isFn, this, args);
202   }
206   ...
207   return true;
```

Convert function calls with variable parameters into fixed-parameter function calls, and support up to three parameters. If there are more than 3 parameters, 'emitMany' is called. The result is self-evident, let's still compare how much worse it will be, taking three parameters as an example: jsperf shows a performance gap of about 1x.

> Learn more at https://github.com/iojs/io.js/pull/601



### The Application of Events in Node.js
#### Monitor file changes and notify interested observers.


```js
1389 function FSWatcher() {
1390   EventEmitter.call(this);
1391 
1392   var self = this;
1393   this._handle = new FSEvent();
1394   this._handle.owner = this;
1395 
1396   this._handle.onchange = function(status, event, filename) {
1397     if (status < 0) {
1398       self._handle.close();
1399       const error = !filename ?
1400           errnoException(status, 'Error watching file for changes:') :
1401           errnoException(status,
1402                          `Error watching file ${filename} for changes:`);
1403       error.filename = filename;
1404       self.emit('error', error);
1405     } else {
1406       self.emit('change', event, filename);
1407     }
1408   };
1409 }
1410 util.inherits(FSWatcher, EventEmitter);
```

L1410, FSWatcher object inherits EventEmitter, which gives it access to EventEmitter's methods.
L1404, When an error occurs in the underlying system, a notification event 'error' is emitted.
L1406, When a file changes, the FSWatcher object emits a 'change' event, with the specific change identified by *event* and the *filename* indicating the name of the file.

L1396, The method 'onchange' attached to the 'FSEvent' object serves as a callback for C++ to call Javascript, with different implementation methods on different platforms.
We will discuss this in detail in the file system chapter.

The above is the implementation of file change monitoring in the fs module, which exports the API: `fs.watch()` for external use, as well as `fs.watchFile()`.
Let's take a look at the official documentation:
```



> fs.watchFile(filename, [options], listener)

> Stability: 2 - Unstable. Use fs.watch instead, if available.

> Watch for changes on filename.

> fs.watch(filename, [options], [listener])

> Stability: 2 - Unstable. Not available on all platforms.

- fs.watch() is recommended by official documentation.
- fs.watch() is not available on all platforms, and only supports the 'recursive' option on OSX and Windows.
- fs.watch() is used to monitor files or directories, while fs.watchFile() is used to monitor files.

If a listener is passed to fs.watch(), like this:
```js
fs.watch('somedir', function (event, filename) {
  console.log('event is: ' + event);
  if (filename) {
    console.log('filename provided: ' + filename);
  }
});
```
then the callback function is added to the observers of the 'change' event by default. Of course, you can also use a different approach, such as:

```js
var watcher = fs.watch('somedir');
watcher.on('change', function (event, filename) {
  console.log('event is: ' + event);
  if (filename) {
    console.log('filename provided: ' + filename);
  }
}).on('error', function(error) {
  
})
```
This allows for chain calls, which is in line with the currently popular Reactive Programming paradigm.
The RP programming paradigm improves the abstraction level of coding, allowing you to better focus on the relationship between various events in business logic, avoiding a large number of trivial and tedious implementations, making coding more concise.

#### Reading Line by Line (Readline)

Let's take a look at how readline handles keyboard input, which involves a complex state machine and event sending, making it a great example for learning the event module.


```js
 212 Interface.prototype._onLine = function(line) {
 213   if (this._questionCallback) {
 214     var cb = this._questionCallback;
 215     this._questionCallback = null;
 216     this.setPrompt(this._oldPrompt);
 217     cb(line);
 218   } else {
 219     this.emit('line', line);
 220   }
 221 };
```
If no specific query is set in advance and a specified callback is triggered after the user responds, the `Interface` object will trigger the `line` event. This event is triggered when the input stream receives a `\n`, usually when the user hits enter or return. It is a powerful tool for listening to user input. An example of listening to the `line` event is:

```js
var readline = require('readline');
var rl = readline.createInterface({
  input: process.stdin,
  output: process.stdout
});
rl.on('line', function (cmd) {
  console.log('You just typed: '+ cmd);
});
```

This module also handles composite function keys, such as Ctrl + c and Ctrl + z. Let's analyze the code for Ctrl + c:

```js
 678 Interface.prototype._ttyWrite = function(s, key) {
 679   key = key || {};
 680 
 681   // Ignore escape key - Fixes #2876
 682   if (key.name == 'escape') return;
 683 
 684   if (key.ctrl && key.shift) {
 685     /* Control and shift pressed */
 686     switch (key.name) {
 687       case 'backspace':
 688         this._deleteLineLeft();
 689         break;
 690 
 691       case 'delete':
 692         this._deleteLineRight();
 693         break;
 694     }
 695 
 696   } else if (key.ctrl) {
 697     /* Control key pressed */
 698 
 699     switch (key.name) {
 700       case 'c':
 701         if (this.listenerCount('SIGINT') > 0) {
 702           this.emit('SIGINT');
 703         } else {
 704           // This readline instance is finished
 705           this.close();
 706         }
 707         break;
 708     省略...
 709 }
```
- L681-L682, Ignore the `ESC` key.
- L684, First, determine if the Ctrl and Shift composite keys are pressed at the same time. If so, L685-L694 are processed first.
- L696, If the Ctrl key is pressed, continue to judge at L699. If the other is `c`, the object is closed by default.
- L701, If there are external observers, send the `SIGINT` event to be handled by the observer.



#### REPL
A Read-Eval-Print-Loop (REPL) can be used for standalone programs or easily integrated into other programs. The REPL provides an interactive way to execute JavaScript and view output. It can be used for debugging, testing, or just trying something out.

When you execute node without any parameters in the command line, you will enter the REPL. It provides a simple Emacs line editor.

REPLServer inherits from Interface, as shown in the code: `inherits(REPLServer, rl.Interface);`

It listens for the line event and customizes keywords to support interactive commands.

```shell
$ NODE_DEBUG=REPL node
REPL 37391: line ".help"
break Sometimes you get stuck, this gets you out
clear Alias for .break
exit  Exit the repl
help  Show repl options
load  Load JS from a file into the REPL session
save  Save all evaluated commands in this REPL session to a file
```

Let's take a look at the code implementation:

```
399   self.on('line', function(cmd) {
 400     debug('line %j', cmd);
 401     sawSIGINT = false;
 402 
 403     // leading whitespaces in template literals should not be trimmed.
 404     if (self._inTemplateLiteral) {
 405       self._inTemplateLiteral = false;
 406     } else {
 407       cmd = self.lineParser.parseLine(cmd);
 408     }
 409 
 410     // Check to see if a REPL keyword was used. If it returns true,
 411     // display next prompt and return.
 412     if (cmd && cmd.charAt(0) === '.' && isNaN(parseFloat(cmd))) {
 413       var matches = cmd.match(/^\.([^\s]+)\s*(.*)$/);
 414       var keyword = matches && matches[1];
 415       var rest = matches && matches[2];
 416       if (self.parseREPLKeyword(keyword, rest) === true) {
 417         return;
 418       } else if (!self.bufferedCommand) {
 419         self.outputStream.write('Invalid REPL keyword\n');
 420         finish(null);
 421         return;
 422       }
 423     }
 424     ...
 425   }
```
- L400, Enable debugging by setting the environment variable NODE_DEBUG=REPL.
- L407, Parse the input cmd and handle regular expressions.
- L412, Check if the input starts with a `.` and is not a floating point number. If so, use regular expressions to match the string.
  - For example, for `.help`, `matches` will be `[ '.help', 'help', '', index: 0, input: '.help' ]`, where keyword is `help` and rest is an empty string.
- L416, Find the corresponding method from the `commands` object using the keyword and execute it.



#### Example of a REPL Instance
An example of a REPL instance running on curl(1) can be found here: https://gist.github.com/2053342


### EventEmitter vs Callbacks
- EventEmitter
  - Can notify multiple listeners
  - Generally called multiple times.
- Callback
  - Can notify at most one listener
  - Usually called once, regardless of whether the operation is successful or not.

### Summary
The Event module is a typical application of the Observer design pattern. It is also the essence of Reactive Programming.

### References
[1].https://segmentfault.com/a/1190000005051034

