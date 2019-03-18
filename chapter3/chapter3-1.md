
## Timer源码解读

### 涉及源码
* [lib/timers.js](https://github.com/nodejs/node/blob/master/lib/timers.js)
* [src/timer_wrap.cc](https://github.com/nodejs/node/blob/master/src/timer_wrap.cc)
* [deps/uv/src/unix/timer.c](https://github.com/nodejs/node/blob/master/deps/uv/src/unix/timer.c)
* [deps/uv/src/heap-inl.h](https://github.com/nodejs/node/blob/master/deps/uv/src/heap-inl.h)

主要分为 javascript 层面的实现和 `libuv` 层面的实现, 而timer_wrap.cc 作为一个bridge,完成 javascript 和 C++的交互调用。

### 使用场景
定时器主要的使用场景或者说适用场景：
* 定时任务，比如业务中定时检查状态等；
* 超时控制，比如网络超时控制重传。

在 node.js 的实现中，
```js
function responseOnEnd() {
    // 省略
    debug('AGENT socket keep-alive');
    if (req.timeoutCb) {
      socket.setTimeout(0, req.timeoutCb);
      req.timeoutCb = null;
    }
 }
````
你可能会有疑问：为啥在 HTTP 模块要用呢？

   我们知道HTTP协议采用“请求-应答”模式，当使用普通模式，即非KeepAlive模式时，每个请求/应答客户和服务器都要新建一个连接，完成之后立即断开连接（HTTP协议为无连接的协议）；当使用Keep-Alive模式（又称持久连接、连接重用）时，Keep-Alive功能使客户端到服务器端的连接持续有效，当出现对服务器的后继请求时，Keep-Alive功能避免了建立或者重新建立连接。

 ```js
  if (req.httpVersionMajor < 1 || req.httpVersionMinor < 1) {
    this.useChunkedEncodingByDefault = chunkExpression.test(req.headers.te);
    this.shouldKeepAlive = false;
  }
```
   HTTP/1.0中默认是关闭的，需要在http头加入"Connection: Keep-Alive"，才能启用Keep-Alive；http/1.1中默认启用Keep-Alive，如果加入"Connection: close"，才关闭。
 
  目前大部分浏览器都是用HTTP/1.1协议，也就是说默认都会发起Keep-Alive的连接请求了，Node.js 针对2种协议按上述代码做了判断处理。
    
   当然了，这个连接不能就这么一直保持着，所以一般都会有一个超时时间，超过这个时间客户端还没有发送新的http请求，那么服务器就需要自动断开从而继续为其他客户端提供服务。
   Node.js的HTTP 模块对于每一个新的连接创建一个 socket 对象，调用socket.setTimeout设置一个定时器用于超时后自动断开连接。

### 数据结构选择
一个Timer本质上是这样的一个数据结构：deadline越近的任务拥有越高优先级，提供以下3种基本操作：
* schedule 新增任务
* cancel 删除任务
* expire 执行到期的任务

实现方式 |	schedule | cancel	| expire 
-------| ----------| --------  | ------
基于链表 |	O(1) |	O(n)  | O(n)
基于排序链表 | O(n)|	O(1) | O(1)
基于最小堆|	O(lgn)|	O(1)  |	O(1)
基于时间轮|	O(1) |	O(1)  |	O(1)

timer 的实现历经变迁，每次变迁都是思维碰撞的火花，让我们走进源码，细细品味。


### libuv 实现
#### 数据结构-最小堆
最小堆首先是二叉堆，二叉堆是完全二元树或者是近似完全二元树，它分为两种：最大堆和最小堆。
最大堆：父结点的键值总是大于或等于任何一个子节点的键值；最小堆：父结点的键值总是小于或等于任何一个子节点的键值。示意图如下：
![binary-tree](http://images.cnitblog.com/i/497634/201403/182339209436216.jpg)

节点定义在deps/uv/src/heap-inl.h，如下:
```c
struct heap_node {
  struct heap_node* left;
  struct heap_node* right;
  struct heap_node* parent;
};
```
根节点定义：
```c
/* A binary min heap.  The usual properties hold: the root is the lowest
 * element in the set, the height of the tree is at most log2(nodes) and
 * it's always a complete binary tree.
 *
 * The heap function try hard to detect corrupted tree nodes at the cost
 * of a minor reduction in performance.  Compile with -DNDEBUG to disable.
 */
struct heap {
  struct heap_node* min;
  unsigned int nelts;
};
```
   这边我们可以清楚的看到，最小堆采用指针组织数据，而不是数组。`min`始终指向最小的节点如果存在的话。作为一个排序的集合，它还需要一个用户指定的比较函数，决定哪个节点更小，或者说当过期时间一样时，决定他们的次序。毕竟没有规则不成方圆。
```c
static int timer_less_than(const struct heap_node* ha,
                           const struct heap_node* hb) {
  const uv_timer_t* a;
  const uv_timer_t* b;

  a = container_of(ha, const uv_timer_t, heap_node);
  b = container_of(hb, const uv_timer_t, heap_node);

  if (a->timeout < b->timeout)
    return 1;
  if (b->timeout < a->timeout)
    return 0;

  /* Compare start_id when both have the same timeout. start_id is
   * allocated with loop->timer_counter in uv_timer_start().
   */
  if (a->start_id < b->start_id)
    return 1;
  if (b->start_id < a->start_id)
    return 0;

  return 0;
}
```

这边我们可以看到，首先比较两者的 `timeout` ，如果二者一样，则比较二者被`schedule`的 id, 该 id 由 `loop->timer_counter` 递增生成，在调用
`uv_timer_start`时赋值给`start_id`.



#### 具体实现
```c
 62 int uv_timer_start(uv_timer_t* handle,
 63                    uv_timer_cb cb,
 64                    uint64_t timeout,
 65                    uint64_t repeat) {
 66   uint64_t clamped_timeout;
 67 
 68   if (cb == NULL)
 69     return -EINVAL;
 70 
 71   if (uv__is_active(handle))
 72     uv_timer_stop(handle);
 73 
 74   clamped_timeout = handle->loop->time + timeout;
 75   if (clamped_timeout < timeout)
 76     clamped_timeout = (uint64_t) -1;
 77 
 78   handle->timer_cb = cb;
 79   handle->timeout = clamped_timeout;
 80   handle->repeat = repeat;
 81   /* start_id is the second index to be compared in uv__timer_cmp() */
 82   handle->start_id = handle->loop->timer_counter++;
 83 
 84   heap_insert((struct heap*) &handle->loop->timer_heap,
 85               (struct heap_node*) &handle->heap_node,
 86               timer_less_than);
 87   uv__handle_start(handle);
 88 
 89   return 0;
 90 }
```
* L68-L69, 做参数的检查，错误则返回 -EINVAL。
* L71-L72，如有是一个活跃的 timer, 则立即停止它。
* L74-L82, 参数赋值，上面提到的`start_id` 就是由`timer_counter`自增得到。
* L84-L86, 插入 timer 节点到最小堆，此处算法复杂度为 O(lgn)。
* L87， 标记句柄非活跃，并加入统计。

```c
 93 int uv_timer_stop(uv_timer_t* handle) {
 94   if (!uv__is_active(handle))
 95     return 0;
 96 
 97   heap_remove((struct heap*) &handle->loop->timer_heap,
 98               (struct heap_node*) &handle->heap_node,
 99               timer_less_than);
100   uv__handle_stop(handle);
101 
102   return 0;
103 }
```
L94，检查 handle, 如果是非活跃的，则说明没有启动过，则返回成功。
L97-L99, 从最小堆中删除 timer的节点。
L100, 重置句柄，并减少计数。

了解了如何开启和关闭一个定时器，我们看如何调度定时器。

```c
int uv_run(uv_loop_t* loop, uv_run_mode mode) {
  ...
  while (r != 0 && loop->stop_flag == 0) {
    uv__update_time(loop);
    uv__run_timers(loop);
    ran_pending = uv__run_pending(loop);
    ...
 }
```
在 node.js 的 event loop 中，更新时间后则立即调用`uv__run_timers`，可见 timer 作为一个外部系统依赖的模块，优先级是最高的。

```c
150 void uv__run_timers(uv_loop_t* loop) {
151   struct heap_node* heap_node;
152   uv_timer_t* handle;
153 
154   for (;;) {
155     heap_node = heap_min((struct heap*) &loop->timer_heap);
156     if (heap_node == NULL)
157       break;
158 
159     handle = container_of(heap_node, uv_timer_t, heap_node);
160     if (handle->timeout > loop->time)
161       break;
162 
163     uv_timer_stop(handle);
164     uv_timer_again(handle);
165     handle->timer_cb(handle);
166   }
167 }
```

L155-L157, 取出最小的timer节点,如果为空，则跳出循环。
L159-L161, 通过 heap_node 的偏移拿到对象的首地址，如果最小的 timeout时间大于当前的时间，则说明过期时间还没到，则退出循环。
L163-L165, 删除 timer, 如果是需要重复执行的定时器，则通过调用`uv_timer_again`再次加入, L165执行 timer的 callback 任务后循环。

#### 改进的分级时间轮实现
https://github.com/libuv/libuv/pull/823


### 桥接层
阅读此节需要node.js addon 的知识，这边默认你已经了解。
```c
 43     env->SetProtoMethod(constructor, "start", Start);
 44     env->SetProtoMethod(constructor, "stop", Stop);
 45 
 46     target->Set(FIXED_ONE_BYTE_STRING(env->isolate(), "Timer"),
 47                 constructor->GetFunction());
```
Timer 的 addon 导出`start`,`stop`的方法，供js 层调用。
```c++
 71   static void Start(const FunctionCallbackInfo<Value>& args) {
 72     TimerWrap* wrap = Unwrap<TimerWrap>(args.Holder());
 73 
 74     CHECK(HandleWrap::IsAlive(wrap));
 75 
 76     int64_t timeout = args[0]->IntegerValue();
 77     int64_t repeat = args[1]->IntegerValue();
 78     int err = uv_timer_start(&wrap->handle_, OnTimeout, timeout, repeat);
 79     args.GetReturnValue().Set(err);
 80   }
 81 
 82   static void Stop(const FunctionCallbackInfo<Value>& args) {
 83     TimerWrap* wrap = Unwrap<TimerWrap>(args.Holder());
 84 
 85     CHECK(HandleWrap::IsAlive(wrap));
 86 
 87     int err = uv_timer_stop(&wrap->handle_);
 88     args.GetReturnValue().Set(err);
 89   }
```

`Start`需要提供两个参数，1.超时时间 timeout; 2. 重复执行的周期。
L78 调用`uv_timer_start`,其中 `OnTimeout`是该定时器的回调函数。
我们看下该函数实现：
```c++
 91   static void OnTimeout(uv_timer_t* handle) {
 92     TimerWrap* wrap = static_cast<TimerWrap*>(handle->data);
 93     Environment* env = wrap->env();
 94     HandleScope handle_scope(env->isolate());
 95     Context::Scope context_scope(env->context());
 96     wrap->MakeCallback(kOnTimeout, 0, nullptr);
 97   }
```
你可能好奇，怎么就由 handle->data 取到对象指针了呢？
```c++
HandleWrap::HandleWrap(Environment* env,
                       Local<Object> object,
                       uv_handle_t* handle,
                       AsyncWrap::ProviderType provider,
                       AsyncWrap* parent)
    : AsyncWrap(env, object, provider, parent),
      flags_(0),
      handle__(handle) {
  handle__->data = this;
  ...
}
```
由于`TimerWrap`继承自`HandleWrap`，对象构造时就把 `handle` 的私有变量 `data` 指向了 this 指针，也就是`HandleWrap`。回调函数通过强转获取了 `TimerWrap` 对象。

令人感兴趣的是 L96,这边是由 C++ 调用 jsland. 查看该处的修改历史，笔者发现：

>  
>   timers: dispatch ontimeout callback by array index
>   
>    Achieve a minor speed-up by looking up the timeout callback on the timer
>    object by using an array index rather than a named property.
    
>    Gives a performance boost of about 1% on the misc/timers benchmarks.

之前的实现是属性查找，而通过极致的优化，属性查找被替换成数组索引，
benchmark性能提升了1%。 而整个系统性能的提升正是来源于这点滴的积累。

### timers.js
有了桥接层，js便有了开启、关闭一个定时器的能力。

为了不影响到nodejs中的event loop，timer模块专门提供了一些内部的api:`timers._unrefActive` 给像socket这样的对象使用。

在最初的设计中，每次执行_unrefActive添加任务时都会维持着unrefList的顺序，保证超时时间最小的处于前面。这样在定时器超时后便可以以最快的速度处理超时任务并设置下一个定时器，但是在添加任务时最坏的情况下需要遍历unrefList链表中的所有节点。

```js
517 exports._unrefActive = function(item) {
518   var msecs = item._idleTimeout;
519   if (!msecs || msecs < 0) return;
520   assert(msecs >= 0);
521 
522   L.remove(item);
523 
524   if (!unrefList) {
525     debug('unrefList initialized');
526     unrefList = {};
527     L.init(unrefList);
528 
529     debug('unrefTimer initialized');
530     unrefTimer = new Timer();
531     unrefTimer.unref();
532     unrefTimer.when = -1;
533     unrefTimer[kOnTimeout] = unrefTimeout;
534   }
535 
536   var now = Timer.now();
537   item._idleStart = now;
538 
539   if (L.isEmpty(unrefList)) {
540     debug('unrefList empty');
541     L.append(unrefList, item);
542 
543     unrefTimer.start(msecs, 0);
544     unrefTimer.when = now + msecs;
545     debug('unrefTimer scheduled');
546     return;
547   }
548 
549   var when = now + msecs;
550 
551   debug('unrefList find where we can insert');
552 
553   var cur, them;
554 
555   for (cur = unrefList._idlePrev; cur != unrefList; cur = cur._idlePrev) {
556     them = cur._idleStart + cur._idleTimeout;
557 
558     if (when < them) {
559       debug('unrefList inserting into middle of list');
560 
561       L.append(cur, item);
562 
563       if (unrefTimer.when > when) {
564         debug('unrefTimer is scheduled to fire too late, reschedule');
565         unrefTimer.start(msecs, 0);
566         unrefTimer.when = when;
567       }
568 
569       return;
570     }
571   }
572 
573   debug('unrefList append to end');
574   L.append(unrefList, item);
575 };

```
L524-L534, 是有且只创建一个`unrefTimer`,来处理超时的内部使用定时器，处理完一个则顺序处理下一个。

L553-L571, 当需要插入一个定时器时，则需要保证`unrefList`有序，需要遍历链表找到插入的位置，最差的情况下是 O(N)。

很显然，在HTTP中建立连接是最频繁的操作，那么向`unrefList`链表中添加节点也就非常频繁了，而且最开始设置的定时器其实最后真正会超时的非常少，因为中间涉及到io的正常操作时便会取消定时器。所以问题就变成最耗性能的操作非常频繁，而几乎不花时间的操作却很少被执行到。

针对这种情况，如何解决呢？

显然这里也遵从80/20原则。思路上我们应该使80%的情况变得更高效。

#### 使用不排序的链表
主要思路就是将对unrefList链表的遍历操作，移到unrefTimeout定时器超时处理中。这样每次查找出已经超时的任务就需要花比较多的时间了O(n)，但是插入操作却变得非常简单O(1)，而插入节点正是最频繁的操作。
```js
572 exports._unrefActive = function(item) {
573   ....省略
574   var now = Timer.now();
575   item._idleStart = now;
576 
577   var when = now + msecs;
578 
579   // If the actual timer is set to fire too late, or not set to fire at all,
580   // we need to make it fire earlier
581   if (unrefTimer.when === -1 || unrefTimer.when > when) {
582     unrefTimer.start(msecs, 0);
583     unrefTimer.when = when;
584     debug('unrefTimer scheduled');
585   }
586 
587   debug('unrefList append to end');
588   L.append(unrefList, item);
589 };
```
可以看到 L588，之前遍历查找在新的实现中 [e5bb668](https://github.com/misterdjules/node/commit/e900f0cf79ce38712ee7f95f3cb0bee8fc56ba89)，简单的变成抽象List的`append`操作。

>  https://github.com/joyent/node/issues/8160

#### 使用二叉堆

二叉堆达到了插入和查找的平衡，和目前 libuv 的实现一致。
有兴趣的可以查看：
* https://github.com/misterdjules/node/commits/fix-issue-8160-with-heap, 基于 v0.12.

#### 社区改进实现

* 有序链表的实现的版本只采用了一个`unrefTimer`来执行任务，在内存上是节省了，但却很难达到性能的平衡。
* 二叉堆实现在正常的连接场景下却输于不排序链表。

社区通过演变，实现采用的是哈希+链表的结合，以空间换时间。其实是一种时间轮算法的演化。
```js
 ╔════ > Object Map
 ║
 ╠══
 ║ refedLists: { '40': { }, '320': { etc } } (keys of millisecond duration)
 ╚══          ┌─────────┘
              │
 ╔══          │
 ║ TimersList { _idleNext: { }, _idlePrev: (self), _timer: (TimerWrap) }
 ║         ┌────────────────┘
 ║    ╔══  │                              ^
 ║    ║    { _idleNext: { },  _idlePrev: { }, _onTimeout: (callback) }
 ║    ║      ┌───────────┘
 ║    ║      │                                  ^
 ║    ║      { _idleNext: { etc },  _idlePrev: { }, _onTimeout: (callback) }
 ╠══  ╠══
 ║    ║
 ║    ╚════ >  Actual JavaScript timeouts
 ║
 ╚════ > Linked List
```
我们先看下数据结构的组织：

* `refedLists`的键是超时时间，值是一个具有相同超时时间的链表。
* `unrefedLists`也是同理。

```js
107 // Internal APIs that need timeouts should use `_unrefActive()` instead of
108 // `active()` so that they do not unnecessarily keep the process open.
109 exports._unrefActive = function(item) {
110   insert(item, true);
111 };
114 // The underlying logic for scheduling or re-scheduling a timer.
115 //
116 // Appends a timer onto the end of an existing timers list, or creates a new
117 // TimerWrap backed list if one does not already exist for the specified timeout
118 // duration.
119 function insert(item, unrefed) {
120   const msecs = item._idleTimeout;
121   if (msecs < 0 || msecs === undefined) return;
122 
123   item._idleStart = TimerWrap.now();
124 
125   const lists = unrefed === true ? unrefedLists : refedLists;
126 
127   // Use an existing list if there is one, otherwise we need to make a new one.
128   var list = lists[msecs];
129   if (!list) {
130     debug('no %d list was found in insert, creating a new one', msecs);
131     // Make a new linked list of timers, and create a TimerWrap to schedule
132     // processing for the list.
133     list = new TimersList(msecs, unrefed);
134     L.init(list);
135     list._timer._list = list;
136 
137     if (unrefed === true) list._timer.unref();
138     list._timer.start(msecs, 0);
139 
140     lists[msecs] = list;
141     list._timer[kOnTimeout] = listOnTimeout;
142   }
143 
144   L.append(list, item);
145   assert(!L.isEmpty(list)); // list is not empty
146 }
```

我们比较下上述实现：
* L128，根据键值（超时时间）拿到 list ，如有不为undefined,则简单的`append`到最后面就好了,复杂度O(1)。
* L130-L141, 如果为 undefined, 则创建一个`TimersList`,包含一个C的定时器，来处理链表中的任务。
* `listOnTimeout`也变得很简单，取出链表的任务，复杂度取决于链表的长度O(m),  m < N。


模块使用一个链表来保存所有超时时间相同的对象，每个对象中都会存储开始时间_idleStart以及超时时间_idleTimeout。链表中第一个加入的对象一定会比后面加入的对象先超时，当第一个对象超时完成处理后，重新计算下一个对象是否已经到时或者还有多久到时，之前创建的Timer对象便会再次启动并设置新的超时时间，直到当链表上所有的对象都已经完成超时处理，此时便会关闭这个Timer对象。

通过这种巧妙的设计，使得一个Timer对象得到了最大的复用，从而极大的提升了timer模块的性能。

### Timer在node中的应用
* 动态更新 HTTP Date字段的缓存

```js
 31 var dateCache;
 32 function utcDate() {
 33   if (!dateCache) {
 34     var d = new Date();
 35     dateCache = d.toUTCString();
 36     timers.enroll(utcDate, 1000 - d.getMilliseconds());
 37     timers._unrefActive(utcDate);
 38   }
 39   return dateCache;
 40 }
 41 utcDate._onTimeout = function() {
 42   dateCache = undefined;
 43 };
 
228   // Date header
229   if (this.sendDate === true && state.sentDateHeader === false) {
230     state.messageHeader += 'Date: ' + utcDate() + CRLF;
231   }
```
L230，每次构造 Date字段值都会去获取系统时间，但精度要求不高，只需要秒级就够了，所以在1S 的连接请求可以复用 dateCache 的值，超时后重置为`undefined`.

L34-L35,下次获取会重启生成。

L36-L37,重新设置超时时间以便更新。

* HTTP 连接超时控制

```js
303   if (self.timeout)
304     socket.setTimeout(self.timeout);
305   socket.on('timeout', function() {
306     var req = socket.parser && socket.parser.incoming;
307     var reqTimeout = req && !req.complete && req.emit('timeout', socket);
308     var res = socket._httpMessage;
309     var resTimeout = res && res.emit('timeout', socket);
310     var serverTimeout = self.emit('timeout', socket);
311 
312     if (!reqTimeout && !resTimeout && !serverTimeout)
313       socket.destroy();
314   });
```
默认的 timeout 为`this.timeout = 2 * 60 * 1000;`也就是120s。 L313，超时则销毁 socket。

### 小结
Node.js 的 timer 模块闪烁着很多程序设计的精髓。
* 数据结构抽象
  - linkedlist.js 抽象出链表的基础操作。
* 以空间换时间
  - 相同超时时间的定时器分组，而不是使用一个`unrefTimer`，复杂度降到 O(1)。
* 对象复用
  - 相同超时时间的定时器共享一个底层的 C的 timer。
* 80/20法则
  - 优化主要路径的性能。


### 参考文档
[1].https://github.com/nodejs/node/wiki/Optimizing-_unrefActive

[2].http://alinode.aliyun.com/blog/9
