---
title: 深入理解 JS 函数：基础语法与函数调用
date: 2018-11-16 19:17:31
tags:
  - function
  - javascript
---

JavaScript 的类函数式编程无疑是该语言中最为强大的特性之一，但这种强大是有代价的 —— 要想深入理解并运用 JavaScript 函数的各种特性，就需要付出更多的系统性的努力。这篇文章大致是一份读书笔记，系统总结一下我近期学到的 JavaScript 函数基础语法及函数调用相关知识。

<!--more-->

## 函数类型

JavaScript 中的函数大致可以这样分类：

- 传统函数
  - 普通函数
  - 方法
  - 构造函数
- 箭头函数
- 生成器函数

### 普通函数

*（没什么必要讲叭*

<div class="note warning">需要注意匿名函数的 name 属性在 ES5 与 ES6 中有不同的取值 。例如`var f = function () {}`在 ES5 中有 `f.name === ""`，而在 ES6 中有`f.name === "f"` 。可以参考阮一峰老师的[《ECMAScript 6 入门》](http://es6.ruanyifeng.com/#docs/function#name-%E5%B1%9E%E6%80%A7) 。</div>

### 箭头函数

箭头函数是 ES6 中提出的新语法，大概形如`() => {expression}`，显得十分直观。同样可以参考阮一峰老师的[《ECMAScript 6 入门》](http://es6.ruanyifeng.com/#docs/function#%E7%AE%AD%E5%A4%B4%E5%87%BD%E6%95%B0)来学习语法细节。这里就记录几点需要特别注意的：

- 箭头函数的`this` 对象与执行上下文无关，其`this` 对象始终指向创建时所在的对象 —— 因为箭头函数本身并没有`this` 对象；
- 箭头函数没有`prototype`属性，而普通函数都具有该属性；
- 箭头函数没有`arguments` 属性，但可以使用 ES6 引入的`...rest`语法；
- 箭头函数没有`super` 属性，无法使用`new` 操作符，也无法使用`yield` 操作符。

### 生成器函数

生成器函数同样是 ES6 中提出的新语法，大概是模仿了 python 中的概念。由于生成器的概念比较难以理解，再加上其异步编程的特性，有时间再单独用一篇文章讨论生成器函数。

## 参数传递

### 默认参数值

在 ES5 中，通常使用`||`来设置函数参数默认值，例如：

```javascript
// default value of x is 0
function f (x) {
    x = x || 0;
    return x;
}
```

但是在 ES6 中，你可以使用更优雅的、直觉性的语法：

```javascript
function f (x = 0) { return x; }
```

一个容易忽略的地方是，参数默认值是惰性求值的。例如：

```javascript
var p = 1, f = (x = p) => x * x;
f() === 1 // true
p = 2;
f() === 4 // true
```

### 可变长参数表

JavaScript 函数（箭头函数除外）默认用有一个类数组的可变长参数表`arguments` 。这个`arguments` 参数包含函数的所有参数。例如：

```javascript
var f = function (x) { console.log(arguments); }
f(1, 2, 3); // Arguments(3) [1, 2, 3 ...]
```

需要注意的是`arguments` 参数并非一个真正的数组，它是一个“类数组”的对象。如果有必要，我们可以通过`Array.prototype.slice.call(arguments)`（ES5）或者`Array.from(arguments)`（ES6）来进行数组转换。

在 ES6 中，你还可以使用`...rest` 语法来收集剩余参数 。例如：

```javascript
var f = function (x, ...rest) { console.log(rest); }
f(1, 2, 3) // [2, 3]
```

### 传递值还是引用？

这一点几乎是常见编程语言的共同坑点。在 JavaScript 中调用函数时，参数究竟是以值的形式还是对原有参数的引用的形式传递呢？直接给出解释并不直观，就先从例子开始吧：

```javascript
var a = 1, b = {}, c = {};
var f = function (a, b, c) {
    a = 2;
    b.item = 1;
    c = {item: 1};
}
f(a, b, c);
console.log(a, b, c); // 1 {item: 1} {}
```

执行完这段代码你会发现，变量a、c 的值并未改变，但变量b 的值改变了。我们暂时可以得出这样的结论：对于变量a，JS 采用了按值传递；对于变量b，就是典型的按引用传递 。问题就在于，为什么变量c 的值没有改变呢？

问题的答案就在于 JS 的求值策略（Evaluation Strategy）：

- 对于**基本类型变量**，JavaScript 采用了**按值传递（call by value）**策略 —— 即形参只拥有实参的值的一份拷贝，形参与实参占用的是完全不同的内存空间，修改形参的值无法影响实参；
- 对于**引用类型变量**， JavaScript 采用了**按共享传递（call by sharing）**策略 —— 即形参拥有实参的引用的一份拷贝（既不是按值传递的对象副本，也不是按引用传递的隐式引用）。按共享传递与按引用传递的主要区别就在于，对形参的赋值并不会影响实参的值，例如上例中的c 变量；但是对形参的修改会通过引用直接映射到实参上，例如上例中的b 变量 。

可以参考[这篇文章](https://segmentfault.com/a/1190000005794070)深入了解 JS 求值策略 。

那么新的问题来了，在 JavaScript 中哪些变量属于基本类型，哪些又属于引用类型呢？答案就是：

- `Boolean`，`Null`， `Number`，`String`，`Symbol`（ES6），`Undefined`属于基本类型 。可以看到它们都属于不可变（Immutable）类型；
- `Array`，`Date`，`Function`，`Object`，`RegExp`属于引用类型 。这些类型的变量都是可变（Mutable）的 。在比较两个引用类型变量是否相等时，JavaScript 直接比较它们的引用是否相等。例如`{} === {}`的结果是`false`，这是因为尽管它们目前值相等，但是引用了不同的堆内存空间。

<div class="note warning">注意：尽管看起来很像可变类型，`String`事实上是不可变的 。对字符串的修改操作最终都返回一个新的字符串 。如果想要深入了解 JS 变量类型，可以参考[这篇文章](https://segmentfault.com/a/1190000006752076) 。</div>

## 调用模式

### 函数调用模式

当一个函数并非一个对象的属性时，这个函数就被当作普通函数来调用 。在这种模式下，函数内部的`this` 会被绑定到全局对象 。例如在 Chrome 调试器中：

```javascript test https://www.google.com
(function () { console.log(this); })();
// Window {postMessage: f, blur: f, ...}
```

函数输出了全局对象`Window`。然而需要注意的是，在严格模式中，函数调用模式下的`this` 对象会被绑定为`undefined` 。

### 方法调用模式

当一个函数作为一个对象的属性时，它被称作一个方法 。当函数作为方法被调用时，它内部的`this`变量被绑定到调用它的对象 。例如：

```javascript
var counter = {
    value: 0,
    increase (x = 1) {
        this.value += x;
        return this.value;
    }
}
couter.increase(); // 1

```

### 构造函数模式

当函数或方法在调用前使用了`new` 操作符时，它就构成了构造函数调用 。

#### 显式 / 隐式返回

通常来说，构造函数不使用`return` 语句 。构造函数在被调用时会隐式创建并返回一个新的对象，这个对象继承自构造函数的`prototype` 对象 。在对这个对象进行完初始化以后，该对象会被自动返回 。但是如果函数内部显式地声明了`return` 语句，那么此时该构造函数就是“显式返回”的，这时如果：

- 显式返回值是个对象，那么该对象就会覆盖隐式创建的新对象；
- 显式返回值是原始值或没有返回值，那么构造函数仍会返回隐式创建的新对象 。

例如可以看看这个例子：

```javascript
var f = function (x) { this.x = x; },
	g = function (x) { this.x = x; return {x: 1}; },
    h = function (x) { this.x = x; return 1; };
new f(); // {x: undefined}
new g(); // {x: 1}
new h(); // {x: undefined}
```

#### 上下文绑定

即使构造函数是某个对象的方法，其上下文依然会被绑定到创建的新对象 。例如在表达式`new o.m()` 中，构造函数内部的`this` 会被绑定到新对象而非`o` 。

#### `prototype` 与`__proto__`

对象都具有`__proto__`属性，这是 JavaScript 内部实现原型继承链的依赖机制；而只有构造函数拥有`prototype` 属性 ，一些内置函数（例如`alert` 和`prompt`）和箭头函数就没有`prototype` 属性 。由于我们所创建的函数一般都可以视作构造函数，所以一般而言函数都具有`prototype` 属性 。

在构造函数模式中，新创建的对象会从构造函数的`prototype` 对象继承属性 。例如：

```javascript
var f = new Function();
f.prototype.x = 1
var o = new f();
o.x === 1                     // true
o.__proto__ === f.prototype   // true
f.prototype.constructor === f // true
console.log(o);               // {}
console.log(o.__proto__)      // {x: 1, constructor: f}
```

一般来说，函数的`prototype` 属性是一个拥有`constructor` 属性且`constructor` 属性值即为该函数的对象 。

### 间接调用模式

在 JavaScript 中，函数也是对象，因而也拥有自己的方法 。而以下三个函数方法允许我们为函数指定调用时的上下文（即指定`this`对象）并间接调用它。

#### apply 方法

`apply`方法接收两个参数：第一个是要绑定的`this` 对象，第二个是一个“类数组”参数表 。例如：

```javascript
var f = function (y) { console.log(this.x + y); };
f.apply({x: 1}, [2]); // 3
```

有几点需要注意：

- 对于箭头函数来说间接调用模式往往是无意义的；因为它没有`this` 属性可供我们绑定；
- 在非严格模式下，传入`null` 或`undefined` 会被全局对象（例如`Window`）代替；
- 在非严格模式下，传入原始值会被相应的包装对象代替（例如`1`被替换为`new Number(1)`）。

#### bind 方法

严格来说，`bind`方法并不属于间接调用模式 。调用`bind`方法会返回一个新的函数，这个函数会表现得像原函数被绑定为某个对象的方法一样 。例如：

```javascript
var f = function () { console.log(this.x); },
	g = f.bind({x: 1});
f(); // undefined
g(); // 1
```

这个方法对于箭头函数同样没有意义。

#### call 方法

`call` 方法和`apply`方法没有太大区别，只不过`call`方法接受展开来的参数列表，而非一个“类数组”对象 。例如对于上面的`apply`方法的例子你可以用`call`方法这样改写：`f.call({x: 1}, 2)` 。

------

**参考文献：**

- [《JavaScript 权威指南》](https://book.douban.com/subject/10549733/)
- [《JavaScript 语言精粹》](https://book.douban.com/subject/3590768/)
- [《ECMAScript 6 入门》](http://es6.ruanyifeng.com/)
- [SegmentFault：JS中的值是按值传递，还是按引用传递呢？](https://segmentfault.com/a/1190000005794070)
- [SegmentFault：JavaScript 深入了解基本类型和引用类型的值](https://segmentfault.com/a/1190000006752076)
- [SegmentFault：为什么实例没有prototype属性？什么时候对象会有prototype属性呢？](https://segmentfault.com/q/1010000004292072)