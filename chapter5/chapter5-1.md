
## Event Loop

> "Event Loop is a programming construct that waits for and dispatches events or messages in a program."

![](5fee18eegw1ewjpoxmdf5j20k80b1win.jpg)


### Event Loop
The responsibility of the event loop is to constantly wait for events to occur, and then execute all the handlers for this event in the order of their subscription to this event. When all the handlers for this event have been executed, the event loop will start to continue waiting for the next event trigger, and so on.

When multiple requests are processed concurrently, the above concept is also correct, and can be understood as follows: in a single thread, event handlers are executed one by one in order.

That is, if an event is bound to two handlers, the second handler will start executing only after the first handler has finished executing. The event loop will not check whether there is a new event trigger until all the handlers for this event have been executed. In a single thread, everything is executed in order one by one!

### Event Loop in Node.js

Node uses V8 as the JavaScript execution engine and uses libuv to implement event-driven asynchronous I/O. Its event loop uses the default event loop of libuv.
In src/node.cc,
```c++
Environment* env = CreateEnvironment(
        node_isolate,
        uv_default_loop(),
        context,
        argc,
        argv,
        exec_argc,
        exec_argv);
```
This code creates a node execution environment. You can see the third line of uv_default_loop() in it. This is a function in the libuv library, which initializes the uv library itself and the default_loop_struct in it, and returns a pointer to it, default_loop_ptr. After that, Node will load the execution environment and complete some setup operations, and then start the event loop:
```c++
bool more;
do {
  more = uv_run(env->event_loop(), UV_RUN_ONCE);
  if (more == false) {
    EmitBeforeExit(env);
    // Emit `beforeExit` if the loop became alive either after emitting
    // event, or after running some callbacks.
    more = uv_loop_alive(env->event_loop());
    if (uv_run(env->event_loop(), UV_RUN_NOWAIT) != 0)
      more = true;
  }
} while (more == true);
code = EmitExit(env);
RunAtExit(env);
```

more is used to indicate whether to perform the next loop. env->event_loop() will return the default_loop_ptr saved in env before, and the uv_run function will start the libuv event loop in the specified UV_RUN_ONCE mode. In this mode, uv_run will process at least one event: this means that if there is no I/O event that needs to be processed in the current event queue, uv_run will block until there is an I/O event that needs to be processed or the next timer time is up. If there are no I/O events or timer events at the moment, uv_run returns false.

Next, Node will decide the next step based on the situation of more:

- If more is true, continue to run the next loop.

- If more is false, it means that there are no events waiting to be processed. EmitBeforeExit(env); triggers the 'beforeExit' event of the process, checks and processes the corresponding processing functions, and then directly exits the loop.

Finally, trigger the 'exit' event, execute the corresponding callback function, and Node ends, and some resource release operations will be performed later.

In libuv, the event loop updates its time at the beginning of each loop to achieve timing function, and I/O events are divided into two categories:

- Network I/O uses the non-blocking I/O solution provided by the system, such as epoll on Linux and IOCP on Windows.

- File operations and DNS operations do not have (good) system solutions, so libuv builds its own thread pool to perform blocking I/O.

In addition, we can also throw custom functions into the thread pool to run. After the operation is completed, the main thread will execute the corresponding callback function. However, Node has not added this function to JavaScript, which means that it is impossible to start a new thread for parallel execution in JavaScript with native Node.

### process.nextTick
![](settimeout.jpeg)
With this question in mind, let's take a look at how the JS layer's nextTick is driven.

In the entry point `src/node.js`, the `processNextTick` method constructs the `process.nextTick` API.

`process._tickCallback` is hung on the `process` object as the callback function of nextTick, and is called back by the C++ layer.

```js
startup.processNextTick = function() {
    var nextTickQueue = [];
    var pendingUnhandledRejections = [];
    var microtasksScheduled = false;

    // Used to run V8's micro task queue.
    var _runMicrotasks = {};

    // *Must* match Environment::TickInfo::Fields in src/env.h.
    var kIndex = 0;
    var kLength = 1;

    process.nextTick = nextTick;
    // Needs to be accessible from beyond this scope.
    process._tickCallback = _tickCallback;
    process._tickDomainCallback = _tickDomainCallback;

    // This tickInfo thing is used so that the C++ code in src/node.cc
    // can have easy access to our nextTick state, and avoid unnecessary
    // calls into JS land.
    const tickInfo = process._setupNextTick(_tickCallback, _runMicrotasks);
    // omitted...
}
```
Register `_tickCallback` to `env`'s `tick_callback_function` through `process._setupNextTick`.


In the `src/async_wrap.cc` file, we find the following call:
```js
Local<Value> AsyncWrap::MakeCallback(const Local<Function> cb,
                                      int argc,
                                      Local<Value>* argv) {
  // ...
  Environment::TickInfo* tick_info = env()->tick_info();

  if (tick_info->in_tick()) {
    return ret;
  }

  if (tick_info->length() == 0) {
    env()->isolate()->RunMicrotasks();
  }

  if (tick_info->length() == 0) {
    tick_info->set_index(0);
    return ret;
  }

  tick_info->set_in_tick(true);

  env()->tick_callback_function()->Call(process, 0, nullptr);

  tick_info->set_in_tick(false);
  // ...
```



Otherwise, `_tickCallback` is called through `tick_callback_function`.

At this point, I also have a question: what if there is no asynchronous IO? How is it driven?

We come to `lib/module.js`, as follows:

```js
// bootstrap main module.
Module.runMain = function() {
  // Load the main module--the command line argument.
  Module._load(process.argv[1], null, true);
  // Handle any nextTicks added in the first tick of the program
  process._tickCallback();
};
```

After `Module._load` loads the main script, it calls `_tickCallback` to handle the first tick.

So the above question has an answer. `nextTick` is mainly driven by `uv__io_poll`. Why do we say mainly? Because it may also be driven by the Timer module. The specific details are left for readers to study.

### Summary


### Reference
* http://acemood.github.io/2016/02/01/event-loop-in-javascript/








