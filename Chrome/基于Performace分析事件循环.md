# 基于 Performace 分析事件循环

## 什么是事件循环?

我们为什么需要事件循环？对于 JavaScript 是一门单线程语言我们是肯定的，JavaScript 单线程的特性保证了渲染和 JavaScript 的正常运行，但同时也存在一定的限制。理想情况下我们希望所有任务是串行执行的，假设串行中存在一个耗时很多的任务时，会阻塞后续任务的运行，这种情况我们怎么去解决呢？这个时候就需要我们的事件循环来处理了。

![image](https://user-images.githubusercontent.com/51777605/166667829-38c33664-5474-4738-b1a6-c9240635740d.png)


## 让人意外的setTimeout

菜鸟教程

> setTimeout() ：在指定的毫秒数后调用函数或计算表达式
>

```js
console.log(1);
setTimeout(()=>{
	console.log(2);
},0)
for (let i = 0; i < 5000; i++) { 
	let sum = 0;
	sum += i; 
}
console.log(3);
```

猜猜上面这段代码执行结果是多少呢？根据 Event Loop 机制我们知道答案是1、3、2。但是针对这段代码中有一个疑问点，0ms 是指 0ms 后执行 callback 吗？答案是否定的，定时器任务被维护在定时器线程中，添加一个定时器时开始计时这个任务，0ms 后会将 callback 添加到事件队列中，如果在事件队列中存在 long Task，定时器的 callback 将等待执行，查看 Performance 执行过程：

![image](https://user-images.githubusercontent.com/51777605/166667904-553187f9-bf00-4ac0-be0a-af81503689f4.png)

## 宏任务与微任务

我们先回顾一下之前 Event Loop 图，在图中描述了函数调用栈、Web Api、一个消息队列，并没有提到宏任务与微任务，那么宏任务与微任务是什么呢？为什么要有微任务呢？我们先来看一个例子：

```jsx
function timerCallback2(){ 
    console.log(2);
} 
function timerCallback(){ 
    console.log(1);
    setTimeout(timerCallback2,0);
} 
setTimeout(timerCallback,0);
```

我们希望通过 setTimeout 按照顺序执行 callback，通过 Performance 发现，在两个任务中间插入了其他任务，如果插入任务是 long task，会影响后续任务的执行。宏任务是浏览器提供给我们的Web Api，时间颗粒度较大，针对像 DOM 等高实时性操作是不太符合的。

![image](https://user-images.githubusercontent.com/51777605/166668041-993d8b7b-ee1d-46ec-ad01-1a8ea7233189.png)

为了满足这种高优先级的任务，V8 引擎在创建全局执行上下文时会在内部创建一个微任务队列，在当前宏任务执行完成时去检查微任务队列，我们把执行微任务的时间点叫检查点。了解了微任务队列后，我们丰富一下之前的 Event Loop。

![image](https://user-images.githubusercontent.com/51777605/166668111-1b0c2b6c-8f1b-4261-bc2a-6ddc03da5901.png)

```js
console.log(1)
new Promise(function (resolve) {
  console.log(2)
  resolve()
}).then(function () {
  console.log(3)
})
console.log(4)
```
![image](https://user-images.githubusercontent.com/51777605/166668222-f35f686f-2619-4720-846b-08e364a9dd9f.png)

## 事件循环与渲染

浏览器按照帧渲染方式一帧一帧渲染网页，但并不是每一帧都会经历管道每个部分的处理。当脚本执行阻塞时会导致后续渲染流畅阻塞，页面卡顿。

![image](https://user-images.githubusercontent.com/51777605/166668508-9e6e9f4d-7b12-4fad-93b4-8402db6665e4.png)

浏览器何时渲染对于我们来说就是一个黑盒，浏览器自身会去判断当前是否需要进行渲染，因此性能优化的是管道帧的流水过程，比如减少脚本执行时长，避免重绘、重排。如果你希望在每轮事件循环中都能变动，你需要去了解一下 requestAnimationFrame。

![image](https://user-images.githubusercontent.com/51777605/166668600-61661a96-e8a2-42df-ae27-e3c3b9433acb.png)

```js
<div id='con'>this is con</div>
<script>
var con = document.getElementById('con');
con.onclick = function () {
    setTimeout(function setTimeout1() {
       con.textContent = 0;
       Promise.resolve().then(function Promise1 () {
            console.log('Promise1')
      })
    }, 0)
    setTimeout(function setTimeout2() {
       con.textContent = 1;
       Promise.resolve().then(function Promise2 () {
            console.log('Promise2')
       })
    }, 0)
};
</script>
```

当两个宏任务耗时不足一帧时，会发生渲染合并现象：

![image](https://user-images.githubusercontent.com/51777605/166668685-65d5a978-cd83-4f40-a89e-ece2e0ba8f91.png)

我们修改上诉代码如下，延长第二个宏任务执行时机：

```js
<div id='con'>this is con</div> 
<script> 
  var con = document.getElementById('con'); 
  con.onclick = function () {     
    setTimeout(function setTimeout1() {        
      con.textContent = 0;        
      Promise.resolve().then(function Promise1 () {
        console.log('Promise1')	 
        })     
    }, 0)     
    setTimeout(function setTimeout2() {        
      con.textContent = 1;        
      Promise.resolve().then(function Promise2 () {
        console.log('Promise2') 
      })     
    }, 17) }; 
</script>
```

两个宏任务执行时间间隔 17ms ，按照代码逻辑宏任务执行完毕就进行渲染，并未发生渲染合并现象。

![image](https://user-images.githubusercontent.com/51777605/166668804-def8bf00-b36d-4863-8228-a60c623eff69.png)

## 事件循环之任务拆分

作为提供数据中台服务的公司，我们不可避免会涉及到一些复杂数据到计算。下面这个例子我们需要计算从1到10 000 000 000数据加起来到合，计算完毕展示我们到弹框。通过 Performance 我们可以看见这个同步任务耗时快4s，浏览器每帧需达到60fps/s，也就是16.7ms每帧，在这个计算结束之前其他任务均得不到执行，导致后续渲染任务的延迟造成卡顿现象。

```js
let i = 0;
let start = Date.now();
function count() {
  // long Task
  for (let j = 0; j < 1e9; j++) {
      i++;
  }
  alert("Done in " + (Date.now() - start) + 'ms');
}
count();
```

![image](https://user-images.githubusercontent.com/51777605/166668862-10e3cf31-996c-4607-a88a-053687eb7139.png)

我们希望这个 long Task 拆分成一个个小的任务，解决长时间阻塞造成的卡顿现象。我们可以利用 setTimeout 拆分我们的任务，修改代码如下，这个计算确实被拆分成了一个个小任务。React Firber架构中也使用了任务拆分这种思想将递归渲染 vdom 转为了链表可中断渲染 vdom，笔者对 Fiber了解并不多，这部分就不展开细说了。

```js
let i = 0;
let start = Date.now();
function count() {
    // long Task
    do{
        i++;
    }while(i % 1e6 != 0)
    if(i == 1e9) {
        alert("Done in " + (Date.now() - start) + 'ms');
    } else {
        setTimeout(count);
    }
}
count();
```

![image](https://user-images.githubusercontent.com/51777605/166668909-5cdbdd9a-bd7c-49b6-ae07-b3754f409a0a.png)

setTimeout 确实将任务进行了拆分处理，但仍占用了主线程资源，我们知道主线程保证了页面的渲染、脚本交互、布局等操作，上诉这种单纯的数据计算放在主线程处理是没有意义的，我们可以将耗时计算放在 Web Worker 中处理。

## 事件循环优化之Web Worker

- 什么是 Web Worker?

<aside>
当在 HTML 页面中执行脚本时，页面的状态是不可响应的，直到脚本已完成。web worker 是运行在后台的 JavaScript，独立于其他脚本，不会影响页面的性能。您可以继续做任何愿意做的事情：点击、选取内容等等，而此时 web worker 在后台运行。

</aside>

- Web Worker 工作原理

![image](https://user-images.githubusercontent.com/51777605/166669146-ef095296-cc9d-4911-b3eb-5f8489f37be6.png)

- Web Worker 改变了 JavaScript 单线程执行这一本质了吗？

并没有改变 JavaScript 是单线程执行这一本质。JavaScript 是一门没有定义线程模型的原型，Web Worker 并不是 JavaScript 的一部分，它是浏览器提供的一种创建线程的方式，所以在使用 Web Worker 时不能操作 DOM，这也就意味着我们不能使用 Web Worker 进行 UI 更新这种操作，但如果把 Web Worker 理解成一个计算器，处理繁重的计算任务，会让我们的主线程执行更加流畅。

```jsx
// worker.js 
self.onmessage = (e=>{
    const { startNum } = e.data;
    let sum = startNum;
    (function count() {
        // long Task
        for (let j = 0; j < 1e9; j++) {
            sum++;
        }
    })();
    self.postMessage(sum);
})

// test.html 
let start = Date.now();
let worker = new Worker('worker.js');
worker.postMessage({startNum: 0});
worker.onmessage = (e) => {
  alert("Done in " + (Date.now() - start) + 'ms');
}
```

![image](https://user-images.githubusercontent.com/51777605/166669258-7bf2e7a3-fd2b-47c3-8a22-14059d9d5c6a.png)

## 总结

笔者最初学习事件循环时只会判断一段简单代码片段的输出结果，查阅网上资料发现大部分资料也是这样介绍事件循环的，这导致笔者思维长时间聚焦在一段脚本的输出结果，接触 Web Worker 时针对复杂计算开一个线程的必要性也持有怀疑。通过这段时间的学习，让我理解到事件循环的本质是保证用户交互、脚本、UI 渲染有序进行的基石，脚本长时间的执行会导致后续任务阻塞，页面呈现卡顿现象，这也是为什么 Web Worker 采用开一个线程进行复杂计算的原因。
