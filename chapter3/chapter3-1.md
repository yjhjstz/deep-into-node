

## Timer Source Code Analysis


### Related Source Code
* [lib/timers.js](https://github.com/nodejs/node/blob/master/lib/timers.js)
* [src/timer_wrap.cc](https://github.com/nodejs/node/blob/master/src/timer_wrap.cc)
* [deps/uv/src/unix/timer.c](https://github.com/nodejs/node/blob/master/deps/uv/src/unix/timer.c)
* [deps/uv/src/heap-inl.h](https://github.com/nodejs/node/blob/master/deps/uv/src/heap-inl.h)

The implementation of timers is divided into two parts: the JavaScript layer and the `libuv` layer. `timer_wrap.cc` acts as a bridge to facilitate the interaction between JavaScript and C++.



### Use Cases
The main use cases or applicable scenarios for timers are:
* Scheduled tasks, such as timed status checks in business logic;
* Timeout control, such as network timeout control for retransmission.

In the implementation of Node.js, for example in the HTTP module:
```js
function responseOnEnd() {
    // omitted
    debug('AGENT socket keep-alive');
    if (req.timeoutCb) {
      socket.setTimeout(0, req.timeoutCb);
      req.timeoutCb = null;
    }
 }
````
You may wonder: why is it used in the HTTP module?

We know that the HTTP protocol uses a "request-response" mode. When using the normal mode, that is, the non-KeepAlive mode, each request/response between the client and server requires a new connection, which is immediately disconnected after completion (HTTP protocol is a connectionless protocol); when using the Keep-Alive mode (also known as persistent connection, connection reuse), the Keep-Alive function keeps the client-to-server connection valid, avoiding the need to establish or re-establish a connection when subsequent requests are made to the server.

 ```js
  if (req.httpVersionMajor < 1 || req.httpVersionMinor < 1) {
    this.useChunkedEncodingByDefault = chunkExpression.test(req.headers.te);
    this.shouldKeepAlive = false;
  }
```
   In HTTP/1.0, it is turned off by default and needs to be enabled by adding "Connection: Keep-Alive" to the HTTP header; in HTTP/1.1, Keep-Alive is enabled by default, and "Connection: close" is added to turn it off.
 
  Currently, most browsers use the HTTP/1.1 protocol, which means that Keep-Alive connection requests are sent by default. Node.js judges and processes the two protocols according to the above code.
    
   Of course, this connection cannot be kept alive forever, so there is usually a timeout period. If the client has not sent a new HTTP request after this period, the server needs to automatically disconnect in order to continue to serve other clients.
   The HTTP module of Node.js creates a socket object for each new connection and calls socket.setTimeout to set a timer for automatic disconnection after timeout.



### Data Structure Selection
A Timer is essentially a data structure where tasks with closer deadlines have higher priority. It provides the following three basic operations:
* schedule: add a task
* cancel: delete a task
* expire: execute expired tasks

Implementation |	schedule | cancel	| expire 
-------| ----------| --------  | ------
Linked List |	O(1) |	O(n)  | O(n)
Sorted Linked List | O(n)|	O(1) | O(1)
Min Heap|	O(lgn)|	O(1)  |	O(1)
Time Wheel|	O(1) |	O(1)  |	O(1)

The implementation of timers has undergone changes, each of which is a spark of collision of ideas. Let's dive into the source code and savor it carefully.




### libuv Implementation
#### Data Structure - Min Heap
A min heap is a binary heap, which is a complete binary tree or an approximately complete binary tree. It is divided into two types: a maximum heap and a minimum heap.
Maximum heap: the key value of the parent node is always greater than or equal to the key value of any child node; minimum heap: the key value of the parent node is always less than or equal to the key value of any child node. The schematic diagram is as follows:
![binary-tree](http://images.cnitblog.com/i/497634/201403/182339209436216.jpg)

The node is defined in `deps/uv/src/heap-inl.h` as follows:

```c
struct heap_node {
  struct heap_node* left;
  struct heap_node* right;
  struct heap_node* parent;
};
```
The root node is defined as follows:
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
Here we can see clearly that the min heap organizes data with pointers, not arrays. `min` always points to the smallest node if it exists. As a sorted set, it also needs a user-specified comparison function to determine which node is smaller, or when the expiration time is the same, to determine their order. After all, there are no rules without rules.
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

Here we can see that first, the `timeout` of the two is compared. If the two are the same, then the `id` of the two that were `schedule` is compared. This `id` is generated by `loop->timer_counter` in `uv_timer_start` and assigned to `start_id`.



#### Implementation
```c
int uv_timer_start(uv_timer_t* handle, uv_timer_cb cb, uint64_t timeout, uint64_t repeat) {
  uint64_t clamped_timeout;
  
  if (cb == NULL) {
    return -EINVAL;
  }
  
  if (uv__is_active(handle)) {
    uv_timer_stop(handle);
  }
  
  clamped_timeout = handle->loop->time + timeout;
  if (clamped_timeout < timeout) {
    clamped_timeout = (uint64_t) -1;
  }
  
  handle->timer_cb = cb;
  handle->timeout = clamped_timeout;
  handle->repeat = repeat;
  handle->start_id = handle->loop->timer_counter++;
  
  heap_insert((struct heap*) &handle->loop->timer_heap, (struct heap_node*) &handle->heap_node, timer_less_than);
  uv__handle_start(handle);
  
  return 0;
}
```
* L68-L69, Check the parameters and return -EINVAL if there is an error.
* L71-L72, If there is an active timer, stop it immediately.
* L74-L82, Assign parameters, and the `start_id` mentioned above is obtained by incrementing `timer_counter`.
* L84-L86, Insert the timer node into the min heap, and the algorithm complexity here is O(lgn).
* L87, Mark the handle as inactive and add it to the statistics.


```c
int uv_timer_stop(uv_timer_t* handle) {
  if (!uv__is_active(handle))
    return 0;

  heap_remove((struct heap*) &handle->loop->timer_heap,
              (struct heap_node*) &handle->heap_node,
              timer_less_than);
  uv__handle_stop(handle);

  return 0;
}
```
L94, Check the handle. If it is inactive, it means it has not been started, so return success.
L97-L99, Remove the timer node from the min heap.
L100, Reset the handle and decrement the count.



After understanding how to start and stop a timer, let's see how to schedule a timer.

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
In the event loop of node.js, after updating the time, `uv__run_timers` is called immediately, indicating that timers, as an external system dependency module, have the highest priority.

```c
void uv__run_timers(uv_loop_t* loop) {
  struct heap_node* heap_node;
  uv_timer_t* handle;

  for (;;) {
    heap_node = heap_min((struct heap*) &loop->timer_heap);
    if (heap_node == NULL)
      break;

    handle = container_of(heap_node, uv_timer_t, heap_node);
    if (handle->timeout > loop->time)
      break;

    uv_timer_stop(handle);
    uv_timer_again(handle);
    handle->timer_cb(handle);
  }
}
```

L155-L157, Get the minimum timer node. If it is empty, exit the loop.
L159-L161, Get the object's address through the offset of heap_node. If the minimum timeout is greater than the current time, it means that the expiration time has not yet arrived, so exit the loop.
L163-L165, Delete the timer. If it is a timer that needs to be executed repeatedly, it is added again by calling `uv_timer_again`. After executing the timer's callback task, loop again.



#### Improved hierarchical time wheel implementation
* https://github.com/libuv/libuv/pull/823



### Bridge Layer
This section requires knowledge of node.js addon. It is assumed that you already have this knowledge.

```c
 43     env->SetProtoMethod(constructor, "start", Start);
 44     env->SetProtoMethod(constructor, "stop", Stop);
 45 
 46     target->Set(FIXED_ONE_BYTE_STRING(env->isolate(), "Timer"),
 47                 constructor->GetFunction());
```
The Timer addon exports the `start` and `stop` methods for use by the JS layer.

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

`Start` requires two parameters: 1. timeout; 2. the period of repeated execution. L78 calls `uv_timer_start`, where `OnTimeout` is the callback function for the timer. Let's take a look at the implementation of this function:

```c++
 91   static void OnTimeout(uv_timer_t* handle) {
 92     TimerWrap* wrap = static_cast<TimerWrap*>(handle->data);
 93     Environment* env = wrap->env();
 94     HandleScope handle_scope(env->isolate());
 95     Context::Scope context_scope(env->context());
 96     wrap->MakeCallback(kOnTimeout, 0, nullptr);
 97   }
```

You may be wondering how `handle->data` retrieves the object pointer. 

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

Since `TimerWrap` inherits from `HandleWrap`, the `data` private variable of the `handle` is pointed to the `HandleWrap` object, which is `this` pointer. The callback function retrieves the `TimerWrap` object by casting.

What's interesting is L96, where C++ calls JS. By checking the modification history of this location, I found that:


>  
>   timers: dispatch ontimeout callback by array index
>   
>    Achieve a minor speed-up by looking up the timeout callback on the timer
>    object by using an array index rather than a named property.
    
>    Gives a performance boost of about 1% on the misc/timers benchmarks.


The previous implementation used property lookup, and through extreme optimization, property lookup was replaced with array indexing, resulting in a 1% performance improvement in the benchmark. The overall performance improvement comes from these incremental improvements.

### timers.js
With the bridge layer, JS has the ability to start and stop a timer.

To avoid affecting the event loop in node.js, the timer module provides some internal APIs, such as `timers._unrefActive`, for objects like sockets.

In the initial design, each time `_unrefActive` adds a task, it maintains the order of `unrefList` to ensure that the object with the smallest timeout is at the front. This way, when the timer times out, it can process the timeout task at the fastest speed and set the next timer. However, in the worst case, when adding a task, it needs to traverse all nodes in the `unrefList` linked list.


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
L524-L534, only create one `unrefTimer` to handle timeouts for internal use, processing one and then the next.

L553-L571, when inserting a timer, it is necessary to ensure that `unrefList` is ordered, requiring traversal of the linked list to find the insertion point, which is O(N) in the worst case.

Obviously, establishing a connection in HTTP is the most frequent operation, so adding nodes to the `unrefList` linked list is also very frequent, and the initially set timer is actually very rarely timed out, because the normal operation of io will cancel the timer in the middle. So the problem becomes that the most performance-consuming operation is very frequent, while the operation that takes almost no time is rarely executed.

How to solve this problem?

Obviously, this also follows the 80/20 principle. In terms of ideas, we should make 80% of the cases more efficient.


#### Use an unsorted linked list
The main idea is to move the traversal operation of the `unrefList` linked list to the `unrefTimeout` timer timeout processing. This way, finding the timed-out tasks requires more time each time, which is O(n), but the insertion operation becomes very simple, which is O(1), and inserting nodes is the most frequent operation.
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
You can see that L588, previously traversed lookup in the new implementation [e5bb668](https://github.com/misterdjules/node/commit/e900f0cf79ce38712ee7f95f3cb0bee8fc56ba89), simply becomes an abstract List 'append' operation.


>  https://github.com/joyent/node/issues/8160

#### Use a binary heap

A binary heap achieves a balance between insertion and lookup, which is consistent with the current implementation of libuv.
For those interested, please refer to:
* https://github.com/misterdjules/node/commits/fix-issue-8160-with-heap, based on v0.12.

#### Community-improved implementation

* The implementation using an ordered linked list only uses one `unrefTimer` to execute tasks, which saves memory but is difficult to achieve performance balance.
* The binary heap implementation is inferior to an unsorted linked list in normal connection scenarios.

Through evolution, the community-improved implementation uses a combination of hash tables and linked lists to trade space for time. In fact, it is an evolution of the time wheel algorithm.

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
Let's first take a look at the organization of the data structure:

* The keys of `refedLists` are the timeout durations, and the values are linked lists with the same timeout duration.
* `unrefedLists` is the same.


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

Let's compare the above implementation:
* L128, get the list according to the key (timeout time), if it is not undefined, simply `append` it to the end, complexity O(1).
* L130-L141, if it is undefined, create a `TimersList`, which contains a C timer to handle tasks in the linked list.
* `listOnTimeout` also becomes very simple, taking out the tasks in the linked list, complexity depends on the length of the linked list O(m), m < N.

The module uses a linked list to store all objects with the same timeout time. Each object stores the start time _idleStart and the timeout time _idleTimeout. The first object added to the linked list will always time out before the later added objects. When the first object completes its timeout processing, the next object's timeout time is recalculated to see if it has already timed out or how long it will take to time out. The previously created Timer object will be restarted and set with a new timeout time until all objects on the linked list have completed their timeout processing, at which point the Timer object will be closed.

Through this clever design, a Timer object is maximally reused, greatly improving the performance of the timer module.

### Application of Timer in Node.js
* Dynamically update the HTTP Date field cache

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
L230, the `utcDate()` function is used to dynamically update the HTTP Date field cache. The function constructs a new `Date` object and stores its UTC string representation in the `dateCache` variable. The function then enrolls itself in the timer module to be called again in 1 second, and marks itself as unrefed to avoid keeping the process open unnecessarily.

L34-L35, the `dateCache` value is reused for subsequent calls to `utcDate()` within 1 second.

L36-L37, the timer for `utcDate()` is reset to update the cache after 1 second has elapsed.

* HTTP connection timeout control

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
The default timeout is `this.timeout = 2 * 60 * 1000;`, which is 120s. L313, the socket is destroyed if it times out.

### Summary
The timer module in Node.js embodies many programming design principles:
* Data structure abstraction
  - linkedlist.js abstracts the basic operations of linked lists.
* Space-time tradeoff
  - Timers with the same timeout time are grouped instead of using a single `unrefTimer`, reducing complexity to O(1).
* Object reuse
  - Timers with the same timeout time share a C timer at the bottom.
* 80/20 rule
  - Optimize the performance of the main path.




### Reference Documents
[1].https://github.com/nodejs/node/wiki/Optimizing-_unrefActive

[2].http://alinode.aliyun.com/blog/9
