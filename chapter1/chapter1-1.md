
## Why libuv

#### Background
Node.js started in 2009 as a project that allowed JavaScript code to be executed outside of the browser. Node.js uses Google's V8 parsing engine and Marc Lehmann's libev. Node.js combines the event-driven I/O model with a programming language (JavaScript) that is suitable for this model. As node.js became increasingly popular, it needed to support Windows as well, but libev could only run on Unix environments. The mechanism corresponding to kernel event notification on the Windows platform is IOCP, which is based on kqueue (FreeBSD) or (e)poll (Linux). Libuv provides a cross-platform abstraction that allows the platform to determine whether to use libev or IOCP. In version node-v0.9.0, libuv removed the content of libev.

#### Why asynchronous

Let's take a look at a table:

| Category | Operation | Time cost |
| --- | --- | ---   |
| Cache | L1 cache | 1 nanosecond |
|     | L2 cache | 4 nanoseconds |
|     | Main memory | 100 ns |
|     | SSD random read | 16,000 ns |
| I/O | Round trip in the same data center | 500,000 ns |
|     | Physical disk seek | 4,000,000 ns |

We can see that even SSD access is still a slow device compared to high-speed CPUs. Therefore, the event-driven IO model was born to solve the problem of synchronous waiting for slow devices or access by high-speed devices. This is not unique to libuv, and NIO natively supported by the Linux kernel is also based on this idea. However, libuv unifies network access, file access, and achieves cross-platform.

### libuv architecture
![](FuX1qcGJgwYtX9zNbBAOSaQeD8Qz.png)
From left to right, it is divided into two parts, one is related to network I/O requests, and the other is composed of file I/O, DNS Ops, and User code requests.

From the figure, it can be seen that the underlying support mechanism for asynchronous processing is completely different for network I/O-related requests and another type of requests represented by File I/O.

For network I/O-related requests, depending on the OS platform, epoll on Linux, kqueue on OSX and BSD-like OS, event ports on SunOS, and IOCP mechanisms on Windows are used.

For requests represented by File I/O, thread pool is used. The asynchronous request processing is implemented using the thread pool method, which can be well supported on various OSs.

I once suggested to the libuv community to replace the thread pool with native NIO on the Linux platform and implemented it [2]. The test found a 3% improvement. Considering the dependence of NIO on the kernel version, the asynchronous request processing is implemented using the thread pool method, which can be well supported on various OSs, which I believe is the result of the libuv author's careful consideration.

In the later detailed module source code analysis, they will be analyzed one by one.

### Reference

- 1. http://luohaha.github.io/Chinese-uvbook/
- 2. https://github.com/libuv/libuv/issues/461


