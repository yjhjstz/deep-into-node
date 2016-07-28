

### javascript 的坑

#### 弱类型
* 判断对象是否不存在使用if(!obj), 此时如果obj为'' 或 0 或 false,也会误认为不存在;
正确的写法也是纷繁复杂：见参考

* 字符串连接,一般会自动转成string, 但不巧如果头两个正好可以转数字,那么它会按数字相加.如:

```js
var a = 1,b = 'b',c = 10;
var s1 = a + c + b;
var s2 = '' + a + c + b;
console.log(s1); //"11b"
console.log(s2); //"110b"

```

保险起见：注意使用 `Number`转换。

#### this指针

* 一般调用时，this是窗口的全局对象，比如在浏览器中就是window对象
* `.call`和`.apply`方法调用时可以改变`this`的值
* 在`prototype`函数内部，this指向该类创造的实例对象

#### 排序 sort
JavaScript的Array的sort()方法就是用于排序的，但是排序结果可能让你大吃一惊：
```js
[10, 20, 1, 2].sort(); // [1, 10, 2, 20]
```
这是因为Array的sort()方法默认把所有元素先转换为String再排序，结果'10'排在了'2'的前面，因为字符'1'比字符'2'的ASCII码小。

如果不知道sort()方法的默认排序规则，直接对数字排序，绝对栽进坑里！
幸运的是，sort()方法也是一个高阶函数，它还可以接收一个比较函数来实现自定义的排序。


#### 异步处理
* 错误处理忘记 `return`.
* 一个异步方法中处理不当有可能会多次触发callback,又或者是一个callback都没触发,需要仔细处理逻辑流程.


#### 科学计算
* v8 引擎中 js 的 Number 对象的内部实现只有两种，一是smi（也就是小整数），二是 `double`。Node.js 根本没有 float!
  如果使用了 float , 注意存在精度的缺失。
* 位运算， javascript 只支持32位，超过32位的，需要用大数模拟。 https://github.com/justmoon/node-bignum

### 参考
* http://www.ruanyifeng.com/blog/2011/05/how_to_judge_the_existence_of_a_global_object_in_javascript.html