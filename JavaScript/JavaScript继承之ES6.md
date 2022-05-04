# JavaScript 继承之ES6

![image](https://user-images.githubusercontent.com/51777605/165922129-5ce3d35b-ffb2-4510-af4e-7ca6d93a80c5.png)

## Class 基础语法

### constructor

constructor() 方法是类的默认方法，默认返回 new 生成的实例对象，也可以返回指定对象。

```js
class Foo {
  constructor() {
    return {};
  }
}

console.log(new Foo() instanceof Foo) // false
```

### 类的实例

1. 实例的方法需要显式定义在 this 上
2. 所有实例共享一个原型

```js
class Point {
  x = 3;
  constructor(x, y) {
    this.y = y;
    this.say = function(){
        return 'say'
    }
  }
  toString() {
    return '(' + this.x + ', ' + this.y + ')';
  }
}
let p1 = new Point(2, 3);
console.log(p1.hasOwnProperty('x')) // true
console.log(p1.hasOwnProperty('y')) // true
console.log(p1.hasOwnProperty('say')) // true
console.log(p1.hasOwnProperty('toString')) // false
console.log(p1.__proto__.hasOwnProperty('toString')) // true

let p2 = new Point(1,2);
console.log(p1.__proto__ === p2.__proto__) // true
```

### getter/setter

```js
class Person{
  state = {
    name: 'qianxun'
  }
  get name(){
    return this.state.name
  }
  set name(newName){
    this.state.name = newName
    return this.state.name
  }
}
let person = new Person();
console.log(person.name) // qianxun
console.log(person.name = 'mumiao') // mumiao
```

### 属性表达式

```js
let methodName = 'getArea';
class Square {
  [methodName]() {
    console.log('qianxun')
  }
}
let square = new Square();
square.getArea() // qianxun
```

### 静态方法与静态属性(static)

静态方法不会被实例继承，静态方法中的 this 是类而不是实例；静态方法可以被子类继承。

```js
class Foo {
  static classMethod() {
    console.log('hello');
  }
}
class FooSon extends Foo {}
Foo.classMethod() // 'hello'
let foo = new Foo();
let fooSon = new FooSon();
FooSon.classMethod() // 'hello'
fooSon.classMethod() // foo.classMethod is not a function
foo.classMethod() // foo.classMethod is not a function
```

静态属性同理

```js
class Foo {
  static bar = 'bar'
}
class FooSon extends Foo{}
let foo = new Foo();
let fooSon = new FooSon(); 
console.log(Foo.bar) // bar
console.log(FooSon.bar) // bar
console.log(foo.bar) // undefined
console.log(fooSon.bar) // undefined
```

### 私有方法与私有属性

引入一个新的前缀 # 表示私有属性，而没有采用 private 关键字，是因为 JavaScript 是一门动态语言，没有类型声明，使用独立的符号似乎是唯一的比较方便可靠的方法，能够准确地区分一种属性是否为私有属性。另外，Ruby 语言使用 @ 表示私有属性，ES6 没有用这个符号而使用 # ，是因为 @ 已经被留给了 Decorator。

私有属性只能在类内部使用，外部使用会报错。

```js
class Foo {
  #bar = 'private bar'
  bar = 'public bar'
  #name(){
    console.log('private function' + this.#bar)
  }
  name(){
    console.log('public function ' + this.#bar)
  }
}
let foo = new Foo();
// console.log(foo.#bar) // 报错
console.log(foo.bar) // public bar
foo.name() // public function private bar
// foo.#name // 报错
```

### new.target

new.target 属性用于检测函数或构造方法是否是通过 new 调用，通过 new 实例化中 new.target 返回指向构造函数的引用，普通函数调用中 new.target 返回 undefined。需要注意的是，子类继承父类时，new.target 返回子类。

```jsx
class Dog {
  constructor() {
    console.log(new.target === Dog); // false
    console.log(new.target === BlackDog); // true
    if(new.target === Dog){
      throw new Error('本类不能实例化');
    }
  }
}

class BlackDog extends Dog {
  constructor() {
    super();
  }
}
// var dog = new Dog(); // 本类不能实例化
var blackDog = new BlackDog();
```

### 其他

1. 类的内部所有定义的方法是不可枚举的，函数是可以的

> Object.keys(obj) 返回给定对象的自身可枚举属性组成的数组
>

> Object.getOwnPropertyNames(obj) 返回指定对象的所有自身属性的属性名（包括不可枚举属性但不包括Symbol值作为名称的属性）组成的数组
>

```js
class Point {
  constructor() {}
  toString() {}
}
console.log(Object.keys(Point.prototype))// []
console.log(Object.getOwnPropertyNames(Point.prototype))// ["constructor","toString"]

function Person(){}
Person.prototype.constructor= function(){}
Person.prototype.toString = function(){}
console.log(Object.keys(Person.prototype))// ['toString']
console.log(Object.getOwnPropertyNames(Person.prototype))// ["constructor","toString"]
```

2. 类必须使用 new 调用，构造函数不用

```js
class Logger{}
let p = Logger() // Class constructor Logger cannot be invoked without 'new'
```

3. this 默认指向类的实例，需要注意 this 丢失的情况

```js
class Logger {
    printName(name = 'there') {
      console.log(this) // undefined, 默认严格模式
      this.print(`Hello ${name}`);
    }
    print(text) {
      console.log(text);
    }
}
const logger = new Logger();
const { printName } = logger;
printName();
```

## 继承

### extends

Class 通过 extends 关键字实现继承，这种写法更加清晰明了。extends 后面可以跟任意函数或null

```js
class Point {}
class ColorPoint extends Point {}
class ColorPoint extends null {}

// extends Point
console.log(ColorPoint.__proto__ === Point); // true
// extends null
console.log(ColorPoint.__proto__ === Function.prototype); // true
```

关于 extends 后面为 null 这种情况笔者找到一个不错的回答，希望可以给予你帮助：

[ES6 class extends null 的语法设计是不是毫无意义？](https://www.zhihu.com/question/282229406)

## super 关键字

```js
super([arguments]);
// 调用 父对象/父类 的构造函数

super.functionOnParent([arguments]);
// 调用 父对象/父类 上的方法
```

```js
class Point {
  constructor(x, y){
    this.x = x;
    this.y = y;
  }
  toString(){
    return '(' + this.x + ',' + this.y + ')';
  }
}

class ColorPoint extends Point {
  constructor(x, y, color) {
    super(x, y); // 调用父类的constructor(x, y)
    this.color = color;
  }
  toString() {
    return this.color + ' ' + super.toString(); // 调用父类的toString()
  }
}
let color = new ColorPoint(2, 3, 'green');
console.log(color.toString()) // green (2,3)
```

为什么子类的构造函数，一定要调用 super()？原因就在于 ES6 的继承机制，与 ES5 完全不同。ES5 的继承机制，是先创造一个独立的子类的实例对象，然后再将父类的方法添加到这个对象上面，即“实例在前，继承在后”。ES6 的继承机制，则是先将父类的属性和方法，加到一个空的对象上面，然后再将该对象作为子类的实例，即“继承在前，实例在后”。这就是为什么 ES6 的继承必须先调用super() 方法，因为这一步会生成一个继承父类的 this 对象，没有这一步就无法继承父类。

### Object.getPrototypeOf(obj)

> 返回指定对象的原型，通常我们使用该方法判断一个类是否继承另一个类
>

```js
class Point {}
class ColorPoint extends Point {}
console.log(Object.getPrototypeOf(ColorPoint) === Point) // true
```

### prototype 与 __proto__ 属性

因为 Class 作为构造函数的语法糖，同时有 prototype 属性和 __proto__ 属性，因此同时存在两条继承链。

（1）子类的 __proto__ 属性，表示构造函数的继承，总是指向父类。

（2）子类 prototype 属性的 __proto__ 属性，表示方法的继承，总是指向父类的 prototype 属性。

```js
class Parent {}
class Child extends Parent {}
console.log(Child.__proto__ === Parent); // true
console.log(Child.prototype.__proto__ === Parent.prototype); // true
```

ES6 的原型链示意图为：

![image](https://user-images.githubusercontent.com/51777605/165923532-1254d5f0-a50e-45a7-bf06-f14ab1b434c3.png)

我们对比 ES5 寄生组合继承看一下他们之间的不同：

![image](https://user-images.githubusercontent.com/51777605/165923670-65b70c16-3f35-4580-935c-0c8d8f9b414b.png)

## Babel 解析转 ES5

Class 基础语法转换

![image](https://user-images.githubusercontent.com/51777605/165923718-a6a599dc-a11f-4c43-a75f-329c67832337.png)

- _classCallCheck:检查 Class 是否通过 new 方式调用

```js
function _classCallCheck(instance, Constructor) {
    if (!(instance instanceof Constructor)) {
        throw new TypeError("Cannot call a class as a function");
    }
}
```

- _defineProperty

```jsx
function _defineProperty(obj, key, value) { 
  if (key in obj) { 
    Object.defineProperty(obj, key, { 
      value: value, 
      enumerable: true, 
      configurable: true, 
      writable: true }); 
  } else { 
    obj[key] = value; 
  } 
    return obj; 
}
```

- _classPrivateFieldInitSpec 定义私有属性

```jsx
var _age = /*#__PURE__*/new WeakMap();
_classPrivateFieldInitSpec(this, _age, {
  writable: true,
  value: 3
});

function _classPrivateFieldInitSpec(obj, privateMap, value) { 
  _checkPrivateRedeclaration(obj, privateMap); 
  privateMap.set(obj, value); 
}
function _checkPrivateRedeclaration(obj, privateCollection) { 
  if (privateCollection.has(obj)) { 
    throw new TypeError("Cannot initialize the same private elements twice on an object"); 
  } 
}
```

- Babel 生成了一个 _createClass 辅助函数，该函数传入三个参数，第一个是构造函数，在这个例子中也就是 Person，第二个是要添加到原型上的函数数组，第三个是要添加到构造函数本身的函数数组，也就是所有添加 static 关键字的函数。该函数的作用就是将函数数组中的方法添加到构造函数或者构造函数的原型中，最后返回这个构造函数。

    ```jsx
    var _createClass = function() {
        function defineProperties(target, props) {
            for (var i = 0; i < props.length; i++) {
                var descriptor = props[i];
                // 防止 Object.keys() 之类的方法遍历到
                descriptor.enumerable = descriptor.enumerable || false;
                descriptor.configurable = true;
                // value 与 setter/getter 互斥，判断是否是getter/setter
                if ("value" in descriptor) descriptor.writable = true;
                Object.defineProperty(target, descriptor.key, descriptor);
            }
        }
        return function(Constructor, protoProps, staticProps) {
            // 添加到原型上的数组
            if (protoProps) defineProperties(Constructor.prototype, protoProps);
            // 添加到构造函数本身的数组（静态方法）
            if (staticProps) defineProperties(Constructor, staticProps);
            // 返回构造函数
            return Constructor;
        };
    }();
    ```

  ### 继承

![image](https://user-images.githubusercontent.com/51777605/165923799-a5125a0e-ea61-48aa-bafa-ecb005862915.png)

    babel 编译后代码
    
    ```jsx
    'use strict';
    
    function _possibleConstructorReturn(self, call) {
        if (!self) {
            throw new ReferenceError("this hasn't been initialised - super() hasn't been called");
        }
        return call && (typeof call === "object" || typeof call === "function") ? call : self;
    }
    
    function _inherits(subClass, superClass) {
        if (typeof superClass !== "function" && superClass !== null) {
            throw new TypeError("Super expression must either be null or a function, not " + typeof superClass);
        }
        subClass.prototype = Object.create(superClass && superClass.prototype, { constructor: { value: subClass, enumerable: false, writable: true, configurable: true } });
        if (superClass) Object.setPrototypeOf ? Object.setPrototypeOf(subClass, superClass) : subClass.__proto__ = superClass;
    }
    
    function _classCallCheck(instance, Constructor) {
        if (!(instance instanceof Constructor)) {
            throw new TypeError("Cannot call a class as a function");
        }
    }
    
    var Parent = function Parent(name) {
        _classCallCheck(this, Parent);
    
        this.name = name;
    };
    
    var Child = function(_Parent) {
        _inherits(Child, _Parent);
    
        function Child(name, age) {
            _classCallCheck(this, Child);
    
            // 调用父类的 constructor(name)
            var _this = _possibleConstructorReturn(this, (Child.__proto__ || Object.getPrototypeOf(Child)).call(this, name));
    
            _this.age = age;
            return _this;
        }
    
        return Child;
    }(Parent);
    
    var child1 = new Child('kevin', '18');
    
    console.log(child1);
    ```
    
    ****_inherits****
    
    ```jsx
    function _inherits(subClass, superClass) {
        // extend 的继承目标必须是函数或者是 null
        if (typeof superClass !== "function" && superClass !== null) {
            throw new TypeError("Super expression must either be null or a function, not " + typeof superClass);
        }
    
        // Object.create，设置子类 prototype 属性的 __proto__ 属性指向父类的 prototype 属性
        subClass.prototype = Object.create(superClass && superClass.prototype, { constructor: { value: subClass, enumerable: false, writable: true, configurable: true } });
    
        // 设置子类的 __proto__ 属性指向父类
        if (superClass) Object.setPrototypeOf ? Object.setPrototypeOf(subClass, superClass) : subClass.__proto__ = superClass;
    }
    ```
    
    ****_possibleConstructorReturn****
    
    ```jsx
    var _this = _possibleConstructorReturn(this, (Child.__proto__ || Object.getPrototypeOf(Child)).call(this, name));
    ```
    
    简化后为：
    
    ```jsx
    var _this = _possibleConstructorReturn(this, Parent.call(this, name));
    ```
    
    _possibleConstructorReturn 源码为:
    
    ```jsx
    function _possibleConstructorReturn(self, call) {
        if (!self) {
            throw new ReferenceError("this hasn't been initialised - super() hasn't been called");
        }
    		// 判断父类 constructor 返回值
    		// object/function 返回父类 constructor 返回值
    		// null、undefined 等返回子类
        return (call && (typeof call === "object" || typeof call === "function")) ? call : self;
    }
    ```
