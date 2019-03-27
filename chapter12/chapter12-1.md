
## Global对象

所有属性都可以在程序的任何地方访问，即全局变量。在javascript中，通常window是全局对象，而node.js的全局对象是global，所有全局变量都是global对象的属性，如：console、process等。


### 全局对象与全局变量
global最根本的作用是作为全局变量的宿主。满足以下条件成为全局变量。

- 在最外层定义的变量
- 全局对象的属性
- 隐式定义的变量（未定义直接赋值的变量）

node.js中不可能在最外层定义变量，因为所有的用户代码都是属于当前模块的，而模块本身不是最外层上下文。node.js中也不提倡自定义全局变量。

**Node提供以下几个全局对象，它们是所有模块都可以调用的。**
- global：表示Node所在的全局环境，类似于浏览器的window对象。需要注意的是，如果在浏览器中声明一个全局变量，实际上是声明了一个全局对象的属性，比如var x = 1等同于设置window.x = 1，但是Node不是这样，至少在模块中不是这样（REPL环境的行为与浏览器一致）。在模块文件中，声明var x = 1，该变量不是global对象的属性，global.x等于undefined。这是因为模块的全局变量都是该模块私有的，其他模块无法取到。

- process：该对象表示Node所处的当前进程，允许开发者与该进程互动。

- console：指向Node内置的console模块，提供命令行环境中的标准输入、标准输出功能。

**Node还提供一些全局函数。**
- setTimeout()：用于在指定毫秒之后，运行回调函数。实际的调用间隔，还取决于系统因素。间隔的毫秒数在1毫秒到2,147,483,647毫秒（约24.8天）之间。如果超过这个范围，会被自动改为1毫秒。该方法返回一个整数，代表这个新建定时器的编号。
- clearTimeout()：用于终止一个setTimeout方法新建的定时器。
- setInterval()：用于每隔一定毫秒调用回调函数。由于系统因素，可能无法保证每次调用之间正好间隔指定的毫秒数，但只会多于这个间隔，而不会少于它。指定的毫秒数必须是1到2,147,483,647（大约24.8天）之间的整数，如果超过这个范围，会被自动改为1毫秒。该方法返回一个整数，代表这个新建定时器的编号。
- clearInterval()：终止一个用setInterval方法新建的定时器。
- require()：用于加载模块。
- Buffer()：用于操作二进制数据。

**伪全局变量。**
* _filename：指向当前运行的脚本文件名。
* _dirname：指向当前运行的脚本所在的目录。
除此之外，还有一些对象实际上是模块内部的局部变量，指向的对象根据模块不同而不同，但是所有模块都适用，可以看作是伪全局变量，主要为module, module.exports, exports等。

### module.exports vs exports

如果想不借助global，在不同模块之间共享代码，就需要用到exports属性。令人有些迷惑的是，在node.js里，还有另外一个属性，是module.exports。一般情况下，这2个属性的作用是一致的，但是如果对exports或者module.exports赋值的话，又会呈现出令人奇怪的结果。


首先，exports和module.exports都是某个对象的引用（reference），初始情况下，它们指向同一个object，如果不修改module.exports的引用的话，这个object稍后会被导出。
```shell
  exports  module.exports
    |         /
    |        /
    V       V
     Object
```

所以如果只是给对象添加属性，不改变exports和module.exports的引用目标的话，是完全没有问题的。

但是有时候，希望导出的是一个构造函数，那么一般会这么写：
```js
// b.js
module.exports = function (name, age) {
    this.name = name;
    this.age = age;
}

exports.sex = "male";
```
```js
var Person = require("./b");
var person = new Person("Tony", 33);
console.log(person); // {name:"Tony", age:33}
console.log(Person.sex); // undefined
```
这个sex属性不会导出，因为引用关系已经改变：
```shell
  exports  module.exports
    |          |
    |          |
    V          V
   Object   function
```

而node.js导出的，永远是module.exports指向的对象，在这里就是function。所以exports指向的那个object，现在已经不会被导出了，为其增加的属性当然也就没用了。


如果希望把sex属性也导出，就需要这样写：
```js
exports = module.exports = function (name, age) {
    this.name = name;
    this.age = age;
}

exports.sex = "male";
```




### 总结

* node.js 设计的2个导出引用的对象，反而增加了迷惑性。
* 避免污染全局空间。


### 参考


