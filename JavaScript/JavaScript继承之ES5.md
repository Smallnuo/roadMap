## 原型与原型链

![image](https://user-images.githubusercontent.com/51777605/165920650-8d059672-8bf8-4099-9ed2-96b8429e4e51.png)

## **ECMAScript5继承**

### 原型链继承

思路：修改子类原型为父类

示例：

```js
function Father(){
  this.color = ['red']
}
function Son(){
}
Son.prototype = new Father()
let son1 = new Son()
```

原型链关系图：
![image](https://user-images.githubusercontent.com/51777605/165920960-ec166a20-fbbc-4e7b-b06e-9744dd30b67e.png)

特点：

1. 在创建实例时，不能向父类的构造函数传参
2. 引用类型的属性被所有实例共享

```js
function Father(){
  this.color = ['red']
}
function Son(){

}
Son.prototype = new Father()
console.log(Son.constructor) // [Function: Father]

let son1 = new Son()
son1.color.push('blue')
console.log(son1.color) // [ 'red', 'blue' ]

let son2 = new Son()
console.log(son2.color) // [ 'red', 'blue' ]
```

### 盗用构造函数

思路：在子类构造函数中调用父类构造函数，父类执行上下文此时为子类

特点：

1. 避免了引用类型的属性被所有实例共享

```js
function Father(){
  this.color = ['red']
}
function Son(){
  Father.call(this)
}
let son1 = new Son()
son1.color.push('black')
console.log(son1.color) // [ 'red', 'black' ]

let son2 = new Son()
console.log(son2.color) // [ 'red' ]
```

1. 可以在 Child 中向 Parent 传参

```js
function Father(name){
    this.name = name
    this.color = ['red']
    this.say = function() {
        console.log(this.name)
    }
}
function Son(name){
    Father.call(this, name)
}

let son1 = new Son('qianxun')
son1.say() // qianxun
```

1. 方法都在构造函数中定义，每次创建实例都会创建一遍方法。子类也不能访问父类原型上定义的方法

```js
function Father(name){
    this.name = name
    this.color = ['red']
}
Father.prototype.say = function(){
    console.log('hello')
}
function Son(name){
    Father.call(this, name)
}

let son1 = new Son('qianxun')
son1.say() // son1.say is not a function
```

### 组合继承

思路：原型链继承结合盗用构造函数

示例：

```js
function Father(myName){
    this.name = myName;
    this.color = ['red']
}
Father.prototype.sayName = function(){
    console.log(this.name)
}
function Son(name){
    Father.call(this, name)
}

Son.prototype = new Father()
Son.prototype.constructor = Son

let son1 = new Son('qianxun')
son1.color.push('blue')
console.log(son1.color) // [ 'red', 'blue' ]
son1.sayName() // qianxun

let son2 = new Son('chengtuo')
console.log(son2.color) // [ 'red' ]
son2.sayName() // qianxun

```

特点：

1. 融合原型链继承和构造函数的优点
2. 父类构造函数会被调用两次

```js
function Father(myName){
    this.name = myName;
    this.color = ['red']
}
Father.prototype.sayName = function(){
    console.log(this.name)
}
function Son(name){
    // 第二次调用Father()
    Father.call(this, name)
}
// 第一次调用Father()
Son.prototype = new Father()
Son.prototype.constructor = Son

let son1 = new Son('qianxun')
console.log(son1)
console.log(Son.prototype)
```

### 原型式继承

思路：利用Object.create()创建对象

```js
function object(obj){
    function F(){}
    F.prototype = obj
    return new F()
 }
```

示例：

```js
function object (obj) {
     function F(){}
     F.prototype = obj
     return new F()
 }

 let person = {
     name: 'qianxun',
     friends: ['mumiao']
 }
let person1 = new object(person)
let person2 = new object(person)

person1.name = 'person1'; // 在person1实例对象上新增name属性，并不是修改原型上的name
console.log(person2.name); // qianxun

person1.friends.push('shuangxu');
console.log(person2.friends); // [ 'mumiao', 'shuangxu' ]

console.log(person1.__proto__ === person2.__proto__) // true
```

原型链图：

![image](https://user-images.githubusercontent.com/51777605/166628049-074c1772-2f8c-4629-9576-f714c3ba8d4f.png)


特点：

1. 引用类型的属性被所有实例共享

### 寄生式继承

思路：封装继承过程的函数，该函数在内部以某种形式来做增强对象，最后返回对象。

示例：

```js
function Create(obj){
    const clone = Object.create(obj)
    clone.say = function () {
        console.log('hello world')
    }
    return clone
}
let res = Create({})
res.say() // hello world
```

### 寄生式组合继承(最优解)

示例：

```js
function Father (name){
    this.name = name
}
Father.prototype.sayName = function () {
    console.log(this.name)
}
function Son (name, age) {
    Father.call(this, name)
    this.age = age
}
// Son.prototype = new Father()
// Son.prototype.constructor = Son
function inherit (Father, Son) {
    const prototype = Object.create(Father.prototype)
    prototype.constructor = Son
    Son.prototype = prototype
}
inherit(Father, Son)
// Son原型上新增的属性和方法必须放在修改关系之后添加
Son.prototype.sayAge = function () {
    console.log(this.age)
}

let son1 = new Son('qianxun', 18)
son1.sayName() // qianxun
son1.sayAge() // 18
```

原型链图：
![image](https://user-images.githubusercontent.com/51777605/165921710-f751b76d-c308-43de-9a04-80cbf9d88ea5.png)
