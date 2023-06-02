
## libuv Selection

### Linux Native AIO

Linux Native AIO has two APIs, one is provided by libaio, and the other is a system call encapsulated API, which is used more because it does not require additional libraries and is simple.

- io_setup: is used to set up the context of an asynchronous request. The first parameter is the number of request events, and the second parameter uniquely identifies an asynchronous request.
- io_commit: is used to submit an asynchronous IO request. Before submitting, you need to set the structure `iocb`.
- io_getevents: is used to obtain completed IO events. The parameter `min_nr` is the minimum number of events, and `nr` is the maximum number of events. If there are not enough events, the function will block.
- io_destroy: After all events are processed, call this function to destroy the asynchronous IO request.

#### Limitations
AIO can only be used for regular file IO and cannot be used for socket, pipe, and other IO. However, the demand for the libuv fs module has been sufficient.

After io_getevents is called, it will block until enough events occur. Therefore, to achieve true asynchronous IO, you need to use eventfd and epoll to achieve the goal.

#### libuv Native AIO Implementation
A native aio based on libuv has been implemented, https://github.com/yjhjstz/libuv/commit/2748728635c4f74d6f27524fd36e680a88e4f04a

In theory, implementing AIO in libuv:
* First: it reduces one write system call compared to the original libuv implementation, and there is no need to implement a thread pool and work queue in user space.
* Second: native aio implementation can achieve batch callbacks.

Let's take a look at the performance comparison data. The test script is a simple file read:
* Threadpool model
```shell
jiangling@young:~/workspace/libuv$ wrk -t4 -c100 -d30s http://127.0.0.1:30003/
Running 30s test @ http://127.0.0.1:30003/
4 threads and 100 connections
Thread Stats Avg Stdev Max +/- Stdev
Latency 16.77ms 1.14ms 31.68ms 86.68%
Req/Sec 1.51k 162.66 2.08k 81.34%
178925 requests in 30.00s, 104.26MB read
Requests/sec: 5963.45
Transfer/sec: 3.47MB
```

* Native AIO model
```shell
jiangling@young:~/workspace/libuv$ wrk -t4 -c100 -d30s http://127.0.0.1:30003/
Running 30s test @ http://127.0.0.1:30003/
4 threads and 100 connections
Thread Stats Avg Stdev Max +/- Stdev
Latency 16.22ms 0.95ms 26.39ms 88.12%
Req/Sec 1.57k 191.14 2.08k 68.50%
185084 requests in 30.00s, 107.85MB read
Requests/sec: 6169.28
Transfer/sec: 3.59MB
```

**Max Latency decreased by 16%, and TPS increased by 3%.**

### Threadpool Model
Let's first look at the schematic diagram of a node.js read call:
![](node-aio.png)

The code goes through the following steps:
- 1 node, libuv initialization;
- 2 The Read method in node_file.cc calls `uv_fs_read` of libuv (fs.c) to encapsulate the request;
- 3 libuv encapsulates the request into uv_work and submits it to the end of the task queue, triggering the signal;
- 4 At this time, the read call of the main thread returns.
- 5 The thread pool takes out a request from the uv_work queue and starts executing read IO;
- 6 Send a signal to the main thread to indicate that the task is completed, and wait for other operations after executing the read call.
- 7 The main thread epoll takes out completed requests from the response queue;
- 8 The main thread responds to epoll events;
- 9 The main thread executes the callback function of the request.

The asynchronous IO of node.js is clear, and we can clearly see that such a Threadpool model is applicable to all platforms.

> Linux AIO
> AIO first appeared in the 2.5 version of the kernel and is now a standard feature of the product kernel of version 2.6.

And because Native AIO was introduced after linux 2.6 and is not stable. The community has also had intense discussions:

- https://github.com/libuv/libuv/issues/28
- https://github.com/libuv/libuv/issues/461

After weighing the pros and cons, I also strongly support the model adopted by the community, giving users more choices and reliability.

### Summary
The user-space thread pool implementation gives users greater flexibility and selectivity. For example:

* 1. The number of thread pools, the default is 4, and users can specify it by setting the environment variable `UV_THREADPOOL_SIZE`.
* 2. Reuse thread pools with time-consuming GETADDRINFO.

It should be pointed out that there is still room for improvement in the thread pool model:
* Optimization of the global lock `static uv_mutex_t mutex;`
* Support task priority.

### Reference


