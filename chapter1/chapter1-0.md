## 架构一览

### 体系架构
Node.js主要分为四大部分，Node Standard Library，Node Bindings，V8，Libuv，架构图如下:
![node.js](a9e67142615f49863438cc0086b594e48984d1c9.jpeg)
- Node Standard Library 是我们每天都在用的标准库，如Http, Buffer 模块。
- Node Bindings 是沟通JS 和 C++的桥梁，封装V8和Libuv的细节，向上层提供基础API服务。
- 这一层是支撑 Node.js 运行的关键，由 C/C++ 实现。
  - V8 是Google开发的JavaScript引擎，提供JavaScript运行环境，可以说它就是 Node.js 的发动机。
  - Libuv 是专门为Node.js开发的一个封装库，提供跨平台的异步I/O能力.
  - C-ares：提供了异步处理 DNS 相关的能力。
  - http_parser、OpenSSL、zlib 等：提供包括 http 解析、SSL、数据压缩等其他的能力。

### 代码结构
树形结构查看，使用 tree 命令
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
进一步查看 `deps`目录：
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
`node.js`核心依赖六个第三方模块。其中核心模块 http_parser, uv, v8这三个模块在后续章节我们会陆续展开。 `gtest`是C/C++单元测试框架。




