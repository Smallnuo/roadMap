## 为什么需要 Etag ？

Etag 处理 Last-Modified 无法处理的场景。

1. Last-Modified 根据文件修改时间来判断，单位是秒。假如我们操作是毫秒级别，Last-Modified 颗粒度就不满足我们现在的诉求。
2. 服务器资源确实被修改了，但实质的内容并没有发生变化，这个时候 Last-Modified 也会更新，但这种情况我们并不希望浏览器去重新加载资源。在这些特殊场景下，我们就需要使用 Etag。Etag 优先级高于 Last-Modified。

## Etag 值如何生成

针对不同的资源 Etag 值生成方式是不一样的，我们学习一下 Etag 库中是如何生成 Etag 值的。

1. 静态资源使用文件大小和修改时间

```jsx
const promisify = require('util').promisify
const fs = require('fs')

const stat = promisify(fs.stat)

stat('./test.txt').then(res => {
    const mtime = res.mtime.getTime().toString(16)
    const size = res.size.toString(16)
    console.log('"' + size + '-' + mtime + '"')
})
```

Q：上面的 mtime 是什么？

A：mtime 表示内容数据更改时更新的时间戳，文件属性或权限更改时该时间戳不会变更

ctime 表示文件元数据更新时更新的时间戳，例如文件权限更改时该时间戳发生变更

atime 表示文件被读取时更新读取时间戳，例如 cat 命令查看文件时，该时间戳更新

2. 字符串，Buffer 和引用类型获取内容的部分 hash 值和内容长度

```jsx
const crypto = require('crypto')
const str = JSON.stringify({
    name:'leon'
})

if(!str.length){
    console.log('"0-2jmj7l5rSw0yVb/vlWAYkK/YBwk"')
}else{
    const hash = crypto
    .createHash('sha1')
    .update(str, 'utf8')
    .digest('base64')
    .substring(0, 27)
    
    const len = typeof str === 'string'
    ? Buffer.byteLength(str, 'utf8')
    : str.length
    console.log('"' + len.toString(16) + '-' + hash + '"')
}
```

第二种情况我们使用了 node 中的 crypto 加密模块生成加密 Hash，处理流程分四步：

1）`crypto.createHash()` 方法创建 Hash 实例，不能使用 new 关键字创建

2）`hash.update(data[,****inputEncoding****])`  指定要更新的内容，并指定编码方式为 utf8

3)`hash.digest()` 生成摘要

4）`str.substring(）`获取摘要的子集

摘要：将长度不固定的消息作为输入，通过运行 hash 函数，生成固定长度的输出，这段输出就叫做摘要
![image](https://user-images.githubusercontent.com/51777605/174844906-a0c4c7ff-1338-4c3f-a691-2e1584f4ab01.png)

## Etag 交互流程
![image](https://user-images.githubusercontent.com/51777605/174845238-7735ba21-161d-4c0e-8430-86dd16ed1268.png)

Q：在 koa 模块中我们会去判断资源是否新鲜，所有请求类型都需要进行判断吗？

A：只有幂等请求需要判断资源是否新鲜，比如 GET、HEAD 请求

Q：koa 模块中需要状态码为20x或304才进行新鲜度检查？

A：状态码20X可以证明请求是成功了，我们需要对成功的请求进行新鲜度判断是肯定的，那为什么304也需要做一层判断呢？304就表示资源新鲜了吗？koa 是中间件的思想，假如我们有一个中间件A，通过 Last-Modify 判断内容新鲜，将状态码置为304；又存在中间件 B，根据 Etag 判断资源并不新鲜，我们无法控制使用者中间件的调用顺序，所以需要对304的情况也做兼容处理。

## Etag 实战

1. 请求静态资源

```jsx
const conditional = require('koa-conditional-get');
const etag = require('koa-etag');
const Koa = require('koa');
const static = require('koa-static');
const path = require('path');
const app = new Koa();

app.use(conditional());
app.use(etag());

const staticPath = './static'
app.use(static(
    path.join(__dirname, staticPath)
));
app.listen(3000);
```

1. 字符串、Buffer、引用类型

```jsx
const conditional = require('koa-conditional-get');
const etag = require('koa-etag');
const Koa = require('koa');
const app = new Koa();

app.use(conditional());
app.use(etag());

app.use(async function (ctx, next){
    await next()
    ctx.body = {name: 'qianxun'}
})
app.listen(3000);
```
