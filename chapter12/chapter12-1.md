
## Global Object

All properties can be accessed anywhere in the program, that is, global variables. In javascript, window is usually the global object, while the global object of node.js is global, and all global variables are properties of the global object, such as console, process, etc.

### Global Object and Global Variables
The most fundamental function of global is to host global variables. To become a global variable, the following conditions must be met.

- Variables defined at the outermost level
- Properties of the global object
- Implicitly defined variables (variables assigned without being defined)

It is impossible to define variables at the outermost level in node.js, because all user code belongs to the current module, and the module itself is not the outermost context. Node.js also does not advocate custom global variables.

**Node provides the following global objects, which can be called by all modules.**
- global: Represents the global environment in which Node is located, similar to the window object in a browser. It should be noted that if a global variable is declared in a browser, a property of the global object is actually declared, such as var x = 1 is equivalent to setting window.x = 1, but Node is not like this, at least not in the module (the behavior of the REPL environment is consistent with the browser). In the module file, declaring var x = 1, the variable is not a property of the global object, global.x is equal to undefined. This is because the global variables of the module are private to the module and cannot be accessed by other modules.

- process: This object represents the current process in which Node is located and allows developers to interact with the process.

- console: Points to the built-in console module in Node, providing standard input and output functions in the command line environment.

**Node also provides some global functions.**
- setTimeout(): Runs the callback function after the specified number of milliseconds. The actual call interval also depends on system factors. The interval in milliseconds is between 1 millisecond and 2,147,483,647 milliseconds (about 24.8 days). If it exceeds this range, it will be automatically changed to 1 millisecond. This method returns an integer representing the number of the newly created timer.
- clearTimeout(): Used to terminate a timer newly created by the setTimeout method.
- setInterval(): Calls the callback function every certain number of milliseconds. Due to system factors, it may not be possible to guarantee that the interval between each call is exactly the specified number of milliseconds, but it will only be more than this interval, not less than it. The specified number of milliseconds must be an integer between 1 and 2,147,483,647 (about 24.8 days). If it exceeds this range, it will be automatically changed to 1 millisecond. This method returns an integer representing the number of the newly created timer.
- clearInterval(): Terminates a timer newly created by the setInterval method.
- require(): Used to load modules.
- Buffer(): Used to manipulate binary data.

**Pseudo-global variables.**
* _filename: Points to the name of the currently running script file.
* _dirname: Points to the directory where the currently running script is located.
In addition, there are some objects that are actually local variables within the module, and the objects they point to are different depending on the module, but they are applicable to all modules and can be regarded as pseudo-global variables, mainly for module, module.exports, exports, etc.

### module.exports vs exports

If you want to share code between different modules without relying on global, you need to use the exports property. What is confusing is that in node.js, there is another property, which is module.exports. In general, the functions of these two properties are the same, but if you assign to exports or module.exports, strange results will be presented.

First of all, exports and module.exports are references to some objects. Initially, they point to the same object. If you do not modify the reference target of module.exports, this object will be exported later.
```shell
  exports  module.exports
    |         /
    |        /
    V       V
     Object
```

So if you just want to add properties to the object without changing the reference targets of exports and module.exports, there is no problem.

But sometimes, if you want to export a constructor, you generally write it like this:
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
This sex property will not be exported because the reference relationship has changed:
```shell
  exports  module.exports
    |          |
    |          |
    V          V
   Object   function
```

What node.js exports is always the object pointed to by module.exports, which is function here. So the object pointed to by exports, which is now no longer exported, is of course useless for the properties added to it.


If you want to export the sex property, you need to write it like this:
```js
exports = module.exports = function (name, age) {
    this.name = name;
    this.age = age;
}

exports.sex = "male";
```




### Summary

* The two export reference objects designed by node.js have increased confusion.
* Avoid polluting the global space.


### Reference




