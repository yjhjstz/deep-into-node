
## Event

### 涉及源码
- lib/events.js

### 观察者模式



![](https://upload.wikimedia.org/wikipedia/commons/thumb/8/8d/Observer.svg/854px-Observer.svg.png)

上图是 UML 的类图，

观察者模式是这样一种设计模式。一个被称作被观察者的对象，维护一组被称为观察者的对象，这些对象依赖于被观察者，被观察者自动将自身的状态的任何变化通知给它们。

当一个被观察者需要将一些变化通知给观察者的时候，它将采用广播的方式，这条广播可能包含特定于这条通知的一些数据。

使用观察者模式更深层次的动机是，当我们需要维护相关对象的一致性的时候，我们可以避免对象之间的紧密耦合。例如，一个对象可以通知另外一个对象，而不需要知道这个对象的信息。

### Event.js 实现
   在 Node.js 中，流（stream）是许许多多原生对象的父类，角色可谓十分重要。但是，当我们沿着“族谱”往上看时，会发现 EventEmitter 类是流（stream）类的父类，所以可以说，EventEmitter 类是 Node.js 的根基类之一，地位可显一般。虽然 EventEmitter 类暴露的接口并不多而且十分简单，并且是少数纯 JavaScript 实现的模块之一，它的应用实在是太广泛，太基础了，在它的实现里处处闪光着一些优化代码的精髓。
   
#### 观察者存储
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



### node 场景
Nodejs的Events实现了一种观察者模式，其支持了Nodejs的核心机制，且http / fs / mongoose等都继承了Events，可以添加监听事件。