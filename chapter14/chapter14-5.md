
### JavaScript Pitfalls

#### Weak Typing
* To check if an object does not exist, using `if(!obj)` will also mistake `''`, `0`, or `false` as non-existent. 
The correct way to do this is more complex: see reference.

* String concatenation will usually automatically convert to a string, but if the first two happen to be able to be converted to numbers, it will add them as numbers. For example:

```js
var a = 1,b = 'b',c = 10;
var s1 = a + c + b;
var s2 = '' + a + c + b;
console.log(s1); //"11b"
console.log(s2); //"110b"

```

To be safe: use `Number` conversion.

#### This Pointer

* Normally, when called, `this` is the global object of the window, for example in a browser it is the window object.
* `.call` and `.apply` methods can change the value of `this`.
* In a `prototype` function, `this` points to the instance object created by the class.

#### Sorting
The sort() method of JavaScript's Array is used for sorting, but the sorting result may surprise you:
```js
[10, 20, 1, 2].sort(); // [1, 10, 2, 20]
```
This is because the sort() method of Array defaults to converting all elements to String before sorting, so '10' is ranked ahead of '2' because the character '1' has a smaller ASCII code than the character '2'.

If you don't know the default sorting rules of the sort() method, sorting numbers directly will definitely get you into trouble!
Fortunately, the sort() method is also a higher-order function, which can receive a comparison function to implement custom sorting.


#### Asynchronous Processing
* Forgetting to `return` when handling errors.
* Improper handling in an asynchronous method may trigger the callback multiple times, or not trigger any callback at all. Careful handling of the logical flow is required.


#### Scientific Calculation
* The internal implementation of the Number object in the v8 engine of JavaScript has only two types: smi (small integers) and `double`. Node.js does not have float at all!
  If you use float, be aware of the loss of precision.
* Bitwise operations are only supported up to 32 bits in JavaScript. For more than 32 bits, you need to simulate with big numbers. https://github.com/justmoon/node-bignum

### Reference
* http://www.ruanyifeng.com/blog/2011/05/how_to_judge_the_existence_of_a_global_object_in_javascript.html

