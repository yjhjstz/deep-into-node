# V8 bailout reasons
v8 bailout reasons 的例子, 解释和建议. 帮助`alinode`的用户根据 CPU-Profiler 的提示进行优化。

## 索引
### [Bailout reasons](#bailout-reasons-1)

* [Assignment to parameter in arguments object](#assignment-to-parameter-in-arguments-object)
* [Bad value context for arguments value](#bad-value-context-for-arguments-value)
* [ForInStatement with non-local each variable](#forinstatement-with-non-local-each-variable)
* [Object literal with complex property](#object-literal-with-complex-property)
* [ForInStatement is not fast case](#forinstatement-is-not-fast-case)
* [Reference to a variable which requires dynamic lookup](#reference-to-a-variable-which-requires-dynamic-lookup)
* [TryCatchStatement](#trycatchstatement)
* [TryFinallyStatement](#tryfinallystatement)
* [Unsupported phi use of arguments](#unsupported-phi-use-of-arguments)
* [Yield](#yield)


## Bailout reasons
### Assignment to parameter in arguments object

* 简单例子

```js
// sloppy mode only
function test(a) {
  if (arguments.length < 2) {
    a = 0;
  }
}
```

* Why
  * 只会在函数中重新赋值参数发生。

* Advices
  * 你不能给变量 a 重新赋值.
  * 最好使用 strict mode .
  * V8 最新的 TurboFan 会有优化 [#1][1].


### Bad value context for arguments value

* 简单例子

```js
// strict & sloppy modes
function test1() {
  arguments[0] = 0;
}

// strict & sloppy modes
function test2() {
  arguments.length = 0;
}

// strict & sloppy modes
function test3() {
  return arguments;
}

// strict & sloppy modes
function test4() {
  var args = [].slice.call(arguments);
}

// strict & sloppy modes
function test5() {
  var a = arguments;
  return function() {
    return a;
  };
}
```

* Why
  * 要求再具体化 `arguments` 数组.

* Advices
  * 可以读读: https://github.com/petkaantonov/bluebird/wiki/Optimization-killers#3-managing-arguments
  * 你可以循环 `arguments` 创建一个新的数组  [Unsupported phi use of arguments](#unsupported-phi-use-of-arguments)
  * V8 最新的 TurboFan 会有优化 [#1][1].

* 外部链接
  * https://github.com/bevry/taskgroup/issues/12
  * [更多][7]

### ForInStatement with non-local each variable

* 简单例子

```js
// strict & sloppy modes
function test1() {
  var obj = {};
  for(key in obj);
}

// strict & sloppy modes
function key() {
  return 'a';
}
function test2() {
  var obj = {};
  for(key() in obj);
}
```

* Why
  * https://github.com/yjhjstz/v8-git-mirror/blob/master/src/hydrogen.cc#L5254

* Advices
  * 只有纯局部变量可以用于 for...in 
  * https://github.com/petkaantonov/bluebird/wiki/Optimization-killers#5-for-in

* 外面链接
  * https://github.com/mbostock/d3/pull/2686


### Object literal with complex property

* 简单例子

```js
// strict & sloppy modes
function test() {
  return {
    __proto__: 3
  };
}
```

* Why
  
* Advices
  * 简化 Object。

### ForInStatement is not fast case

* 简单例子
```js
for (var prop in obj) {
  /* lots of code */
}
```

* Why
  * for 循环中包含太多的代码。

* Advices
  * for 循环中的提取代码提取为函数。


### Reference to a variable which requires dynamic lookup

* 简单例子
```js
// sloppy mode only
function test() {
  with ({x:1}) {
    return x;
  }
}
```

* Why
  * 编译时编译定位失败，Crankshaft需要重新动态查找。[#3][3]

* Advices
  * TurboFan可以优化。


### TryCatchStatement

* 简单例子

```js
// strict & sloppy modes OR // sloppy mode only
function func() {
  return 3;
  try {} catch(e) {}
}
```

* Why
  * try/catch 使得控制流不稳定，很难在运行时优化。
* Advices
  * 不要在负载重的函数中使用try/catch.
  * 可以重构为 `try {  func() } catch`


### TryFinallyStatement

* 简单例子

```js
// strict & sloppy modes OR // sloppy mode only
function func() {
  return 3;
  try {} finally {}
}
```

* Why
  * See [TryCatchStatement](#trycatchstatement)

* Advices
  * See [TryCatchStatement](#trycatchstatement)



### Unsupported phi use of arguments

* 简单例子

```js
// strict & sloppy modes
function test1() {
  var _arguments = arguments;
  if (0 === 0) { // anything evaluating to true, except a number or `true`
    _arguments = [0]; // Unsupported phi use of arguments
  }
}

// strict & sloppy modes
function test2() {
  var _arguments = arguments;
  for (var i = 0; i < 1; i++) {
    _arguments = [0]; // Unsupported phi use of arguments
  }
}

// strict & sloppy modes
function test3() {
  var _arguments = arguments;
  var again = true;
  while (again) {
    _arguments = [0]; // Unsupported phi use of arguments
    again = false;
  }
}
```

* Why
  * Crankshaft 无法知道 `_arguments`是 object 或 array. 
  * [深入了解](http://mrale.ph/blog/2015/11/02/crankshaft-vs-arguments-object.html)

* Advices
  * 最好操作 `arguments` 的拷贝.
  * TurboFan 可以优化 [#1][1].



### Yield

* 简单例子

```js
// strict & sloppy modes
function* test() {
  yield 0;
}
```

* Why
  * generator 状态保持、恢复通过拷贝函数栈帧实现，但在优化编译器中并不适用。

* Advices
  * 暂时不用考虑，TurboFan 可以优化。

* 外部链接：
  * https://groups.google.com/forum/#!topic/v8-users/KnnUb-u4rA8

---

[1]: https://chromium.googlesource.com/v8/v8/+/d3f074b23195a2426d14298dca30c4cf9183f203%5E%21/src/bailout-reason.h
[2]: https://codereview.chromium.org/1272673003
[3]: https://groups.google.com/forum/#!msg/google-chrome-developer-tools/Y0J2XQ9iiqU/H60qqZNlQa8J
[4]: https://github.com/GoogleChrome/devtools-docs/issues/53#issuecomment-37269998
[5]: https://github.com/GoogleChrome/devtools-docs/issues/53#issuecomment-140030617
[6]: https://github.com/GoogleChrome/devtools-docs/issues/53#issuecomment-145192013
[7]: https://github.com/GoogleChrome/devtools-docs/issues/53#issuecomment-147569505



### Resources

- [All bailout reasons in Chromium codebase](https://code.google.com/p/chromium/codesearch#chromium/src/v8/src/bailout-reason.h)
- [Bad value context for arguments value](https://gist.github.com/Hypercubed/89808f3051101a1a97f3)
- [I-want-to-optimize-my-JS-application-on-V8 checklist](http://mrale.ph/blog/2011/12/18/v8-optimization-checklist.html)
- [JavaScript: Performance loss on incorrect arguments using](http://techblog.dorogin.com/2015/05/performance-loss-on-incorrect-arguments-using.html)
- [Optimization killers](https://github.com/petkaantonov/bluebird/wiki/Optimization-killers)
- [OptimizationKillers](https://github.com/zhangchiqing/OptimizationKillers)
- [Performance Tips for JavaScript in V8](http://www.html5rocks.com/en/tutorials/speed/v8/)
- [thlorenz/v8-perf](https://github.com/thlorenz/v8-perf/blob/master/compiler.md)



