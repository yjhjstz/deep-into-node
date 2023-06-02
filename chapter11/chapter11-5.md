
## File IO

The previous chapter discussed the thread pool model, which gave readers a rough understanding of the process of asynchronous IO from initiation to completion.

The fs module also provides synchronous interfaces, such as `readFileSync`, which is rare in the core modules of node.js's asynchronous model.

### Request Object

* Asynchronous file reading interface definition: 
`fs.readFile = function(path, options, callback_) `
* Synchronous file reading interface definition:
`fs.readFileSync = function(path, options) `

The obvious difference between the two is the third parameter `callback_`. Asynchronous requests submit the request and wait for the callback, while synchronous requests block until they return.

Let's take a look at the impact of the third parameter on the implementation.

```js
  // fs.js
  // ...
  var context = new ReadFileContext(callback, encoding);
  var req = new FSReqWrap();
  req.context = context;
  req.oncomplete = readFileAfterOpen;
```

In the implementation of asynchronous requests, a request object is created and the callback and context environment are bound to it. This request object is exported by C++ bindings.

#### FSReqWrap (node.js)
FSReqWrap is implemented and exported by `src/node_file.cc` for use by JavaScript.

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
FSReqWrap inherits from ReqWrap<uv_fs_t>, ReqWrap is a template class, `T req_;` stores requests of different types, and after template compilation, `uv_fs_t req_;` stores the uv_fs_t request object. This comes from a performance optimization submission.
> fs: improve `readFile` performance
    
> This commit improves `readFile` performance by
> reducing number of closure allocations and using
> `FSReqWrap` directly.

For more information, see https://github.com/iojs/io.js/pull/718.

After the request is initiated in JavaScript, it will come to the C++ binding layer,
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

Here, the libuv request object required is generated, and for read requests, `uv_fs_read` is called to submit the request and specify the callback function as `After`.

#### uv_fs_t (libuv)
Let's take a look at the asynchronous file reading code in libuv, deps/uv/src/unix/fs.c:

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

It can be seen that an asynchronous file reading operation is encapsulated in a uv_fs_t structure in the libuv layer, and req->cb is the callback function from the upper layer (the After function in node C++ layer: src/node_file.cc).

The main body of the io operation `uv__fs_work` is executed in the thread pool at this time. However, `uv__fs_done` must be called back in the thread of the event loop, because this function will eventually call back to the user's js code callback function, and all the code in the js code must be in the same thread.

#### Thread Pool Request Object —— struct uv__work
First, let's look at the definition of `uv__work`:
```c++
struct uv__work {
  void (*work)(struct uv__work *w);
  void (*done)(struct uv__work *w, int status);
  struct uv_loop_s* loop;
  void* wq[2];
};
```

Let's take a look at what `uv__work_submit` does:
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

`uv__work_submit` encapsulates `uv__fs_work` and `uv__fs_done` into a `uv__work` structure, which represents a request for a thread operation. The request is submitted to the thread pool through `post`.

In the `post` function, `QUEUE_INSERT_TAIL` adds the `uv__work` object to the `wq` linked list. `wq` is a global static variable. That is to say, all threads in the process space share the same `wq` linked list.

When `post` sends a signal to the corresponding condition variable `cond` through `uv_cond_signal`, a thread that is suspended and waiting in `uv_cond_wait` is activated.

The worker thread continues to execute, takes out `w` from `wq`, and executes `w->work()`.

After the worker thread completes the task, it calls `uv_async_send` to notify the main thread's unified IO observer to execute the callback.

### Callback

### Summary
Through the sorting of request objects at each layer, the context of a read request is also detailed, giving readers a rational understanding.

### Reference







