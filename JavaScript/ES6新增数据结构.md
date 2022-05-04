![image](https://user-images.githubusercontent.com/51777605/166628913-f335c82e-40b8-4270-b9e9-6921369876c9.png)

## Symbol

### 通识

> Symbol：ES6 提出的一种新的原始数据类型，表示独一无二的值
>

**Symbol 基础认识**

```jsx
// 通过 Symbol 函数创建，不能使用 new 进行生成，因为 Symbol 是一个基础类型，而不是对象
let s = Symbol('foo');
console.log(typeof s); // symbol
// 无原型链
console.log(s instanceof Symbol); // false

/* Symbol([description])
	description: string | Object 对Symbol实例的描述，便于区分
  string: 可以显示 sym.toString/String(sym) 查看描述信息，也可以 description 查看  
  Object: 调用 toString 方法转换成字符串
*/
let s1 = Symbol('11')
let s2 = Symbol({
    toString(){
        return '22'
    }
})
console.log(s1.toString()) // Symbol(11)
console.log(String(s1)) //Symbol(11)
console.log(s2.description) // 22

//Symbol 值不能与其他类型的值进行运算，会报错
let sym = Symbol('nuonuo')
console.log('my name is' + sym )// Cannot convert a Symbol value to a string
// 转布尔值
console.log(Boolean(sym)) // symbol 是真值，永远为 true
console.log(Number(sym)); // Cannot convert a Symbol value to a number
```

**作为属性名的 Symbol**

ES5 对象属性名都是字符串，容易造成属性名冲突。ES6 提出 Symbol 值可以作为标识符，用于对象的属性名，可以保证不会出现同名的属性。Symbol 做属性名时不能使用点运算。

```jsx
var mySymbol = Symbol();

// 第一种写法
var a = {};
a[mySymbol] = 'Hello!';

// 第二种写法
var a = {
  [mySymbol]: 'Hello!'
};

// 第三种写法
var a = {};
Object.defineProperty(a, mySymbol, { value: 'Hello!' });

// 以上写法都得到同样结果
console.log(a[mySymbol]); // "Hello!"
```

属性名遍历

Symbol 作为属性名，该属性不会出现在 for...in、for...of 循环中，也不会被 Object.keys()、Object.getOwnPropertyNames()、JSON.stringfy() 返回。但是，它也不是私有属性，有一个 Object.getOwnPropertySymbols 方法，可以获取指定对象的所有 Symbol 属性名。

| Reflect.ownKeys(obj)              | 目标对象自身的属性键组成的数组             |
|-----------------------------------|-----------------------------|
| Object.keys(obj)                  | 可枚举属性                       |
| Object.getOwnPropertySymbols(obj) | 遍历对象 Symbol 属性名             |
| Object.getOwnPropertyNames(obj)   | 包括不可枚举属性但不包括 Symbol值作为名称的属性 |
| for in                            | 遍历对象                        |
| for of                            | 遍历具有 Iterator 结构的数据         |

```jsx
var obj = {};
var a = Symbol('a');
var b = Symbol('b');

obj[a] = 'Hello';
obj[b] = 'World';

var objectSymbols = Object.getOwnPropertySymbols(obj);

console.log(objectSymbols);
// [Symbol(a), Symbol(b)]
```

**Symbol.for()与Symbol.keyFor()**

Symbol.for()

如果我们希望使用同一个 Symbol 值，可以使用 Symbol.for。它接受一个字符串作为参数，然后搜索有没有以该参数作为名称的 Symbol 值。如果有，就返回这个 Symbol 值，否则就在全局环境中登记并创建一个新的 Symbol 值返回。

```jsx
let s1 = Symbol('foo');
let s2 = Symbol('foo');
console.log(s1 === s2) // false

let s1 = Symbol.for('foo');
let s2 = Symbol.for('foo');
console.log(s1 === s2); // true
```

Symbol.keyFor()

返回一个已登记的 Symbol 类型值的 key。

```jsx
let s1 = Symbol.for("foo");
Symbol.keyFor(s1) // "foo"

let s2 = Symbol("foo");
Symbol.keyFor(s2) // undefined,变量 s2 属于未登记的 Symbol 值
```

### 内置的 Symbol 值

- Symbol.hasInstance

对象的 ****Symbol.hasInstance**** 属性，指向一个内部方法。当其他对象使用 **instanceof** 运算符，判断是否为该对象的实例时，会调用这个方法。

- Symbol.iterator

****Symbol.iterator**** 属性指向该对象的默认遍历器方法

```jsx
class Collection {
  *[Symbol.iterator]() {
    let i = 0;
    while(this[i] !== undefined) {
      yield this[i];
      ++i;
    }
  }
}

let myCollection = new Collection();
myCollection[0] = 1;
myCollection[1] = 2;

for(let value of myCollection) {
  console.log(value); 
}
// 1
// 2
```

- Symbol.toStringTag

****Symbol.toStringTag**** 指向一个方法，在该对象上面调用 Object.prototype.toString 方法时，如果这个属性存在，它的返回值会出现在 toString 方法返回的字符串之中，表示对象的类型。

```jsx
// ES6 内置的 Symbol.toStringTag
let toString = Object.prototype.toString
toString.call(new Set()) // '[object Set]'
toString.call(null) // '[object Null]'

// 自定义
class Collection {
  get [Symbol.toStringTag]() {
    return 'qianxun';
  }
}
let x = new Collection();
Object.prototype.toString.call(x) // '[object qianxun]'
```

- Symbol.toPrimitive

****Symbol.toPrimitive**** 是一个内置的 Symbol 值，当一个对象转换为对应的原始值时，会调用此函数。

```jsx
let obj = {
  [Symbol.toPrimitive](hint) {
    switch (hint) {
      case 'number':
        return 123;
      case 'string':
        return 'str';
      case 'default':
        return 'default';
      default:
        throw new Error();
     }
   }
};

2 * obj // 246
3 + obj // '3default'
obj == 'default' // true
String(obj) // 'str'
```

**Map**

**基础知识**

集合与字典的区别：

- 共同点：集合、字典可以储存不重复的值
- 不同点：集合 是以 [value, value]的形式储存元素，字典是以 [key, value] 的形式储存

```jsx
// 参数：具有 iterable 接口的数据
// Map 是[key,value]形式存储，所以数据只能是二维数组而不是一维数组
const items = [
    ['name', 'qianxun'],
    ['frineds', ['mumiao', 'shuangxu']]
];
let map = new Map(items);
console.log(map) // Map(2) {'name' => 'qianxun', 'frineds' => Array(2)}
```

下面代码的 set 和 get 方法，表面是针对同一个键，但实际上这是两个值，内存地址是不一样的，因此 get 方法无法读取该键，返回 undefined。

```jsx
let map = new Map();

map.set(['a'], 555);
console.log(map.get(['a'])) // undefined
```

由上可知，Map 的键实际上是跟内存地址绑定的，只要内存地址不一样，就视为两个键。这就解决了同名属性碰撞的问题，我们扩展别人的库的时候，如果使用对象作为键名，就不用担心自己的属性与原作者的属性同名。

如果 Map 的键是一个简单类型的值（数字、字符串、布尔值），则只要两个值严格相等，Map 将其视为一个键，比如 0 和 -0 就是一个键，布尔值 true 和字符串 true 则是两个不同的键。另外，undefined 和 null 也是两个不同的键。虽然 NaN 不严格相等于自身，但 Map 将其视为同一个键。

```jsx
let map = new Map();
map.set(NaN, '1')
map.set(NaN, '2')
console.log(map.size) // 1
```

Map **实例的属性与方法**

```jsx
// Map 结构的默认遍历器接口（Symbol.iterator属性），就是entries方法
map[Symbol.iterator] === map.entries // true
```

**Map 与其他数据结构相互转换**

- Map 转数组

```jsx
const myMap = new Map().set(true, 7).set({foo: 3}, ['abc']);
[...myMap] // [ [ true, 7 ], [ { foo: 3 }, [ 'abc' ] ] ]
```

- 数组转 Map

```jsx
new Map([
  [true, 7],
  [{foo: 3}, ['abc']]
])
// Map {
//   true => 7,
//   Object {foo: 3} => ['abc']
// }
```

- Map 转对象

```jsx
// 因为 Object 的键名都为字符串，而 Map 的键名为对象，
// 所以转换的时候会把非字符串键名转换为字符串键名
function strMapToObj(strMap) {
  let obj = Object.create(null);
  for (let [k,v] of strMap) {
    obj[k] = v;
  }
  return obj;
}

const myMap = new Map()
  .set('yes', true)
  .set({name: 'qianxun'}, false);
console.log(strMapToObj(myMap))// { yes: true, no: false }
```

- 对象转 Map

```jsx
let obj = {"a":1, "b":2};
let map = new Map(Object.entries(obj));
```

****WeakMap****

WeakMap 与 Map 的区别：

1. WeakMap 只接受对象作为键名（null除外），不接受其他类型的值作为键名
2. WeakMap 的**键名**所指向的对象是弱引用，不计入垃圾回收机制

![image](https://user-images.githubusercontent.com/51777605/166629029-f1297448-dcf7-4969-98b6-9429a958a7ed.png)

1. 没有遍历操作（即没有 keys()、values() 和 entries()方法），也没有 size 属性
2. 无法清空，不支持 clear()

**WeakMap 的应用场景**

1. 部署私有属性

```jsx
const privateData = new WeakMap();

class Person {
  constructor(name, age) {
    this.age = age;
    privateData.set(this, { name: name });
  }

  getName() {
    return privateData.get(this).name;
  }
}
const p = new Person("qianxun", 3);
console.log(p); // Person {age: 3}
console.log(p.getName()); // qianxun
```

1. 额外数据
![image](https://user-images.githubusercontent.com/51777605/166629109-7301e038-19f5-4706-b23c-5e9c41cb97c6.png)

2. 数据缓存
![image](https://user-images.githubusercontent.com/51777605/166629225-16b81a1e-298e-49a3-96db-0010e337f593.png)

3. 管理 DOM 元素

```jsx
const div = document.querySelector('div')
let map = new Map()
map.set(div, 'foo')
div = null  // 不再需要这个对象
map.delete(div) // 手动解除map中的引用

const div = document.querySelector('div')
let weakmap = new WeakMap()
weakmap.set(div, 'foo')
div = null  // 不再需要这个对象
```

## **Set**

Set 是一种类数组，集合（元素不重复）数据结构

**基础知识**

> new Set([Iterable])
>

```jsx
// 参数：数组或具有 iterable 接口的其他数据结构（类数组）
const s = new Set([1,1,2,3,5])
console.log(s) // Set(4) {1, 2, 3, 5}

const s = new Set({
    a: 'a',
    b: 'b'
}) // object is not iterable (cannot read property Symbol(Symbol.iterator))
```

Set.add() 时不会发生类型转换，所以 5 和 ‘5’ 是两个不同的值。Set 内部判断值是否相同根据Same-value-zero equality算法，类似于 === ，区别是 NaN 在 Set 时会判断他们是相同的。

```jsx
let set = new Set();
set.add(NaN);
set.add(NaN);
console.log(set); // Set(1) {NaN}
```

Set **实例的属性与方法**

| 属性      | Set.prototype.constructor   | 构造函数，默认就是Set函数          |
|---------|-----------------------------|-------------------------|
|         | Set.prototype.size          | 返回Set                   |
| 实例的成员总数 |                             |                         |
| 操作方法    | Set.prototype.add(value)    | 添加某个值，返回 Set 结构本身       |
|         | Set.prototype.delete(value) | 删除某个值，返回一个布尔值，表示删除是否成功。 |
|         | Set.prototype.has(value)    | 返回一个布尔值，表示该值是否为Set      |
| 的成员     |                             |                         |
|         | Set.prototype.clear()       | 清除所有成员，没有返回值            |
| 遍历方法    | Set.prototype.keys()        | 返回键名的遍历器                |
|         | Set.prototype.values()      | 返回键值的遍历器                |
|         | Set.prototype.entries()     | 返回键值对的遍历器               |
|         | Set.prototype.forEach()     | 使用回调函数遍历每个成员            |

```jsx
// 默认遍历器生成函数是它的 values 方法
Set.prototype[Symbol.iterator] === Set.prototype.values // true
```

**Set 应用场景**

```jsx
let a = new Set([1, 2, 3]);
let b = new Set([4, 3, 2]);

// 并集
let union = new Set([...a, ...b]);
// Set {1, 2, 3, 4}

// 交集
let intersect = new Set([...a].filter(x => b.has(x)));
// set {2, 3}

// （a 相对于 b 的）差集
let difference = new Set([...a].filter(x => !b.has(x)));
// Set {1}
```

## ****WeakSet****

**WeakSet 与 Set 的区别**：

1. WeakSet 的成员只能是对象，而不能是其他类型的值
2. WeakSet 中的对象都是弱引用
3. WeakSet 不可遍历

| WeakSet.prototype.add(value)    | 向 WeakSet 实例添加一个新成员           |
|---------------------------------|-------------------------------|
| WeakSet.prototype.delete(value) | 清除 WeakSet 实例的指定成员            |
| WeakSet.prototype.has(value)    | 返回一个布尔值，表示某个值是否在 WeakSet 实例之中 |
