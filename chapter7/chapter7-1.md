
## Event
> Node.js uses an event-driven, non-blocking I/O model that makes it lightweight and efficient.

这是Node.Js官网对自身的介绍,明确强调了Node.Js使用了一个事件驱动、非阻塞式 I/O 的模型,使其轻量又高效。

而且在Node中大量核心模块都使用了Event的机制,因此可以说是整个Node里最重要的模块之一.


### 涉及源码
- [lib/events.js](https://github.com/nodejs/node/blob/v6.0.0/lib/events.js)

### 观察者模式


![](https://upload.wikimedia.org/wikipedia/commons/thumb/8/8d/Observer.svg/854px-Observer.svg.png)

上图是 UML 的类图，

观察者模式是这样一种设计模式。一个被称作被观察者的对象，维护一组被称为观察者的对象，这些对象依赖于被观察者，被观察者自动将自身的状态的任何变化通知给它们。

当一个被观察者需要将一些变化通知给观察者的时候，它将采用广播的方式，这条广播可能包含特定于这条通知的一些数据。

使用观察者模式更深层次的动机是，当我们需要维护相关对象的一致性的时候，我们可以避免对象之间的紧密耦合。例如，一个对象可以通知另外一个对象，而不需要知道这个对象的信息。

### Event.js 实现
EventEmitter 允许我们注册一个或多个函数作为 listeners。 在特定的事件触发时被调用。如下图：
![](https://github.com/yjhjstz/deep-into-node/blob/master/chapter7/2016-05-09%2014.13.19.png)
#### listeners 存储
一般观察者的设计模式的实现逻辑是类似的，都是有一个类似map的结构，存储监听事件和回调函数的对应关系。
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
在 EventEmitter 类中，以 键 / 值 对的方式来存储事件名和对应的监听器。
你可以会好奇，为什么创建一个最简单的 键 / 值 对搞的这么复杂，简单的一个
`this._events = {};` 不就好咯。

是的，社区的最初实现是这样的，但随着 V8的升级，对 ES6支持的越来越完备，它的实现办法是使用一个空的构造函数，并且把这个构造的原型事先置空。

通过jsperf 比较两者的性能, 我们发现这种实现竟是简单实现性能的2倍！

#### 增加事件监听
addListener: 增加事件监听, on： addListener的别名，实际上是一样的。
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
实际使用复杂场景时，会出现对回调顺序的需求。L250,默认添加监听是在事件监听数组的末尾。L247-L248，`prepend`标记是否在事件数组的前部添加。

> 深入了解 https://github.com/nodejs/node/pull/6032

#### 删除事件监听
在 EventEmitter#removeListener 这个 API 的实现里，需要从存储的监听器数组中除去一个元素，我们首先想到的就是使用 Array#splice 这个 API ，即 arr.splice(i, 1) 。不过这个 API 所提供的功能过于多了，它支持去除自定义数量的元素，还支持向数组中添加自定义的元素。所以，源码中选择自己实现一个最小可用的：
```js
function spliceOne(list, index) {
  for (var i = index, k = i + 1, n = list.length; k < n; i += 1, k += 1)
    list[i] = list[k];
  list.pop();
}
```
性能是原生调用的1.5倍。

#### 事件触发
在事件触发时，监听器拥有的参数数量是任意的。
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
把不定参数的函数调用转变成固定参数的函数调用，且最多支持到三个参数。超过3个参数则调用`emitMany`.
结果不言而喻，我们还是比较下会差多少，以三个参数为例：
jsperf 显示的性能差距在1倍左右。

> 深入了解 https://github.com/iojs/io.js/pull/601



### event在node中的应用
#### 监控文件变化，通知感兴趣的观察者。

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

L1410, FSWatcher 对象继承 EventEmitter，使自身有了EventEmitter的方法。
L1404， 当底层发生错误时，会发出通知事件 `error`。
L1406， 文件发生变化时，FSWatcher 对象发射 `change`事件，具体的变化由 *event*标识，*filename*标识文件名。

L1396, 挂在`FSEvent`对象上的方法 `onchange`作为 C++调用 Javascript 的回调，在不同的平台实现方式也不一样，
我们在文件系统章节将详细讲述。

上述是 fs 模块监听文件变化的实现，并导出API: `fs.watch()` 给外部使用，另外还有一个 `fs.watchFile()`。
我们查看官方文档：

> fs.watchFile(filename, [options], listener)

> Stability: 2 - Unstable. Use fs.watch instead, if available.

> Watch for changes on filename.

> fs.watch(filename, [options], [listener])

> Stability: 2 - Unstable. Not available on all platforms.

- fs.watch() 官方建议使用。
- fs.watch() 并不是全平台支持，只有 OSX 和 Windows 支持recursive选项。
- fs.watch() 监听文件或目录， fs.watchFile() 监听文件。


fs.watch() 如果传入 listener, 如下：
```js
fs.watch('somedir', function (event, filename) {
  console.log('event is: ' + event);
  if (filename) {
    console.log('filename provided: ' + filename);
  }
});
```
则默认添加函数 callback 到 `change`事件的观察者中。当然也可以换个姿势，如：

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
可以实现链式调用, 比如符合目前很火的Reactive Programming。
RP编程范式提高了编码的抽象程度，你可以更好地关注在商业逻辑中各种事件的联系避免大量细节而琐碎的实现，使得编码更加简洁。

#### 逐行读取 (Readline)

我们来看看逐行读取对键盘输入的处理， 这涉及到比较复杂的状态机和事件发送，是学习事件模块非常好的一个例子。

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
如果没有预先设定指定的query，然后用户应答后触发指定的callback，那么 `Interface`对象会触发 `line`事件。
在 input 流接受了一个 `\n` 时触发，通常在用户敲击回车或者返回时接收。 这是一个监听用户输入的利器。
监听 line 事件的示例:

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

该模块对复合功能按键，比如 Ctrl + c, Ctrl + z也做了相应的处理, 我们拿对 Ctrl + c 的代码进行分析：
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
- L681-L682, 忽略 `ESC` 键。
- L684, 首先判断是否是 Ctrl 和 Shift复合键同时按下，如果是则L685-L694优先处理。
- L696, 如果是按下 Ctrl 键，L699 继续判断，如果另一个是 `c` , 默认是关闭对象。 
- L701, 如果外部有观察者, 则发送 `SIGINT`事件，交由观察者处理。


#### REPL
一个 Read-Eval-Print-Loop（REPL，读取-执行-输出循环）既可用于独立程序也可很容易地被集成到其它程序中。REPL 提供了一种交互地执行 JavaScript 并查看输出的方式。它可以被用作调试、测试或仅仅尝试某些东西。

在命令行中不带任何参数执行 node 您便会进入 REPL。它提供了一个简单的 Emacs 行编辑。

REPLServer 继承 Interface，如代码所示： `inherits(REPLServer, rl.Interface);`

并监听 line 事件, 自定义关键字，以支持交互式的命令。
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

我们看下代码实现：
```js
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
- L400， 通过设置环境变量NODE_DEBUG=REPL打开调试功能。
- L407， 解析 cmd 输入， 处理正则的情况。
- L412, 查看是否以 `.`开头，并且不是浮点数，则利用正则匹配字符串，
   - 以 .help 为例，得到的 `matches` 为 `[ '.help', 'help', '', index: 0, input: '.help' ]`，
     keyword 为 help, rest 为 ''.
- L416, 通过 keyword 从 commands 对象找到对应的方法执行。



#### REPL实例
一个在curl(1)上运行的REPL实例的例子可以查看这里： https://gist.github.com/2053342



### EventEmitter vs Callbacks
- EventEmitter
  - 可以通知多个listeners
  - 一般被调用多次。
- Callback
  - 最多通知一个listener
  - 通常被调用一次，无论操作是成功还是失败。

### 总结
Event 模块是观察者设计模式的典型应用。同时也是Reactive Programming的精髓所在。


### 参考
[1].https://segmentfault.com/a/1190000005051034

