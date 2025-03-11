## 文件io

上一章节在讲述了线程池的模型，读者对一次异步 IO 发起到结束有了一个大致的认识。

fs 模块还提供了同步接口，如 `readFileSync`， 这在异步模型的 node.js 的
核心模块中是极为少见的。


### 请求对象

* 异步读文件接口定义： 
`fs.readFile = function(path, options, callback_) `
* 同步读文件接口定义：
`fs.readFileSync = function(path, options) `

两者明显的差异在于第三个参数 `callback_`, 异步会提交请求然后等待回调，同步则阻塞直到返回。

让我们来看看第三个参数对实现的影响。

```js
  // fs.js
  // ...
  var context = new ReadFileContext(callback, encoding);
  var req = new FSReqWrap();
  req.context = context;
  req.oncomplete = readFileAfterOpen;
```

异步的实现中会创建一个请求对象，并且绑定回调和上下文环境。 该请求对象由 C++ 绑定导出。


#### FSReqWrap (node.js)
FSReqWrap 是由 `src/node_file.cc` 实现并导出，提供给 javascript 使用。

```c++
class FSReqWrap: public ReqWrap<uv_fs_t> {
 public:
  enum Ownership { COPY, MOVE };

  inline static FSReqWrap* New(Environment* env,
                               Local<Object> req,
                               const char* syscall,
                               const char* data = nullptr,
                               Ownership ownership = COPY);

  inline void Dispose();
  
  //...
};

```
FSReqWrap 继承 ReqWrap<uv_fs_t>, ReqWrap 是个模板类，`T req_;` 存储了不同类型的请求，在这里
模板编译后，`uv_fs_t req_;`, req_ 存储了 uv_fs_t 请求对象。 这源自于一次性能优化的提交。
> fs: improve `readFile` performance
    
> This commit improves `readFile` performance by
> reducing number of closure allocations and using
> `FSReqWrap` directly.

具体了解, https://github.com/iojs/io.js/pull/718 。


在js 层发起请求后，会来到C++绑定层，
```c++
#define ASYNC_DEST_CALL(func, req, dest, ...)                                 \
  Environment* env = Environment::GetCurrent(args);                           \
  CHECK(req->IsObject());                                                     \
  FSReqWrap* req_wrap = FSReqWrap::New(env, req.As<Object>(), #func, dest);   \
  int err = uv_fs_ ## func(env->event_loop(),                                 \
                           &req_wrap->req_,                                   \
                           __VA_ARGS__,                                       \
                           After);                                            \
  req_wrap->Dispatched();                                                     \
  if (err < 0) {                                                              \
    uv_fs_t* uv_req = &req_wrap->req_;                                        \
    uv_req->result = err;                                                     \
    uv_req->path = nullptr;                                                   \
    After(uv_req);                                                            \
    req_wrap = nullptr;                                                       \
  } else {                                                                    \
    args.GetReturnValue().Set(req_wrap->persistent());                        \
  }

#define ASYNC_CALL(func, req, ...)                                            \
  ASYNC_DEST_CALL(func, req, nullptr, __VA_ARGS__)                            \
```

这里才会生成 libuv 所需的请求对象， 对于读请求调用 `uv_fs_read`, 提交请求，指定回调函数为 `After`。

#### uv_fs_t (libuv)
看一下libuv的异步读文件代码，deps/uv/src/unix/fs.c：

```c++
/* uv_fs_t is a subclass of uv_req_t. */
struct uv_fs_s {
  UV_REQ_FIELDS
  uv_fs_type fs_type;
  uv_loop_t* loop;
  uv_fs_cb cb;
  ssize_t result;
  void* ptr;
  const char* path;
  uv_stat_t statbuf;  /* Stores the result of uv_fs_stat() and uv_fs_fstat(). */
  UV_FS_PRIVATE_FIELDS
};
```

```c++
#define INIT(subtype)                                                         \
  do {                                                                        \
    req->type = UV_FS;                                                        \
    if (cb != NULL)                                                           \
      uv__req_init(loop, req, UV_FS);                                         \
    req->fs_type = UV_FS_ ## subtype;                                         \
    req->result = 0;                                                          \
    req->ptr = NULL;                                                          \
    req->loop = loop;                                                         \
    req->path = NULL;                                                         \
    req->new_path = NULL;                                                     \
    req->cb = cb;                                                             \
  }                                                                           \
  while (0)
```

可以看到一次异步文件读操作在libuv层被封装到一个uv_fs_t的结构体，req->cb是来自上层的回调函数（node C++层：src/node_file.cc 的After函数）。

