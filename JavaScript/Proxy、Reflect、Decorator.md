# 认识 Proxy

场景：HTML 中有个 span 标签和 button 标签，当点击 button 时 span 标签里的值加一，我们发现在这个场景中我们需要监听 button 的点击，并将计算后的值重新渲染在 span 标签中。

传统做法：

```jsx
let button = document.querySelector('#button')
let container = document.querySelector('#container')
button.addEventListener('click', addOption)
function addOption(){
    container.innerHTML = Number(container.innerHTML) + 1
}
```

Object.defineProperty：

```jsx
let button = document.querySelector('#button')
let container = document.querySelector('#container')
let obj = {}, value = 1;
Object.defineProperty(obj, 'value', {
    get: ()=>{
        return value;
    },
    set: (newValue)=>{
        value = newValue;
        container.innerHTML = newValue;
    }
})
button.addEventListener('click', addOption)
function addOption(){
    obj.value += 1;
}
```

咦？Object.defineProperty 这种方式代码量增多了，这样写的优势是什么呢？笔者认为有两个优势：

1. 我们通过Object.defineProperty 自动监听了我们数据的变化，当数据变化时会自动触发 set
2. 将渲染数据独立出来，通过 obj.value 存储渲染的值，如果需要改变 span 中的值直接改变 obj.value 就可以

Object.defineProperty 的缺陷：

1. 需要重新申明一个value存储值，如果你直接 obj.value = newValue 会陷入死循环，栈溢出
2. 监控多个属性变化，需要写多遍如下的代码
3. 只能监听 get、set 行为，无法监听更多的行为

Proxy:

> let proxy = new Proxy(target, handler);
>

target: 拦截的目标对象

handler:  一个对象，用来定制拦截行为

```jsx
let button = document.querySelector('#button')
    let container = document.querySelector('#container')
    let obj = { value: 1 }
    let proxy = new Proxy(obj, {
        get: (target, key)=>{
            return target[key]
        },
        set: (target, key, value)=>{
            target[key] = value;
            container.innerHTML = value;
        }
    })
    button.addEventListener('click', addOption)
    function addOption(){
        proxy.value += 1;
    }
```

1. Object.defineProperty 监听属性，Proxy 监听整个代理对象，并且 Proxy 允许我们监听更多的行为
2. 使用 Object.defineProperty，我们修改原来的 obj 对象就可以触发拦截，而使用 proxy 必须修改代理对象(Proxy 实例)才可以触发拦截

**取消代理**

```jsx
// 创建一个可撤销的代理对象，返回一个包含了代理对象本身和它的撤销方法的可撤销 Proxy 对象
Proxy.revocable(target, handler);
```

```jsx
const revocable = Proxy.revocable({}, {
    get(target, name) {
        return "[[" + name + "]]";
    }
});
const proxy = revocable.proxy;
console.log(proxy.foo);  // "[[foo]]"

revocable.revoke();

console.log(proxy.foo); // 抛出 TypeError
```

**this 问题**

Proxy 代理时，目标对象内部的 this 指向 Proxy 实例。

```jsx
const target = {
    m: function () {
        console.log(this === proxy);
    }
};
const handler = {};

const proxy = new Proxy(target, handler);

target.m() // false
proxy.m()  // true
```

此时 this 为 proxy 代理对象，没有 getDate() 方法

```jsx
const target = new Date();
const handler = {};
const proxy = new Proxy(target, handler);

proxy.getDate();
// TypeError: this is not a Date object.
```

我们可以通过代理将 getDate 的 this 修改为实例 target

```jsx
const target = new Date();
const handler = {
    get(target, prop) {
        if (prop === 'getDate') {
            return target.getDate.bind(target);
        }
        return Reflect.get(target, prop);
    }
};
const proxy = new Proxy(target, handler);

proxy.getDate()
```

## Proxy 实例方法

```jsx
target 目标对象
property 被获取的属性名
receiver Proxy 或者继承 Proxy 的对象
value 新属性值
argumentsList constructor 的参数列表
prototype 对象新原型或为 null
```

receiver 参数情况1

```jsx
const proxy = new Proxy({}, {
    // get陷阱中target表示原对象 key表示访问的属性名
    get(target, key, receiver) {
        console.log(receiver === proxy); // true
        return target[key];
    },
});

console.log(proxy.name);
```

receiver 参数情况2:

```jsx
let proxy = new Proxy({}, {
    get(target, prop, recevier){
        console.log(recevier === obj)
    }
})
let obj = Object.create(proxy)
```

场景1:

const 定义的对象是可以被修改属性值的，如何让 const 定义的对象属性值不可被修改？

