
## Architecture Overview

### System Architecture
Node.js is mainly divided into four parts: Node Standard Library, Node Bindings, V8, and Libuv. The architecture diagram is as follows:
![node.js](a9e67142615f49863438cc0086b594e48984d1c9.jpeg)
- Node Standard Library is the standard library we use every day, such as Http, Buffer modules.
- Node Bindings is the bridge between JS and C++, encapsulating the details of V8 and Libuv, and providing basic API services to the upper layer.
- This layer is the key to support the operation of Node.js, implemented by C/C++.
  - V8 is a JavaScript engine developed by Google, which provides a JavaScript runtime environment. It can be said that it is the engine of Node.js.
  - Libuv is a specially developed encapsulation library for Node.js, which provides cross-platform asynchronous I/O capabilities.
  - C-ares: Provides asynchronous processing of DNS-related capabilities.
  - http_parser, OpenSSL, zlib, etc.: Provide other capabilities including http parsing, SSL, data compression, etc.

### Code Structure
View the tree structure using the tree command
```shell
➜  nodejs git:(master) tree -L 1
.
├── AUTHORS
├── BSDmakefile
├── BUILDING.md
├── CHANGELOG.md
├── CODE_OF_CONDUCT.md
├── COLLABORATOR_GUIDE.md
├── CONTRIBUTING.md
├── GOVERNANCE.md
├── LICENSE
├── Makefile
├── README.md
├── ROADMAP.md
├── WORKING_GROUPS.md
├── android-configure
├── benchmark
├── common.gypi
├── config.gypi
├── config.mk
├── configure
├── deps
├── doc
├── icu_config.gypi
├── lib
├── node.gyp
├── out
├── src
├── test
├── tools
└── vcbuild.bat
```
Further view the `deps` directory:
```shell
➜  nodejs git:(master) tree deps -L 1
deps
├── cares
├── gtest
├── http_parser
├── npm
├── openssl
├── uv
├── v8
└── zlib
```
The core of `node.js` depends on six third-party modules. Among them, the core modules http_parser, uv, and v8 will be introduced in subsequent chapters. `gtest` is a C/C++ unit testing framework.


