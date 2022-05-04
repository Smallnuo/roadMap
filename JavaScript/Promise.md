![image](https://user-images.githubusercontent.com/51777605/166630874-f0044457-c211-478c-bb35-e26110f56bed.png)

# Promise 基础

## 认识 Promise

Promise 是异步编程的一种解决方案。我们可以在返回的 Promise 实例对象上绑定不同结果下对应的回调函数(then、catch、finally)，针对不同的情况做处理。

1. 拥有三种状态 pending，fulfilled 和 rejected。

```jsx
const p1 = new Promise(() => {});
console.log(p1);//pending

const p2 = new Promise((resolve) => {
  resolve();
});
setTimeout(() => {
  console.log(p2);//fulfilled
});

const p3 = new Promise((resolve, reject) => {
  reject();
});
setTimeout(() => {
  console.log(p3);//rejected
});
```

1. 状态一旦发生更改，便无法改变。

```jsx
const p = new Promise((resolve, reject) => {
  resolve();
  reject();
});
setTimeout(() => {
  console.log(p);//fulfilled
});
```

1. 新建后就会立即执行。

```jsx
const p = new Promise((resolve, reject) => {
  console.log(1);
  resolve();
});
console.log(2);
// 1 2
```

1. 当一个 promise 对象 resolve 另一个对象时，两者会互相依赖；但是 reject 不会

```jsx
const p1 = new Promise((resolve, reject) => {});
const p2 = new Promise((resolve, reject) => {
  resolve(p1);//p2 依赖 p1，p2 自己的状态无效 没有任何输出
});
p2.then(
  (res) => {
    console.log('resolve');
    console.log(res);
  },
  (e) => {
    console.log('reject');
    console.log(e);
  }
);
```

1. promise 的 resolve 和 reject 并不会中断后续代码的执行，一般情况下我们不建议在 resolve 和 reject 后再继续执行代码

```jsx
new Promise((resolve, reject) => {
  resolve(1);
  console.log(2);
}).then((r) => {
  console.log(r);
});
//2 1
```

1. Promise 循环引用

```jsx
// Promise 循环引用导致系统异常，但并不会中断代码执行
const promise = new Promise((resolve, reject) => {
  resolve(100);
});
const p1 = promise.then((value) => {
  return p1;
});
setTimeout(() => {
  console.log("timer"); // timer
}, 100);
```

## Promise.prototype.then

1. then 方法会接收两个回调函数，分别在 promise 对象 resolve 和 reject 时进行调用

```jsx
const p = new Promise((resolve) => {
  resolve();
});
p.then(
  () => {
    console.log('resolve');
  },
  () => {
    console.log('reject');
  }
);
//resolve

const p = new Promise((resolve, reject) => {
  reject();
});
p.then(
  () => {
    console.log('resolve');
  },
  () => {
    console.log('reject');
  }
);

// reject
```

1. 在 then 的回调函数中如果返回一个 promise，那么链式调用的下一个 then 就会依赖它

```jsx
const p = new Promise((resolve, reject) => {
  reject();
});
p.then(
  () => {
    console.log("resolve1");
  },
  () => {
    console.log("reject1");
    return new Promise((resolve, reject) => {});
  }
).then(
  () => {
    console.log("resolve2");
  },
  () => {
    console.log("reject2");
  }
);
//reject1
```

1. then 中的回调函数会放入到微任务队列中

```jsx
Promise.resolve()
  .then(() => {
    console.log(1);
  })
  .then(() => {
    console.log(2);
  });

Promise.resolve()
  .then(() => {
    console.log(3);
  })
  .then(() => {
    console.log(4);
  });
// 1
// 3
// 2
// 4
```

## **Promise.prototype.catch**

1. catch 本质上是调用了 .then(null, rejection) 或 .then(undefined, rejection)

```jsx
const fetchAPI = () => {
  return new Promise((resolve, reject) => {
    reject(new Error('资源下载出错'));
  });
};
fetchAPI()
  .then(() => {
    console.log('then');
  })
  .catch((e) => {
    console.log(e);
  });
// 资源下载出错
```

1. promise 不会将异常抛出到外面，不会中断外部代码的执行，需要在 catch 中进行捕获

```jsx
const someAsyncThing = function () {
  return new Promise(function (resolve, reject) {
    // 下面一行会报错，因为x没有声明
    resolve(x + 2);
  });
};
someAsyncThing()
  .catch(function (error) {
    console.log('oh no', error);
  })
  .then(function () {
    console.log('carry on');
  });

console.log(22);
// 22 
// oh no ReferenceError {}
// carry on
```

1. catch 中也可以抛出错误，并且可以被后面的 catch 捕获
2. 如果多个 then 都有错误 catch 会捕获到第一个的错误，本质上后面两个 .then 的回调函数并不会被执行

```jsx
const someAsyncThing = function () {
  return new Promise(function (resolve, reject) {
    resolve(x);
  });
};
someAsyncThing()
  .then(function () {
    y;
  })
  .then(function () {
    z;
  })
  .catch(function (error) {
    console.log('oh no', error);
  })
  .then(function () {
    console.log('carry on');
  });
// oh no x ReferenceError
// carry on
```

## **Promise.prototype.finally**

不管 promise 的结果如何，都会调用 finally 回调函数的代码

```jsx
const someAsyncThing = function () {
  return new Promise(function (resolve, reject) {
    resolve();
  });
};
someAsyncThing()
  .finally(() => {
    console.log('finally');
  })
  .then(function () {
    console.log('carry on');
  });
//finally
//carry on

const someAsyncThing = function () {
  return new Promise(function (resolve, reject) {
    reject();
  });
};
someAsyncThing()
  .finally(() => {
    console.log('finally');
  })
  .then(function () {
    console.log('carry on');
  });
//finally
//carry on
```

## Promise.resolve

1. 将现有对象转为 Promise 对象

```jsx
Promise.resolve('foo')
// 等价于
new Promise(resolve => resolve('foo'))
```

- 转化的对象为 promise 对象，则直接返回该对象，不做任何修改

```jsx
const p = new Promise((resolve) => resolve());
const p1 = Promise.resolve(p);
const p2 = new Promise((resolve) => {
  resolve(p);
});
console.log(p1 === p); //true
console.log(p2 === p); // false
```

- 转化的对象为 thenable 对象，会立即调用其 then 方法

```jsx
let thenable = {
  then: (resolve, reject) => {
    resolve(1);
  },
};
let p1 = Promise.resolve(thenable);
p1.then((value) => {
  console.log(value); // 1
});
```

- 普通的类型或者不带参数

```jsx
const p = Promise.resolve('Hello');
p.then(function (s) {
  console.log(s); // Hello
});
```

## **Promise.reject**

- reject 方法会返回一个新的 Promise 实例

```jsx
const p = Promise.reject('出错了');
// 等同于
const p = new Promise((reject) => reject('出错了'))
```

# Promise API 与场景

## Promise 链式调用与递归

题目：红灯三秒亮一次，绿灯一秒亮一次，黄灯2秒亮一次；如何让三个灯不断交替重复亮灯？（用 Promse 实现）

```jsx
function red() {
  console.log("red");
}
function green() {
  console.log("green");
}
function yellow() {
  console.log("yellow");
}

var light = function (timmer, cb) {
  return new Promise(function (resolve, reject) {
    setTimeout(function () {
      cb();
      resolve();
    }, timmer);
  });
};

var step = function () {
  Promise.resolve()
    .then(function () {
      return light(3000, red);
    })
    .then(function () {
      return light(2000, green);
    })
    .then(function () {
      return light(1000, yellow);
    })
    .then(function () {
      step();
    });
};

step();
```

## Promise.retry

```jsx
Promise.retry = function (promiseFn, times = 3) {
  return new Promise((resolve, reject) => {
    while (times--) {
      try {
        var ret = promiseFn();
        resolve(ret);
        break;
      } catch (error) {
        if (!times) reject(error);
      }
    }
  });
};
function getProm() {
  const n = Math.random();
  return new Promise((resolve, reject) => {
    setTimeout(() => (n > 0.1 ? resolve(n) : reject(n)), 1000);
  });
}
Promise.retry(getProm)
  .then((value) => {
    console.log("value", value);
  })
  .catch((error) => {
    console.log("error", error);
  });
```

## Promise 超时

Promise 超时使用 Promise.race(Iterator) API 实现，我们首先来理解一下这个API。

参数：可迭代对象

返回值：一个待定的 Promise 只要给定的迭代中的一个promise解决或拒绝，就采用第一个promise的值作为它的值

特点：

1. 异步性
2. 采用第一个状态改变的 Promise 的值作为它的值

```jsx
let timerFunc = (timer, message) => {
  return new Promise((resolve, reject) => {
    setTimeout(() => {
      resolve(message);
    }, timer);
  });
};

let p1 = timerFunc(30, "promise1");
let p2 = timerFunc(1000, "promise2");
let p3 = timerFunc(60, "promise3");
Promise.race([p1, p2, p3]).then((data) => {
  console.log(data); // promise1
});
```

1. 超时场景实现

```jsx
function promiseTimeout(promise, delay) {
  let timeout = new Promise(function (reslove, reject) {
    setTimeout(function () {
      reject("超时啦~");
    }, delay);
  });
  return Promise.race([timeout, promise]);
}

function foo() {
  return new Promise(function (reslove, reject) {
    setTimeout(function () {
      reslove("request sucess!");
    }, 300);
  });
}

promiseTimeout(foo(), 200).then(
  function (data) {
    console.log(data); // 超时啦
  },
  function (err) {
    console.log(err);
  }
);
```

## Promise 并行与串行

我们看下面这个并行的例子：

```jsx
// forEach 回调函数是一个个独立的匿名函数
function square(num) {
  return new Promise((resolve, reject) => {
    setTimeout(() => {
      resolve(num * num);
    }, 1000);
  });
}
function test(list) {
  list.forEach(async (element) => {
    let res = await square(element);
    console.log(res); // 同时打印 1 4 9
  });
}
test([1, 2, 3]);
```

如何将上面的并行例子改为串行执行呢？

1. 使用 for of 循环或普通循环代替

```jsx
function square(num) {
  return new Promise((resolve, reject) => {
    setTimeout(() => {
      resolve(num * num);
    }, 1000);
  });
}
async function test(list) {
  for (let element of list) {
    let res = await square(element);
    console.log(res); // 串行打印 1 4 9
  }
}
test([1, 2, 3]);
```

1. 递归

```jsx
const list = [1, 2, 3];
function square(num) {
  return new Promise((resolve, reject) => {
    setTimeout(() => {
      resolve(num * num);
    }, 1000);
  });
}
function test(i) {
  if (i === list.length) return;
  square(list[i]).then((res) => console.log(res)); // 串行打印 1 4 9
}
test(0);
```

1. reduce

```jsx
let list = [1, 2, 3];
list.reduce((pre, next) => {
  return pre.then(() => {
    return new Promise((resolve) => {
      setTimeout(() => resolve(console.log(next * next)), 1000); 串行打印 1 4 9
    });
  });
}, Promise.resolve());
```

Promise.all(Iterator) 和 Promise.race(Iterator) 也是并行执行任务的。在介绍 Promise 超时部分时我们已经了解了 Promise.rece(Iterator)，下面我们来看看 Promise.all(Iterator)。

```jsx
Promise.all([p1, p2, p3])
```

参数：Iterator 对象

返回值：

1.只有p1、p2、p3的状态都变成fulfilled，p的状态才会变成fulfilled，此时p1、p2、p3的返回值组成一个数组，传递给p的回调函数。
2.只要p1、p2、p3之中有一个被rejected，p的状态就变成rejected，此时第一个被reject的实例的返回值，会传递给p的回调函数。

特点：

1. 异步性
2. 快速返回失败行为

```jsx
let timerFunc = (timer) => {
  return new Promise((resolve, reject) => {
    setTimeout(() => {
      resolve(timer);
    }, timer);
  });
};

let p1 = Promise.resolve(1);
let p2 = timerFunc(1000);
let p3 = Promise.reject("throw error");
Promise.all([p1, p2, p3])
  .then((value) => {
    console.log(value);
  })
  .catch((err) => {
    console.log(err); // throw error
  });
```

处理并行任务选择 Promise.all(Iterator) 是一个不错的选择，但 Promise.all(Iterator) 对于错误非常敏感，一旦发生错误其他任务都不进行执行，如果你想发生错误也继续执行可以选择 Promise.allSettled(Iterator)。

返回值：只有等到参数数组的所有 Promise 对象都发生状态变更（不管是fulfilled还是rejected），返回的 Promise 对象才会发生状态变更。

```jsx
let timerFunc = (timer) => {
  return new Promise((resolve, reject) => {
    setTimeout(() => {
      resolve(timer);
    }, timer);
  });
};

let p1 = Promise.resolve(1);
let p2 = timerFunc(1000);
let p3 = Promise.reject("throw error");
Promise.allSettled([p1, p2, p3])
  .then((value) => {
    console.log(value); // 结果如下图
  })
  .catch((err) => {
    console.log(err);
  });
```

![image](https://user-images.githubusercontent.com/51777605/166634020-a4b430c2-6f5f-4dd7-bc26-3c1a64337fb4.png)

## Promise.promisify

1. callback 处理返回结果，容易形成回调地狱

```jsx
let fs = require("fs");
let path = require("path");
fs.readFile(
  path.resolve(__dirname, "text.txt"),
  "utf8",
  function (err, content) {
    if (err) {
      console.log(err);
    } else {
      console.log(content);
    }
  }
);
```

1. 改写成 Promise 写法

```jsx
let fs = require("fs");
let path = require("path");
let { promisify } = require("util");
let promiseReadFile = promisify(fs.readFile);
promiseReadFile(path.resolve(__dirname, "text.txt"), "utf8").then(
  (content) => {
    console.log(content);
  },
  (error) => {
    console.log(error);
  }
);
```

1. 实现 promisify

```jsx
let fs = require("fs");
let path = require("path");

const promisify = function (fn) {
  return function (...args) {
    return new Promise((resolve, reject) => {
      fn.apply(this, [
        ...args,
        (err, content) => {
          return err ? reject(err) : resolve(content);
        },
      ]);
    });
  };
};

let promiseReadFile = promisify(fs.readFile);
promiseReadFile(path.resolve(__dirname, "text.txt"), "utf8").then(
  (content) => {
    console.log(content);
  },
  (error) => {
    console.log(error);
  }
);
```

## Promise 局限性与反模式

### 反模式

**1.Promise 嵌套**

```jsx
// bad
loadSomething().then(function(something) {
    loadAnotherthing().then(function(another) {
        DoSomethingOnThem(something, another);
    });
});

// good
Promise.all([loadSomething(), loadAnotherthing()])
.then(function ([something, another]) {
    DoSomethingOnThem(...[something, another]);
});
```

**2.断开的 Promise 链**

```jsx
// bad
function anAsyncCall() {
    var promise = doSomethingAsync();
    promise.then(function() {
        somethingComplicated();
    });

    return promise;
}

// good
function anAsyncCall() {
    var promise = doSomethingAsync();
    return promise.then(function() {
        somethingComplicated()
    });
}
```

**3.混乱的集合**

```jsx
// bad
function workMyCollection(arr) {
    var resultArr = [];
    function _recursive(idx) {
        if (idx >= resultArr.length) return resultArr;

        return doSomethingAsync(arr[idx]).then(function(res) {
            resultArr.push(res);
            return _recursive(idx + 1);
        });
    }

    return _recursive(0);
}
```

你可以写成：

```jsx
function workMyCollection(arr) {
    return Promise.all(arr.map(function(item) {
        return doSomethingAsync(item);
    }));
}
```

**4.catch**

```jsx
// bad
somethingAync.then(function() {
    return somethingElseAsync();
}, function(err) {
    handleMyError(err);
});
```

如果 somethingElseAsync 抛出错误，是无法被捕获的。你可以写成：

```jsx
// good
somethingAsync
.then(function() {
    return somethingElseAsync()
})
.then(null, function(err) {
    handleMyError(err);
});

// good
somethingAsync()
.then(function() {
    return somethingElseAsync();
})
.catch(function(err) {
    handleMyError(err);
});
```

### 局限性

1. 错误被吃掉

```jsx
let promise = new Promise(() => {
    throw new Error('error')
});
console.log(2333333); // 2333333
```

2. 单一值

Promise 只能有一个完成值或一个拒绝原因，然而在真实使用的时候，往往需要传递多个值，一般做法都是构造一个对象或数组，然后再传递，then 中获得这个值后，又会进行取值赋值的操作，每次封装和解封都无疑让代码变得笨重。

并没有什么好的方法，建议是使用 ES6 的解构赋值：

```jsx
Promise.all([Promise.resolve(1), Promise.resolve(2)])
.then(([x, y]) => {
    console.log(x, y);
});
```

3. 无法取消

Promise 一旦新建它就会立即执行，无法中途取消。

4. 无法得知 pending 状态

当处于 pending 状态时，无法得知目前进展到哪一个阶段（刚刚开始还是即将完成）。

## Promise.any()

返回值：空的可迭代对象，返回一个 fulfilled；只要参数实例有一个变成 fulfilled 状态，包装实例就会变成 fulfilled 状态；如果所有参数实例都变成 rejected 状态，包装实例就会变成rejected 状态。

```jsx
const pErr = new Promise((resolve, reject) => {
  reject("总是失败");
});

const pSlow = new Promise((resolve, reject) => {
  setTimeout(resolve, 500, "最终完成");
});

const pFast = new Promise((resolve, reject) => {
  setTimeout(resolve, 100, "很快完成");
});

Promise.any([pErr, pSlow, pFast]).then((value) => {
  console.log(value);
  // pFast fulfils first
})
// 期望输出: "很快完成"
```

场景：显示第一张加载的图片

```jsx
function fetchAndDecode(url) {
  return fetch(url).then((response) => {
    if (!response.ok) {
      throw new Error(`HTTP error! status: ${response.status}`);
    } else {
      return response.blob();
    }
  });
}

let coffee = fetchAndDecode("coffee.jpg");
let tea = fetchAndDecode("tea.jpg");

Promise.any([coffee, tea])
  .then((value) => {
    let objectURL = URL.createObjectURL(value);
    let image = document.createElement("img");
    image.src = objectURL;
    document.body.appendChild(image);
  })
  .catch((e) => {
    console.log(e.message);
  });
```
