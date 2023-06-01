
# V8 bailout reasons
Examples, explanations and suggestions for v8 bailout reasons. Help `alinode` users optimize according to CPU-Profiler prompts.

## Index
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

* Simple example

```js
// sloppy mode only
function test(a) {
  if (arguments.length < 2) {
    a = 0;
  }
}
```

* Why
  * Only occurs when reassigning parameters in a function.

* Advices
  * You cannot reassign variable a.
  * It is better to use strict mode.
  * V8's latest TurboFan will have optimizations [#1][1].


### Bad value context for arguments value

* Simple example

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
  * Requires further specification of the `arguments` array.

* Advices
  * You can read: https://github.com/petkaantonov/bluebird/wiki/Optimization-killers#3-managing-arguments
  * You can loop through `arguments` to create a new array [Unsupported phi use of arguments](#unsupported-phi-use-of-arguments)
  * V8's latest TurboFan will have optimizations [#1][1].

* External links
  * https://github.com/bevry/taskgroup/issues/12
  * [More][7]

### ForInStatement with non-local each variable

* Simple example

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
  * Only pure local variables can be used in for...in.
  * https://github.com/petkaantonov/bluebird/wiki/Optimization-killers#5-for-in

* External links
  * https://github.com/mbostock/d3/pull/2686


### Object literal with complex property

* Simple example

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
  * Simplify the Object.

### ForInStatement is not fast case

* Simple example
```js
for (var prop in obj) {
  /* lots of code */
}
```

* Why
  * Too much code in the for loop.

* Advices
  * Extract the code in the for loop into a function.


### Reference to a variable which requires dynamic lookup

* Simple example
```js
// sloppy mode only
function test() {
  with ({x:1}) {
    return x;
  }
}
```

* Why
  * Compilation fails to locate the variable at compile time, and Crankshaft needs to dynamically look it up again. [#3][3]

* Advices
  * TurboFan can optimize.


### TryCatchStatement

* Simple example

```js
// strict & sloppy modes OR // sloppy mode only
function func() {
  return 3;
  try {} catch(e) {}
}
```

* Why
  * try/catch makes the control flow unstable and difficult to optimize at runtime.
* Advices
  * Do not use try/catch in heavily loaded functions.
  * Can be refactored as `try {  func() } catch`


### TryFinallyStatement

* Simple example

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
  * Crankshaft cannot determine whether `_arguments` is an object or an array.
  * [Learn more](http://mrale.ph/blog/2015/11/02/crankshaft-vs-arguments-object.html)

* Advices
  * It is best to operate on a copy of `arguments`.
  * TurboFan can optimize [#1][1].
```



### Yield

* Simple example

```js
// strict & sloppy modes
function* test() {
  yield 0;
}
```

* Why
  * Generator state preservation and restoration is achieved by copying function stack frames, which is not suitable for optimized compilers.

* Advices
  * Currently, there is no need to consider this issue. TurboFan can optimize.

* External links:
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