异步io请求最后调用uv__work_submit，把异步io请求提交给线程池。这里有两个函数：

* `uv__fs_work`：这个是文件io的处理函数，可以看到当cb为NULL的时候，即非异步模式，`uv__fs_work`在当前线程（事件循环所在线程）直接被调用。如果cb != NULL，即文件io为异步模式，此时把`uv__fs_work`和`uv__fs_done`提交给线程池。

* `uv__fs_done`：这个是异步文件io结束后的回调函数。在uv__fs_done里面会回调上层C++模块的cb函数（即req->cb）。

需要特别注意的是：** 此时io操作的主体 `uv__fs_work`函数是在线程池里执行的。
但是`uv__fs_done`
必须在事件循环的线程里被回调，因为这个函数最终会回调到用户js代码的回调函数，而js代码里的所有代码必须在同个线程里面。**

#### 线程池的请求对象 —— struct uv__work
先看下 `uv__work`的定义：
```c++
struct uv__work {
  void (*work)(struct uv__work *w);
  void (*done)(struct uv__work *w, int status);
  struct uv_loop_s* loop;
  void* wq[2];
};
```
再看看`uv__work_submit`做了什么：
```c++
static void post(QUEUE* q) {
  uv_mutex_lock(&mutex);
  QUEUE_INSERT_TAIL(&wq, q);
  if (idle_threads > 0)
    uv_cond_signal(&cond);
  uv_mutex_unlock(&mutex);
}

void uv__work_submit(uv_loop_t* loop,
                     struct uv__work* w,
                     void (*work)(struct uv__work* w),
                     void (*done)(struct uv__work* w, int status)) {
  uv_once(&once, init_once);
  w->loop = loop;
  w->work = work;
  w->done = done;
  post(&w->wq);
}
```

`uv__work_submit` 把传进来的`uv__fs_work`、`uv__fs_done`封装到`uv__work`结构体里面，这个结构体表示一个线程操作的请求。通过post把请求提交给线程池。

看到post函数里面的QUEUE_INSERT_TAIL，把该`uv__work`对象加进`wq`链表里面。wq是一个全局静态变量。也就是说，进程空间里的所有线程共用同一个wq链表。


看到post函数通过uv_cond_signal向相应的条件变量——cond发送信号，处在uv_cond_wait挂起等待的工作线程当中的某个线程被激活。

worker线程往下执行，从wq取出w，执行w->work()。

工作线程完成任务后，调用uv_async_send通知主线程统一的io观察者，执行 callback。


### 回调
在一次异步文件 I/O 操作中，回调函数是整个流程的最后一环，用以将底层 I/O 操作的结果传递给上层应用。具体流程如下：

1. **工作线程完成任务**  
   工作线程从全局工作队列中取出一个 `uv__work` 请求，执行其中封装的 `uv__fs_work` 函数，该函数完成实际的文件 I/O 操作（例如文件读取）。

2. **通知主线程**  
   完成 I/O 操作后，工作线程调用 `uv_async_send`，向主线程发送信号。这一机制确保了回调函数不会在工作线程中执行，而是延迟到主线程中被调度，从而保证了 JavaScript 层代码的单线程执行环境。

3. **执行回调函数**  
   主线程在检测到异步事件后，会调用 `uv__fs_done` 回调函数。该函数负责将 `uv_fs_t` 请求对象中的操作结果（如读取的数据、错误码等）传递给上层 C++ 模块中的 `After` 函数。随后，`After` 函数会调用用户在 JavaScript 层传入的回调函数，将操作结果反馈给用户。

4. **事件循环与回调协调**  
   由于 Node.js 的 JavaScript 代码运行在单线程的事件循环中，因此所有用户回调都必须在主线程中执行。这一设计确保了回调函数能够安全地访问共享状态，而无需担心线程安全问题，同时也使得异步 I/O 操作与事件循环的协同工作成为可能。

这种回调机制将底层的异步 I/O 操作与上层的 JavaScript 回调解耦合，通过工作线程与主线程之间的信号通知，实现了高效且安全的异步操作流程。

### 总结
通过对各层请求对象的梳理，也详细梳理出了一次 read 请求的脉络, 使读者有了一个理性的认识。

### 参考

- https://github.com/nodejs/node/blob/master/src/node_file.cc
- https://github.com/libuv/libuv 
- https://github.com/iojs/io.js/pull/718
- https://github.com/libuv/libuv/blob/v1.x/src/unix/fs.c

