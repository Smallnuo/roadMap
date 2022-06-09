![image](https://user-images.githubusercontent.com/51777605/166630393-a45af1b3-74ba-435f-b556-573347496c57.png)

## Iterator

**遍历方法对比**

**Iterator 概念**

Iterator 提供了一种统一的接口机制，为各种不同数据结构提供统一的访问机制。定义 Iterator 就是提供一个具有 next() 方法的对象，每次调用 next() 都会返回一个结果对象，该结果对象有两个属性，value 表示当前的值，done 表示遍历是否结束。

```jsx
function makeIterator(Array){
    let index = 0;
    return {
        next: function(){
            return (
                Array.length > index ?
                {value: Array[index++]}:
                {done: true}
            )
        }
    }
}

let iterator = makeIterator(['1','2'])
console.log(iterator.next()); // {value: '1'}
console.log(iterator.next()); // {value: '2'}
console.log(iterator.next()); // {done: true}
```

Iterator 的作用：

1. 为各种数据结构，提供一个统一的、简便的访问接口；
2. 使得数据结构的成员能够按某种次序排列；
3. 供 for...of 消费

**默认 Iterator 接口**

ES6 提供了 for of 语句遍历迭代器对象，我们将上述创建的迭代器使用 for of 语句遍历一下：

```jsx
let iterator = makeIterator(['1','2'])
for (let value of iterator) {
    console.log(value);
} // iterator is not iterable
```

结果报错说 iterator is not iterable，这是为什么呢？

ES6 规定默认的 Iterator 接口部署在数据结构的 Symbol.iterator 属性中，如果一个数据结构存在Symbol.iterator属性，则该数据结构可遍历。我们将自定义的 makeIterator 改造如下：

```jsx
const MakeIterator = (Arry) => ({
    [Symbol.iterator](){
        let index = 0;
        return {
            next(){
                let length = Arry.length;
                if(index < length){
                    return {value: Arry[index++]}
                }else{
                    return {done: true}
                }
            }
        }
    }
})
for(let value of MakeIterator([1,2])){
    console.log(value)
}
// 1
// 2
```

**Iterator 的 return()**

我们为 MakeIterator 添加 return 方法，如果for...of循环提前退出（通常是因为出错，或者有break语句），就会调用return()方法，终止遍历。基于这一特性，如果一个对象在完成遍历前，需要清理或释放资源，我们可以部署return()方法，列入文件读取失败时关闭文件。

```jsx
const MakeIterator = (Arry) => ({
    [Symbol.iterator](){
        let index = 0;
        return {
            next(){
                let length = Arry.length;
                if(index < length){
                    return {value: Arry[index++]}
                }else{
                    return {done: true}
                }
            },
            return(){
                return {done: true} 
            }
        }
    }
})
for(let value of new MakeIterator([1, 2, 3])){
    console.log(value) // 1
    方式1
    break;
    // 方式2
    // throw new Error('error');
}
```

**原生具备 Iterator 接口的数据结构**

1. 数组
2. Set
3. Map
4. 类数组对象，如 arguments 对象、DOM NodeList 对象、typedArray 对象

```jsx
// arguments 对象
function sum(){
    for(let value of arguments){
        console.log(value)
    }
}
sum(1,2)
// 1
// 2

// typedArray 对象
let typeArry = new Int8Array(2);
typeArry[0] = 1;
typeArry[1] = 2;
for(let value of typeArry){
    console.log(value) 
}
// 1
// 2
```

5. Generator 对象

```jsx
function* gen(){
    yield 1;
    yield 2;
}
for(let value of gen()){
    console.log(value)
}
```

6. String

为什么 Object 不具有原生 Iterator ？

对象（Object）之所以没有默认部署 Iterator 接口，是因为对象的哪个属性先遍历，哪个属性后遍历是不确定的。本质上，遍历器是一种线性处理，对于任何非线性的数据结构，部署遍历器接口，就等于部署一种线性转换。不过，严格地说，对象部署遍历器接口并不是很必要，因为这时对象实际上被当作 Map 结构使用，ES5 没有 Map 结构，而 ES6 原生提供了。

****调用 Iterator 接口的场合****

解构赋值

```jsx
let set = new Set().add('a').add('b').add('c');

let [x,y] = set; // x='a'; y='b'
```

扩展运算符

```jsx
var str = 'hello';
[...str] //  ['h','e','l','l','o']
```

Q: 扩展运算符等是调用 Iterator 接口，那么 Object 没有部署 Iterator 接口，为什么也能使用 ... 运算符呢？
A: 扩展运算符分为两种
   + 一种是用在函数参数、数组展开的场合，这种情况要求对象是可迭代的（iterable）
   + 另一种是用于对象展开，也就是 {…obj} 形式，这种情况需要对象是可枚举的（enumerable）

```jsx
let obj1 = {
    name: 'qianxun'
} 
let obj2 = {
    age: 3
}
// 数组对象是可枚举的
let obj = {...obj1, ...obj2}
console.log(obj) //{name: 'qianxun', age: 3}

// 普通对象默认是不可迭代的
let obj = [...obj1, ...obj2]
console.log(obj) // object is not iterable
```

****模拟实现 for of****

```jsx
function forOf(obj, cb){
    let iteratorValue = obj[Symbol.iterator]();
    let result = iteratorValue.next()
    while(!result.done){
        cb(result.value)
        result = iteratorValue.next()
    }
}

forOf([1,2,3], (value)=>{
    console.log(value)
})
// 1
// 2
// 3
```

## Generator 基础认识

**认识Generator**

```markdown
// 概念上
Generator 函数是 ES6 提供的一种异步编程解决方案。Generator 函数是一个状态机，封装了多个内部状
态；Generator 函数还是一个遍历器对象生成函数，执行后返回一个遍历器对象。

// 形式上
1.function 关键字与函数名之间有一个星号；
2.函数体内部使用 yield 表达式，定义不同的内部状态。
```

```jsx
function* simpleGenerator(){
    yield 1;
    yield 2;
}
simpleGenerator()
```

如上我们创建了一个简单的 Generator，我们带着两个问题进行探究：

1. Generator 函数运行后会发生什么？
2. 函数中的 ****yield 表达式****有什么作用？

```jsx
function* simpleGenerator(){
    console.log('hello world');
    yield 1;
    yield 2;
}
let generator = simpleGenerator(); // simpleGenerator {<suspended}}
console.log(generator.next())
// hello world
// {value: 1, done: false}
console.log(generator.next())
// {value: 2, done: false}
```

Generator 生成器函数运行后返回一个生成器对象，而普通函数会直接执行函数内部的代码；每次调用生成器对象的 next 方法会执行函数到下一次 yield 关键字停止执行，并且返回一个 {value: Value, done: false} 的对象。

****next 方法的参数****

yield 表达式本身没有返回值，或者说总是返回 undefined。next 方法可以带一个参数，该参数就会被当作上一个 yield 表达式的返回值。通过 next 方法的参数，可以在 Generator 函数运行的不同阶段，从外部向内部注入不同的值，从而调整函数行为。

由于 next 方法的参数表示上一个 yield 表达式的返回值，所以在第一次使用 next 方法时，传递参数是无效的。

```jsx
function sum(x){
    return function(y){
        return x + y;
    }
}
console.log(sum(1)(2))

// 利用next传参改写
function* sum(x){
    let y = yield x;
    while(true){
       y = yield x + y;
    }
}

let gen = sum(2)
console.log(gen.next()) // 2
console.log(gen.next(1)) // 3
console.log(gen.next(2))  // 4
```

****yield 表达式****

yield 表达式的作用：定义内部状态和暂停执行

yield 表达式 与 return 语句的区别

- yield 表达式表示函数暂停执行，下一次再从该位置继续向后执行，而 return语句不具备位置记忆的功能
- 一个函数里，只能执行一个 return 语句，但是可以执行多个 yield表达式
- 任何函数都可以使用 return 语句，yield 表达式只能用在 Generator 函数里面，用在其他地方都会报错
- yield 表达式如果参与运算放在圆括号里面；用作函数参数或放在赋值表达式的右边，可以不加括号

```text
function *gen () {
  console.log('hello' + yield) ×
  console.log('hello' + (yield)) √
  console.log('hello' + yield 1) ×
  console.log('hello' + (yield 1)) √
  foo(yield 1)  √
  const param = yield 2  √
}
```

基于 Generator 生成器函数中可以支持多个 yield，我们可以实现一个函数有多个返回值的场景：

```jsx
function* gen(num1, num2){
    yield num1 + num2;
    yield num1 - num2;
}

let res = gen(2, 1);
console.log(res.next()) // {value: 3, done: false}
console.log(res.next()) // {value: 1, done: false} 
```

**Generator 与 Iterator 之间的关系**

由于 Generator 函数就是遍历器生成函数，因此可以把 Generator 赋值给对象的 Symbol.iterator 属性，从而使得该对象具有 Iterator 接口。Generator 实现方式代码更加简洁。

```jsx
let obj = {
    name: 'qianxun',
    age: 3,
    [Symbol.iterator]: function(){
        let that = this;
        let keys = Object.keys(that)
        let index = 0;
        return {
            next: function(){
                return index < keys.length ?
                {value: that[keys[index++]], done: false}:
                {value: undefined, done: true}
            }
        }
    }
}
for(let value of obj){
    console.log(value)
}
```

Generator:

```jsx
let obj = {
    name: 'qianxun',
    age: 3,
    [Symbol.iterator]: function* (){
        let keys = Object.keys(this)
        for(let i=0; i< keys.length; i++){
            yield this[keys[i]];
        }
    }
}
for(let value of obj){
    console.log(value)
}
```

****Generator.prototype.return()****

> `return()`方法，可以返回给定的值，并且终结遍历 Generator 函数。
>

```jsx
function* gen() {
  yield 1;
  yield 2;
  yield 3;
}

var g = gen();

g.next()        // { value: 1, done: false }
// 如果return()方法调用时，不提供参数，则返回值的value属性为undefined
g.return('foo') // { value: "foo", done: true }
g.next()        // { value: undefined, done: true }
```

如果 Generator 函数内部有`try...finally`代码块，且正在执行`try`代码块，那么`return()`方法会导致立刻进入`finally`代码块，执行完以后，整个函数才会结束。

```jsx
function* numbers () {
  yield 1;
  try {
    yield 2;
    yield 3;
  } finally {
    yield 4;
    yield 5;
  }
  yield 6;
}
var g = numbers();
g.next() // { value: 1, done: false }
g.next() // { value: 2, done: false }
g.return(7) // { value: 4, done: false }
g.next() // { value: 5, done: false }
g.next() // { value: 7, done: true }
```

**yield* 表达式**

> 如果想在 Generator 函数内部，调用另一个 Generator 函数。我们需要在前者的函数体内部，自己手动完成遍历，如果函数调用多层嵌套会导致写法繁琐不易阅读，ES6 提供了 yield* 表达式作为解决方法。
>

委托给其他生成器

```jsx
function* g1() {
    yield 2;
    yield 3;
  }
  
  function* g2() {
    yield 1;
    yield* g1();
    yield 4;
  }
  
  var iterator = g2();
  
  console.log(iterator.next()); // { value: 1, done: false }
  console.log(iterator.next()); // { value: 2, done: false }
  console.log(iterator.next()); // { value: 3, done: false }
  console.log(iterator.next()); // { value: 4, done: false }
  console.log(iterator.next()); // { value: undefined, done: true }
```

委托给其他可迭代对象

```jsx
function* gen(){
    yield* [1,2,3]
}
console.log(gen().next()) // {value: 1, done: false}
```

yield* 表达式

```jsx
function* g4() {
    yield* [1, 2];
    return "foo";
}  
function* g5() {
    let result = yield* g4();
    console.log(result) // 'foo'
}

var iterator = g5();

console.log(iterator.next()); // { value: 1, done: false }
console.log(iterator.next()); // { value: 2, done: false }
console.log(iterator.next()); // { value: undefined, done: true }
```

****Generator 函数的 this****

Generator 函数返回一个遍历器，ES6 规定这个遍历器是 Generator 函数的实例，继承了 Generator.prototype 对象上的方法，但无法获取 this 上的属性，因为这时 this 是全局对象，而不是实例对象。

```jsx
function* gen(){

}
gen.prototype.say = function(){
    console.log('hi')
}   
let obj = gen()
console.log(obj instanceof gen) // true
obj.say() // hi
```

如果想像构造函数一样访问实例属性，可以修改 this 绑定到 Generator.prototype 上。

```jsx
function* gen(){
    this.a = 1
}
gen.prototype.say = function(){
    console.log('hi')
}   
let obj = gen.call(gen.prototype)
console.log(obj instanceof gen) // true
obj.say() // hi
obj.next()
console.log(obj.a) //1
```

**Babel 编译后的 Generator**

先来一个简单的🌰

```jsx
function* gen(){
    yield 1;
}
let g1 = gen();
console.log(g1.next());
```

我们用 babel 将上面代码编译的 polyfill 如下：

```jsx
"use strict";

var _marked = /*#__PURE__*/regeneratorRuntime.mark(gen);

function gen() {
  return regeneratorRuntime.wrap(function gen$(_context) {
    while (1) {
      switch (_context.prev = _context.next) {
        case 0:
          _context.next = 2;
          return 1;

        case 2:
        case "end":
          return _context.stop();
      }
    }
  }, _marked);
}

var g1 = gen();
console.log(g1.next());
```

我们可以看到编译后使用了 regeneratorRuntime 上的 mark 和 wrap 方法，但 babel 后的内容并看不出这两个方法具体做了什么，我们可以使用 facebook 提供的 regenerator 编译后查看完整的代码，使用如下命令：

```jsx
regenerator --include-runtime babelGenerator.js > babelGenerator-es5.js
```

mark 函数根据 ES6 规范构建了 Generator 的关系链，让返回的对象是 Generator 的实例。本文重点还是聚焦在 Generator 如何实现 Iterator 上，主要介绍 mark 函数的执行流程。

下文中会提到状态机概念，我们简单看一下用 Generator 如何实现一个状态机：

```jsx
function* StateMachine(state){
    let transition;
    while(true){
        if(transition === "INCREMENT"){
            state++;
        }else if(transition === "DECREMENT"){
            state--;
        }
        transition = yield state;
    }
}
const iterator = StateMachine(0);
console.log(iterator.next()); // 0
console.log(iterator.next('INCREMENT')); // 1
console.log(iterator.next('DECREMENT')); // 0
```

_context 数据结构：

```jsx
const _context = {
	prev: 0, // 表示本次生成器函数执行时的指针位置
  next: 0, // 表示下次生成器函数调用时的指针位置
  sent: '', // 表示调用 g.next(params) 时,传入的 params 参数,作为上一次 yield 返回值
  done: false, // 是否完成
	method: 'next', // 方法
  abrupt: function(type, arg) { // return
    var record = {};
    record.type = type;
    record.arg = arg;

    return this.complete(record);
  },
  complete: function(record, afterLoc) {
    if (record.type === "return") {
      this.rval = this.arg = record.arg;
      this.method = "return";
      this.next = "end";
    }

    return ContinueSentinel;
  },
  stop: function() { // 完成函数
    this.done = true;
    return this.rval;
  }
```

还是学习 Generator 开始的那个问题，执行一个 Genetator 后返回了什么？我们执行编译后的代码查看 Genetator 结构如下。咦？_invoke() 这个函数是什么？

![image](https://user-images.githubusercontent.com/51777605/166630557-f4a1c043-319c-4c60-9457-680b1d8b17f1.png)

我们执行 g1.next()：

```jsx
function makeInvokeMethod(innerFn, self, context) {
    return function invoke(method, arg) {
      context.method = method;
      context.arg = arg;

      while (true) {
        if (context.method === "next") {
          // Setting context._sent for legacy support of Babel's
          // function.sent implementation.
          context.sent = context._sent = context.arg;
        } 
        var record = tryCatch(innerFn, self, context);
        if (record.type === "normal") {
          // If an exception is thrown from innerFn, we leave state ===
          // GenStateExecuting and loop back for another invocation.
          state = context.done
            ? GenStateCompleted
            : GenStateSuspendedYield;

          if (record.arg === ContinueSentinel) {
            continue;
          }

          return {
            value: record.arg,
            done: context.done
          };
        } 
      }
    };
  }
```

上面代码省略了其他的逻辑判断，我们可以看到我们判断是 next 方法后会先进行参数处理，接下来执行 tryCatch(innerFn, self, context)。

```jsx
function tryCatch(fn, obj, arg) {
    try {
      return { type: "normal", arg: fn.call(obj, arg) };
    } catch (err) {
      return { type: "throw", arg: err };
    }
  }
```

tryCatch 方法先执行 fn方法后返回了一个对象，这里的 fn 就是我们传给 wrap 函数的状态机，因为_context.next 为 0，执行 case 0 语句。梳理下来 tryCatch 执行后返回如下的对象，继续执行 _invoke 函数后面的逻辑。
```jsx
{type: 'normal', arg: 1}
```