```jsx
const term = {
    id: 1,
    value: 'hello',
    properties: [{ type: 'usage', value: 'greeting' }],
};

const immutable = obj =>
    new Proxy(obj, {
        get(target, prop) {
            return typeof target[prop] === 'object'
                ? immutable(target[prop])
                : target[prop];
        },
        set() {
            throw new Error('This object is immutable.');
        },
    });

const immutableTerm = immutable(term);
const immutableProperty = immutableTerm.properties[0];

immutableTerm.value = 'hi';            // Error: This object is immutable.
```

场景2:

单例模式：一个类只能生成一个实例

```jsx
const singletonify = (className) => {
    return new Proxy(className.prototype.constructor, {
        instance: null,
        construct: (target, argumentsList) => {
            if (!this.instance)
                this.instance = new target(...argumentsList);
            return this.instance;
        }
    });
}

class MyClass {
    constructor(msg) {
        this.msg = msg;
    }
    printMsg() {
        console.log(this.msg);
    }
}

MySingletonClass = singletonify(MyClass);

const myObj = new MySingletonClass('first');
myObj.printMsg();           // 'first'
const myObj2 = new MySingletonClass('second');
myObj2.printMsg();           // 'first'
```

场景3: 观察者模式

```jsx
const queuedObservers = new Set();

const observe = fn => queuedObservers.add(fn);
const observable = obj => new Proxy(obj, {set});

function set(target, key, value, receiver) {
    const result = Reflect.set(target, key, value, receiver);
    queuedObservers.forEach(observer => observer());
    return result;
}

const person = observable({
    name: 'qianxun',
    age: 20
});

function print() {
    console.log(`${person.name}, ${person.age}`)
}
observe(print);

person.name = 'mumiao'; // mumiao, 20
```

## 设计 Reflect 的目的

1. 将 Object 对象内部的方法放在 Reflect 对象上
2. 修改 Object 某些内部方法的返回结果，可以通过结果判断操作是否成功。

```jsx
// 老写法
try {
  Object.defineProperty(target, property, attributes);
} catch (e) {
}

// 新写法
if (Reflect.defineProperty(target, property, attributes)) {
} else {
}
```

1. 将 Object 操作行为都改为函数行为。

```jsx
// 老写法
'key' in Object
delete obj[key]

// 新写法
Reflect.has(Object, 'key')
Reflect.deleteProperty(obj, key)
```

1. Reflect 对象的方法与 Proxy 对象的方法一一对应，在 Proxy 中我们可以结合 Reflect 使用，下图展示了 Reflect 对应之前的一些操作，你可以用 Reflect 的写法进行替换

Proxy 结合 Reflect 结合实现单例模式

```jsx
function Person(name){
    this.name = name
}

const proxy = new Proxy(Person, {
    instance: null,
    construct(target, argArray, newTarget) {
        if(!this.instance){
            this.instance = Reflect.construct(target, [...argArray])
        }
        return this.instance
    }
})

const p = new proxy('1')
console.log(p.name); // 1
const p2 = new proxy('111')
console.log(p2.name); // 1
```

Reflect.apply

