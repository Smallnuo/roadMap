## 为什么需要 Etag ？

Etag 处理 Last-Modified 无法处理的场景。

1. Last-Modified 根据文件修改时间来判断，单位是秒。假如我们操作是毫秒级别，Last-Modified 颗粒度就不满足我们现在的诉求。
2. 服务器资源确实被修改了，但实质的内容并没有发生变化，这个时候 Last-Modified 也会更新，但这种情况我们并不希望浏览器去重新加载资源。在这些特殊场景下，我们就需要使用 Etag。Etag 优先级高于 Last-Modified。

## Etag 交互过程

![image](https://user-images.githubusercontent.com/51777605/168314979-9c49b2d3-5a2e-488e-9ab1-bd8000811d1a.png)

## Etag 实战

我们先来认识一些可能返回的响应实体：

- 静态资源：html，css，javaScript ，图片等
- 字符串
- Buffer
- 引用类型
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
![image](https://user-images.githubusercontent.com/51777605/168315104-8c504a31-c97b-4bf3-8a63-4d11f7433faf.png)

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

## Etag 值如何生成

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

1. 字符串，Buffer 和引用类型获取内容的部分 hash 值和内容长度

```jsx
const crypto = require('crypto')
const str = JSON.stringify({
    name:'qianxun'
})

//str.length 主要是针对 Buffer 缓冲区
if(!str.length){//缓冲区中无数据时，进入该条件
	console.log('"0-2jmj7l5rSw0yVb/vlWAYkK/YBwk"')
}else
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

## 强校验与弱校验

利用 koa-etag 生成 Etag 时我们可以配置 weak 说明返回 Etag 值是否加上 W/

```jsx
app.use(etag({
    weak: true
}));
```

主要策略是：

如果配置存在，并且 weak 字段是布尔类型，那么根据 weak 实际配置的布尔值来决定；若是不配置，那么会根据当前响应的资源是否为静态资源，如果是静态资源采取弱校验，否则采用强校验。

```jsx
var weak = options && typeof options.weak === 'boolean'
    ? options.weak
    : isStats
```

我们可以来验证一下不配置时：

- 静态资源：为弱校验
  ![image](https://user-images.githubusercontent.com/51777605/168315388-1ab49ec6-3c5c-4179-8fc8-7bf627115c2c.png)

- 字符串等：为强校验
  ![image](https://user-images.githubusercontent.com/51777605/168315511-3bb331e8-009e-4e35-bd04-6b0a04d89cfc.png)

## Etag 处理流程分析

```jsx
const Koa = require('koa')
const etag = require('koa-etag')
const conditional = require('koa-conditional-get')

const app = new Koa()
app.use(conditional())
app.use(etag());

app.use(async function(ctx, next){

    await next();
  
    ctx.body = 'hello';
  
  });

app.listen(3000,() => {
    console.log('ok')
})
```

我们拿上面的代码为例：
![image](https://user-images.githubusercontent.com/51777605/168315613-210cb1b1-9f15-4a1b-ba5f-f8befa67b1cb.png)

- 首先使用 conditional 中间件，该中间件一开始调用了 next 方法，进入下一个中间件
- 接下来进入 etag 中间件，该中间件一开始调用了 next 方法，也进入下一个中间件
- 现在执行到，我们的业务代码，执行了 next(),执行完之后没有下一个中间件了，接下来执行，ctx.body = ‘hello’
- 回到 etag 中间件，对 body 进行处理

```js
async function etag(){
  /*...*/
  // 获取到的 entity 为字符串 
  const entity = await getResponseEntity(ctx)
  // 使用这个字符串设置etag
  setEtag(ctx, entity, options)
}

// getResponseEntity 因为我们的 body 为 string，不处理直接将 body 返回
if (body instanceof Stream) {
    if (!body.path) return
    return await stat(body.path)
  } else if ((typeof body === 'string') || Buffer.isBuffer(body)) {
    return body
  } else {
    return JSON.stringify(body)
  }
```

设置 etag

```jsx
function setEtag (ctx, entity, options) {
  if (!entity) return
  // 使用我们设置在 body 上的字符串进行 etag 计算，并且设置在响应头上
  ctx.response.etag = calculate(entity, options)
}
```

计算 etag

```jsx
// 看看 body 是否为静态资源，如果为静态资源，那么调用 stattag，
// 我们设置的是字符串，因此调用 entitytag 方法
var tag = isStats
    ? stattag(entity)
    : entitytag(entity)

//使用 crypto 的一系列加密算法算出一个 hash 值并截取其中的一部分字符 
// 并且获取字符串的字节长度，二者并且生成一个 etag 值
function entitytag (entity) {
  if (entity.length === 0) {
    // fast-path empty
    return '"0-2jmj7l5rSw0yVb/vlWAYkK/YBwk"'
  }

  // compute hash of entity
  var hash = crypto
    .createHash('sha1')
    .update(entity, 'utf8')
    .digest('base64')
    .substring(0, 27)
  console.log(typeof entity);
  console.log(entity.length);
  // compute length of entity
  var len = typeof entity === 'string'
    ? Buffer.byteLength(entity, 'utf8')
    : entity.length

  return '"' + len.toString(16) + '-' + hash + '"'
}
```

- etag 中间件处理完，回到 conditional 中间件,看一下缓存是否新鲜来判断返回值，完成流程

```jsx
if (ctx.fresh) {
    ctx.status = 304
    ctx.body = null
}
```

## 新鲜度判断

我们经过上面的流程之后，有一个东西需要澄清，就是 conditional 中间中的 fresh 字段是如何处理的。因为这是个条件请求的中间件，我们可以去 koa 的 lib 目录的 request.js 中搜索 fresh 关键字。

```jsx
get fresh() {
    const method = this.method;
    const s = this.ctx.status;

    // 只有 get 或者 head 方法会被缓存
    if ('GET' !== method && 'HEAD' !== method) return false;
    
    // 只有 [200,300) 区间或者 304 有被缓存的可能
    if ((s >= 200 && s < 300) || 304 === s) {
      return fresh(this.header, this.response.header);
    }

    return false;
  },
```

我们再继续看 fresh 方法的具体逻辑：

```jsx
function fresh (reqHeaders, resHeaders) {
  // fields
  var modifiedSince = reqHeaders['if-modified-since']
  var noneMatch = reqHeaders['if-none-match']

  // 如果不是条件请求，不能被缓存
  if (!modifiedSince && !noneMatch) {
    return false
  }

  // 如果请求头是 cache-control:no-cache,不会缓存，下面会验证
  var cacheControl = reqHeaders['cache-control']
  if (cacheControl && CACHE_CONTROL_NO_CACHE_REGEXP.test(cacheControl)) {
    return false
  }

  // 如果是 if-none-match
  if (noneMatch && noneMatch !== '*') {
    var etag = resHeaders['etag']

    if (!etag) {
      return false
    }

    var etagStale = true //一开始假设缓存是不新鲜的
    var matches = parseTokenList(noneMatch)
    for (var i = 0; i < matches.length; i++) {
      var match = matches[i]
      //重点：请求头中的 match 和响应头中的 etag 匹配
      if (match === etag || match === 'W/' + etag || 'W/' + match === etag) {
        etagStale = false // 标记为新鲜
        break
      }
    }

    if (etagStale) {//如果上面遍历完，还是不新鲜的 返回不新鲜
      return false
    }
  }

  // 如果是 if-modified-since
  if (modifiedSince) {
    var lastModified = resHeaders['last-modified']
    // last-modified 响应头字段不存在 或者 响应头的时间戳大于等于请求头时间时（表示文件更改时间已经更新）
    // 上面的条件满足时表示不新鲜
    var modifiedStale = !lastModified || !(parseHttpDate(lastModified) <= parseHttpDate(modifiedSince))

    if (modifiedStale) {//要是不新鲜了，返回不新鲜
      return false
    }
  }
  // 上述条件都不满足时，表示时新鲜的，返回新鲜，即 ctx.fresh 为 true
  return true
}
```
