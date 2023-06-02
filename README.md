
# 《Deep into Node.js》: Core Concepts and Source Code Analysis

This book analyzes the source code of Node.js based on node v6.0.0.

The source code analysis includes (libuv, v8), and requires a certain level of C and C++ foundation. The source code of Node.js is full of developers' wisdom and the spirit of pursuing the ultimate, including but not limited to:

- System architecture

- Design patterns

- Performance optimization

- Tricks

This book explains the core concepts of node's asynchronous IO and event loop by analyzing the implementation of node's core modules, helping developers to better use Node.js.

By tracing the development issues of the node community, we can explore the changes and evolution of node and learn the design philosophy of node.js.

# Table of content

* [Overview](chapter1/README.md)
  * [Architecture Overview](chapter1/chapter1-0.md)
  * [Why libuv](chapter1/chapter1-1.md)
  * [V8 Concepts](chapter2/chapter2-0.md)
  * [C++ and JS Interaction](chapter2/chapter2-1.md)
* [Starting from Hello World](chapter1/chapter1-2.md)
* [Module Loading](chapter2/chapter2-2.md)
* [Global Object](chapter12/chapter12-1.md)
* [Event Loop](chapter5/chapter5-1.md)
* [Timer Interpretation](chapter3/chapter3-1.md)
* [Yield Magic](chapter3/chapter3-2.md)
* [Buffer](chapter6/chapter6-1.md)
* [Event](chapter7/chapter7-1.md)
* [Domain](chapter13/chapter13-2.md)
* [Stream](chapter8/chapter8-1.md)
* [Net](chapter9/README.md)
  * [Socket](chapter9/chapter9-1.md)
  * [Building Applications](chapter9/chapter9-2.md)
  * [Encryption](chapter9/chapter9-3.md)
* [HTTP](chapter10/README.md)
  * [HTTP Server](chapter10/chapter10-1.md)
  * [HTTP Client](chapter10/chapter10-2.md)
* [FS File System](chapter11/README.md)
  * [File System](chapter11/chapter11-2.md)
  * [File Abstraction](chapter11/chapter11-1.md)
  * [IO](chapter11/chapter11-3.md)
  * [libuv Selection](chapter11/chapter11-4.md)
  * [File IO](chapter11/chapter11-5.md)
  * [Fs Essence](chapter11/chapter11-6.md)
* [Process](chapter13/README.md)
  * [Child Process](chapter13/chapter13-1.md)
  * [Cluster](chapter4/chapter4-1.md)
* [Node.js Pitfalls](chapter14/chapter14-5.md)
* [Others](chapter14/README.md)
  * [Node.js & Android](chapter14/chapter14-1.md)
  * [Node.js & Docker](chapter14/chapter14-2.md)
  * [Node.js Tuning](chapter14/chapter14-3.md)
* [Appendix](chapter14/chapter14-0.md)


This book is copyrighted by the author. 
Unauthorized reproduction in any form is prohibited.

- Contact the author @江凌 Weibo: http://weibo.com/yangjianghua

- Email: yjhjstz#gmail.com, Blog: http://alinode.aliyun.com

This book is still being written. Welcome readers to discuss https://github.com/yjhjstz/deep-into-node/issues

**If you think it's good, please buy me a cup of coffee, welcome to Star, submit PR**