```jsx
// 老写法
Function.prototype.apply.call(Math.floor, undefined, [1.75]);
// 新写法
Reflect.apply(Math.floor, undefined, [1.75])
```
![image](https://user-images.githubusercontent.com/51777605/167292404-a0495ea4-b68e-474e-9509-786e1a0cd267.png)

1. 为什么推荐 Reflect 与 Proxy 一起使用？

我们有如下例子，我们本意是获取 obj 中的 mumiao，却发现拿到的是 qianxun，为什么会这样呢？

执行obj.value 时由于该对象没有 value 属性，会往他的原型 proxy 找，proxy 是 parent 对象的一个代理，所以我们会执行 handler 中的 get 操作，在这个操作中我们 Reflect.get(target, key); 相当于 target[key]，由于此时  target 是 parent，所以获取的是 parent 对象上的 name。

```jsx
const parent = {
    name: 'qianxun',
    get value() {
        return this.name;
    },
};
const handler = {
    get(target, key, receiver) {
        return Reflect.get(target, key);
    },
};
const proxy = new Proxy(parent, handler);
const obj = {
    name: 'mumiao',
};
// 设置obj继承与parent的代理对象proxy
Object.setPrototypeOf(obj, proxy);
console.log(obj.value); // qianxun
```

我们可以使用 `Reflect.get(target, key, receiver)` , `receiver` 是 obj，可以通过第三个参数修正我们的上下文。

```jsx
const parent = {
    name: 'qianxun',
    get value() {
        return this.name;
    },
};
const handler = {
    get(target, key, receiver) {
        return Reflect.get(target, key, receiver);
    },
};
const proxy = new Proxy(parent, handler);
const obj = {
    name: 'mumiao',
};
// 设置obj继承与parent的代理对象proxy
Object.setPrototypeOf(obj, proxy);
console.log(obj.value); // mumiao
```

## Decorator

Decorator 是 ES7 中的一个提案，用来为类、类的属性、类的方法提供额外的功能。Decorator 使用了设计模式中的装饰者模式。装饰者模式就是在不改变对象自身的基础上，为对象动态的增添一些功能。比如人穿衣服，天热少穿一些，天冷多穿一些，并没有改变人本身，针对不同的天气情况动态添、减衣服。

写法：@ + 函数名，放在类、类方法、类属性定义前面

## 类的装饰

```jsx
@testable
class MyTestableClass {
}

function testable(target) {
    // 添加静态属性
    target.isTestable = true;
}

let p = new MyTestableClass()
```

## 类的属性和方法

```jsx
class MyClass {
  @readonly
  method() { }
	@readonly name = 'qianxun'
}

function readonly(target, name, descriptor) {
  descriptor.writable = false;
  return descriptor;
}
```

需要注意的是，我们装饰方法时可以通过 descriptor.value 拿到修饰的方法，但是装饰属性时 descriptor.value 就为null，如果你想拿到对应的值可以使用 descriptor.initializer && descriptor.initializer()。

```jsx
class MyClass {
    @readonly name = 'qianxun'
}
function readonly(target, name, descriptor) {
    console.log(descriptor.value) // undefined
    console.log(descriptor.initializer && descriptor.initializer()) // qianxun
    descriptor.writable = true;
    return descriptor;
}
```

## 装饰器不能用于函数

```jsx
const readOnly = require("some-decorator");

@readOnly
function foo() {
}
```

由于函数会进行申明提升，提升后代码如下：

```jsx

@readOnly
function foo() {
}

const readOnly = require("some-decorator");
```

## 应用场景

1. log 函数

```jsx
class Math {
    @log
    add(a, b) {
        return a + b;
    }
}
function log(target, name, descriptor) {
    const originalMethod = descriptor.value;
    descriptor.value = function() {
        console.log(`Calling "${name}" with`, arguments);
        return originalMethod.apply(null, arguments);
    };
    return descriptor;
}
const math = new Math();
math.add(2, 4); // Calling "add" with [Arguments] { '0': 2, '1': 4 }
```

1. 统计方法执行时间

```jsx
function time(prefix) {
    let count = 0;
    return function handleDescriptor(target, key, descriptor) {
        const fn = descriptor.value;
        if (prefix == null) {
            prefix = `${target.constructor.name}.${key}`;
        }
        if (typeof fn !== 'function') {
            throw new SyntaxError(`@time can only be used on functions, not: ${fn}`);
        }
        return {
            ...descriptor,
            value() {
                const label = `${prefix}-${count}`;
                count++;
                console.time(label);

                try {
                    return fn.apply(this, arguments);
                } finally {
                    console.timeEnd(label);
                }
            }
        }
    }
}

class Calculate{
    @time()
    sum(){
        let sum = 0;
        for(let i=0; i<100; i++){
            sum += i;
        }
    }
}
Calculate.prototype.sum() // Calculate.sum-0: 0.055ms
```

1. Mixin

概念：在一个对象中混入另外一个对象的方法

```jsx
function mixin(...mixins) {
    return target => {
        if (!mixins.length) {
            throw new SyntaxError(`@mixin() class ${target.name} requires at least one mixin as an argument`);
        }

        for (let i = 0, l = mixins.length; i < l; i++) {
            const descs = Object.getOwnPropertyDescriptors(mixins[i]);
            const keys = Object.getOwnPropertyNames(descs);

            for (let j = 0, k = keys.length; j < k; j++) {
                const key = keys[j];

                if (!target.prototype.hasOwnProperty(key)) {
                    Object.defineProperty(target.prototype, key, descs[key]);
                }
            }
        }
    };
}

const SingerMixin = {
    sing(sound) {
        console.log(sound);
    },
    say(){
        console.log('hello world');
    }
};

@mixin(SingerMixin)
class Bird {
    singMatingCall() {
        this.sing('tweet tweet');
        this.say()
    }
}

const bird = new Bird();
bird.singMatingCall();
// tweet tweet
// hello world
```

## 一个对象使用多个装饰器

当一个对象使用多个装饰器装饰时，就是一个洋葱模型，从外向内进去，再从内向外执行。

```jsx
function decorator1() {
    console.log('decorator1');
    return function decFn1(targetClass) {
        console.log('decFn1');
        return targetClass;
    };
}

function decorator2() {
    console.log('decorator2');
    return function decFn2(targetClass) {
        console.log('decFn2');
        return targetClass;
    };
}

@decorator1()
@decorator2()
class TargetClass{
}
//decorator1
//decorator2
//decFn2
//decFn1
```

参考链接

1. https://www.zoo.team/article/decorator
2. https://www.telerik.com/blogs/decorators-in-javascript
