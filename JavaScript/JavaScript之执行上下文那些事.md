![image](https://user-images.githubusercontent.com/51777605/158967142-ec19fab2-0d99-47ca-a0c9-ebc7462c4b90.png)

## 执行上下文与执行上下文栈

JavaScript 中存在函数调用函数的关系，我们将**函数调用的关系**用栈结构维护起来，这就是我们的执行上下文栈，也称函数调用栈。**全局代码、函数代码、eval代码**执行时push进执行上下文栈中，执行上下文中有一些属性，**变量环境(作用域链)、词法环境、this**。

## 变量环境

**概念**

变量环境中存储了上下文中定义的变量和函数申明，我们称存储的这些内容为变量对象。因为变量环境中变量对象的生命周期不同，分为**全局上下文和函数上下文**(activation object)，函数上下文会在函数执行完时销毁，全局上下文会在整个代码执行完才销毁。

**变量环境存储内容**

1. var 定义的变量声明
2. 函数声明
3. outer

**var 变量提升**

```jsx
for (var i = 0; i < 10; i++) {
    ...
}
console.log(window.i); // 10
```

由于 var 定义的变量会进行变量提升，所以上述代码会打印 i 为 10，这个问题算是一个历史性问题， JavaScript 在设计时为了便于管理变量采用了变量整体提升，产生了一些与直觉不符合的代码片段，后面通过 let、const 定义了块级作用域，解决了这一问题。

var 变量提升带来了那些问题呢？

1. 变量声明提升，允许一段代码中变量名重复定义，导致不经意间修改了变量值
2. 应该销毁的变量没有销毁

## **词法环境**

ES6 之前我们采用 var 定义变量，var 变量提升这一机制导致在 ES6 之前 JavaScript 只有全局作用域与函数作用域，但相比与别的语言都是支持块级作用域的，我们需要引入块级作用域解决 var 变量带来的问题。

💡 全局作用域：对象在代码中的任何地方都能访问，其生命周期伴随着页面的生命周期。

💡 函数作用域：在函数内部定义的变量或者函数，并且定义的变量或者函数只能在函数内部被访问。函数执行结束之后，函数内部定义的变量会被销毁

💡 块级作用域：{} 包裹的一段代码就形成一段块级作用域

ES6 引入 let、const 定义变量，提供了块级作用域。那么 ES6 是如何支持块级作用域的呢？在词法环境内部，维护了一个小型栈结构，栈底是函数最外层的变量，进入一个作用域块后，就会把该作用域块内部的变量压到栈顶；当作用域执行完成之后，该作用域的信息就会从栈顶弹出，这就是词法环境的结构。

![image](https://user-images.githubusercontent.com/51777605/158968087-c7bbb00e-30ab-4d40-ba5a-5a8854545232.png)


1）暂时死区

let 和 const 声明的变量不会被提升到作用域顶部，如果在声明之前访问这些变量，会导致报错，我们称这种现象为临时死区(Temporal Dead Zone)，简写为 TDZ。

```jsx
console.log(typeof value); // Uncaught ReferenceError: value is not defined
let value = 1;
```

2）let 和 const 定义的变量不可重复申明，声明的全局变量不会挂在顶层对象下面；const 声明变量后需要立即赋值，简单类型一旦声明就不能再更改，复杂类型(数组、对象等)指针指向的地址不能更改，内部数据可以更改(const 不能更改是指指针指向的地址不能更改，所以简单类型表现为不能更改，复杂类型更改值不会导致他的指针地址变化)

3）循环中的块级作用域

这个部分会涉及到闭包的知识，建议先定位闭包理解闭包后再来看这个例子：

```jsx
var funcs = [];
for (var i = 0; i < 3; i++) {
    funcs[i] = function () {
        console.log(i);
    };
}
funcs[0](); // 3
```

解决方案1：引入闭包解决

```jsx
var funcs = [];
for (var i = 0; i < 3; i++) {
    funcs[i] = (function(i){
        return function() {
            console.log(i);
        }
    }(i))
}
funcs[0](); // 0
```

解决方案2：ES6 提供的 let 定义变量

```jsx
var funcs = [];
for (let i = 0; i < 3; i++) {
    funcs[i] = function () {
        console.log(i);
    };
}
funcs[0](); // 0
```

4）Babel 如何编译 let、const

我们先看一个简单的例子：

```jsx
let value = 1;
{
    let value = 2;
}
value = 3;
```

Babel 编译后发现最终还是使用 var，但是块级作用域内和外使用不同的变量名称；

```jsx
var value = 1;
{
    var _value = 2;
}
value = 3;
```

上述使用 let 解决 var 循环的例子中，Babel 又会如何编译呢？

```jsx
var funcs = [];

var _loop = function _loop(i) {
  funcs[i] = function () {
    console.log(i);
  };
};

for (var i = 0; i < 3; i++) {
  _loop(i);
}

funcs[0](); // 0
```

5）使用 ES5 实现 let 和 const

```jsx
// let
(function(){
	var a = 1;
	console.log(a); // 1
})()
console.log(a); // a is not defined

// const
var __const = function __const(key, value) {
  window[key] = value 
  Object.defineProperty(window, key, {
      enumerable: false,
      configurable: false,
      get: function () {
          return value
      },
      set: function (newValue) {
          if (newValue !== value) { // 当要对当前属性进行赋值时，则抛出错误！
              throw new TypeError('const 定义常量，不能修改')
          } else {
              return value
          }
      }
  })
  }
  __const('a', 10)
  console.log(a)
  for (let item in window) { // 因为const定义的属性在global下是不存在的，所以用到了enumerable: false来模拟这一功能
      if (item === 'a') { // 因为不可枚举，所以不执行
          console.log(window[item])
      }
  }
  // a = 20 // 报错
```

## 作用域链(outer)

**概念**

作用域规定了如何查找变量。作用域分为动态作用域和静态作用域，动态作用域在程序运行时确定变量的查找，例如 Shell 采用动态作用域；静态作用域也称词法作用域，是根据代码书写位置来查找变量，JavaScript 就是采用的静态作用域。

**变量环境中的 outer**

每个执行上下文中都有outer，他指向定义的时候所在的执行上下文，outer将不同的执行上下文串联起来，形成作用域链。我们查找变量时也是根据作用域链进行查找。

```jsx
function bar() {
    console.log(myName)
}
function foo() {
    var myName = "极客邦"
    bar()
}
var myName = "极客时间"
foo()
```

![image](https://user-images.githubusercontent.com/51777605/158968657-4e0c3bb1-991a-437e-b188-b33a24ad67fa.png)

**作用域链**

作用域链告诉我们一个变量的查找规则是怎么样的，将 outer 与词法环境结合我们可以按照下面这张图知道变量完整的查找过程：

![image](https://user-images.githubusercontent.com/51777605/158968867-137d529c-5e0a-41b8-8bc0-d3a7ec9b07b2.png)

## **闭包**

ECMAScript 中，闭包指的是：

1. 从理论角度：闭包是指那些能够访问自由变量(自由变量是指在函数中使用的，但既不是函数参数也不是函数的局部变量的变量)的函数；
2. 从实践角度：以下函数才算是闭包：
    1. 即使创建它的上下文已经销毁，它仍然存在（比如，内部函数从父函数中返回）
    2. 在代码中引用了自由变量

```jsx
var scope = "global scope";
function checkscope(){
    var scope = "local scope";
    function f(){
        return scope;
    }
    return f;
}

var foo = checkscope();
foo();
```

分析这段代码的执行上下文，在调用 f 函数时形成了闭包，f 函数中 scope 变量查找链为：f 函数上下文 —> checkscope 函数闭包 —> 全局执行上下文。

![image](https://user-images.githubusercontent.com/51777605/165919935-b8e74183-230b-4358-b0d6-707d5590cce6.png)

## **this**

💡 在对象内部的方法中使用对象内部的属性是一个非常普遍的需求，基于这个需求 JavaScript 提出了 this 机制，this 机制与作用域链是完全不同的两套机制，这点非常重要。在我们的架构中可以看出 this 是和执行上下文绑定的，所以 this 也有3种，全局执行上下文中的 this、函数中的 this 和 eval 中的 this。

**全局执行上下文中的 this**

全局执行上下文中的 this 是指向 window 对象的。这也是 this 和作用域链的唯一交点，作用域链的最顶端包含了 window 对象，全局执行上下文中的 this 也是指向 window 对象。

**函数执行上下文中的 this**

```jsx
function foo(){
 // 默认情况下调用一个函数，其执行上下文中的 this 也是指向 window 对象
  console.log(this) // window
}
foo()
```

call、apply、bind 修改函数 this 指向:

call 模拟

```jsx
Function.prototype.call2 = function(context){
    var context = context || window;
    context.fn = this;

    var args = [];
    for(var i = 1, len = arguments.length; i < len; i++) {
        args.push('arguments[' + i + ']');
    }

    let result = eval('context.fn(' + args +')');
    delete context.fn;
    return result;
}
```

apply 模拟

```jsx
Function.prototype.apply2 = function (context, arr) {
    var context = Object(context) || window;
    context.fn = this;

    var result;
    if (!arr) {
        result = context.fn();
    }
    else {
        var args = [];
        for (var i = 0, len = arr.length; i < len; i++) {
            args.push('arr[' + i + ']');
        }
        result = eval('context.fn(' + args + ')')
    }

    delete context.fn
    return result;
}
```

bind 模拟

```jsx
Function.prototype.bind2 = function (context) {

    if (typeof this !== "function") {
      throw new Error("Function.prototype.bind - what is trying to be bound is not callable");
    }

    var self = this;
    var args = Array.prototype.slice.call(arguments, 1);

    var fNOP = function () {};

    var fBound = function () {
        var bindArgs = Array.prototype.slice.call(arguments);
        return self.apply(this instanceof fNOP ? this : context, args.concat(bindArgs));
    }

    fNOP.prototype = this.prototype;
    fBound.prototype = new fNOP();
    return fBound;
}
```

**箭头函数与普通函数**

笔者认为 this 这一套机制是高度抽象的，普通函数中 this 的指向总是随着函数调用动态变化，这对于新手来说如何处理好 this 是困难的，所以在 ES6 新增了箭头函数。我们来比较一下箭头函数与普通函数。

箭头函数，引用 MDN 介绍：

> 箭头函数表达式的语法比函数表达式更短，并且不绑定自己的 this，arguments，super 或 new.target。这些函数表达式最适合用于非方法函数(non-method functions)，并且它们不能用作构造函数。
>

```jsx
// this 不同
function logThis() {
  console.log(this); // document
}
document.addEventListener('click', logThis);

const logThisArrow = () => {
  console.log(this); // window
};
document.addEventListener('click', logThisArrow);

// bind 无法修改箭头函数指向
function logThis() {
    console.log(this);
}
logThis.call({});       // {}

const logThisArrow = () => {
    console.log(this);
}
logThisArrow.call(42);  // window

// 箭头函数不适合做对象中的方法，但在 class 中可以作为方法使用
var obj = {
  i: 10,
  b: () => console.log(this.i, this),
  c: function() {
    console.log( this.i, this)
  }
}
obj.b(); // undefined Window
obj.c(); // 10, Object {...}

// 普通函数可以作为构造函数，使用 new 关键字实例化，箭头函数会抛出异常
function Foo(bar) {
  this.bar = bar;
}
const a = new Foo(42);  
console.log(a) // Foo {bar: 42}

const Bar = foo => {
    this.foo = foo;
};
const b = new Bar(42);  // TypeError: Bar is not a constructor
```

**this 优先级**

默认绑定

```jsx
var number = 1;
function baz() {
    console.log(this.number);
}
baz(); // 1
```

当函数 baz 被调用时，this.number 被解析成全局变量 number。函数在调用时，进行默认绑定，此时的 this 指向全局对象（非严格模式），严格模式下 this 为 undefined。

隐式绑定

```jsx
function baz() {
    console.log(this.number);
}
var object = {
    number: 1,
    baz: baz
};
object.baz(); // 1
```

函数baz()的声明方式，严格来说是不属于object对象的，但是调用位置会使用object上下文来引用函数。隐式绑定容易 this 丢失。

```jsx
function baz() {
    return function(){console.log(this.number)}
}
var object = {
    number: 1,
    baz: baz
};
var bar = object.baz();
var number = 2;
bar(); // 2
```

显式绑定

使用 apply、call、bind 进行的绑定

new 绑定

优先级排序

new绑定优先级 > 显示绑定优先级 > 隐式绑定优先级 > 默认绑定优先
