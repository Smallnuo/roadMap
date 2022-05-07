缓存是提升 Web 性能的重要手段，前端缓存可以看作是 HTTP 缓存和浏览器缓存的结合，本文将结合简单的 Nodejs 例子深入理解 HTTP 缓存的作用。

![image](https://user-images.githubusercontent.com/51777605/167260206-43f1342b-d86c-407d-84c3-389b1a614581.png)

# 分级缓存策略

![image](https://user-images.githubusercontent.com/51777605/167260256-f3431c18-9382-458f-a1c3-e2d5b33391c7.png)

# HTTP缓存控制

HTTP 缓存策略分为强缓存和协商缓存两种，在 HTTP 中设置对应首部字段进行控制。强缓存由 Expires 或 Cache-Control 控制，协商缓存基于 Last-Modified 或 Etag 控制。

## Expires

HTTP 首部字段分为以下四种类型：

- 通用首部字段（请求报文和响应报文都会使用的首部）
- 请求首部字段（请求报文使用的首部）
- 响应首部字段（响应报文使用的首部）
- 实体首部字段（针对报文实体部分使用的首部）

Expires 是 HTTP/1.0 的产物，属于实体首部字段。Expires 使用绝对时间标识该资源的过期时间，修改本地时间可能造成缓存失效。Cache-Control 优先级高于 Expires，设置 Cache-Control 会导致 Expires 被忽略。

```jsx
if(request.url === '/expires.js'){
  deadline = new Date(Date.now() + 100000).toGMTString();
  response.writeHead(200, {
      'Content-Type': 'text/javascript',
      'Expires': deadline,
  });

  response.end("console.log('qianxun load !!!')");
}
```

![image](https://user-images.githubusercontent.com/51777605/167260293-25aced7a-4b0d-4ed6-acb8-c923e33a87f6.png)

![image](https://user-images.githubusercontent.com/51777605/167260315-48f91a25-fef6-4a6e-b167-682f1bcd2768.png)

## Cache-Control

Cache-Control 是 HTTP/1.1 的产物，属于通用首部字段，可以在请求头或响应头中设置，优先级高于 Expires。Cache-Control 有很多参数，常用参数如下。

### 可缓存性

public: 响应内容可被客户端、代理服务器等任意节点缓存

private: 客户端可缓存，代理服务器不可缓存，设置后 s-maxage 失效

no-store: 不使用任何缓存

no-cache: 请求首部表示不直接使用缓存，向服务器发起请求；响应首部时客户端可以缓存资源，需要通过协商缓存向服务器确认有效性

must-revalidate: 如果缓存不过期可以使用，过期了需要向服务器验证有效性。与 no-cache 的区别是 must-revalidate 是过期了才去验证，no-cache 是每次请求时都需要进行校验。

### 过期时间

max-age: 单位秒(s)，优先级高于 Expires，使用相对时间设置缓存到多少秒后过期

### Webpack为什么新增Hash值？

```jsx
const http = require('http');
const fs = require('fs');
const port = 3010;

http.createServer((request, response) => {
    console.log('request:', request.url);

    if(request.url === '/expires.html' || request.url === '/favicon.ico'){
        const html = fs.readFileSync('expires.html', 'utf-8');
        response.writeHead(200, {
            'Content-Type': 'text/html'
        })
        response.end(html);
    }else if(request.url === '/script.js'){
        response.writeHead(200, {
            'Content-Type': 'text/javascript',
            'Cache-Control': 'max-age=200'
        });
				// 修改 response.end("console.log('script load !!!')");
        response.end("console.log('qianxun load !!!')");
    }
}).listen(port, '127.0.0.1');
```

设置 'Cache-Control': 'max-age=200’ 进行浏览器缓存，第一次运行正常响应，修改服务器文件返回值并重启服务，再次获取内容从浏览器内存中获取，并未获取最新修改内容，这是为什么呢？

![image](https://user-images.githubusercontent.com/51777605/167260342-aa3c8cda-c560-4f8e-b45a-0aa028970a41.png)

由于我们请求的 url 并没有发生变化，按照浏览器分级策略我们会先在浏览器缓存中寻找，寻找成功后直接返回资源，并未向服务器发起真正的请求，查看请求图发现并未请求/script.js。这显然是个问题，我们如何解决这个问题呢？webpack 在打包时会给资源文件加上一串 hash 码，内容改变时 hash 值会发生变化，反映到页面上就是文件 url 发生了变化，这样就可以达到更新缓存到目的了。

![image](https://user-images.githubusercontent.com/51777605/167260362-e01f80be-d496-4e82-ab44-ff64e372fe39.png)

### 启发式缓存(Last-Modified/If-Modified-Since)

```jsx
if(request.url === '/last-modified.js'){
        const filePath = path.join(__dirname, request.url); // 拼接当前脚本文件地址
        const stat = fs.statSync(filePath); // 获取当前脚本状态
        const mtime = stat.mtime.toGMTString() // 文件的最后修改时间
        const requestMtime = request.headers['if-modified-since']; // 来自浏览器传递的值

        console.log(stat);
        console.log(mtime, requestMtime);

        // 走协商缓存
        if (mtime === requestMtime) {
            response.statusCode = 304;
            response.end();
            return;
        }

        // 协商缓存失效，重新读取数据设置 Last-Modified 响应头
        console.log('协商缓存 Last-Modified 失效');
        response.writeHead(200, {
            'Content-Type': 'text/javascript',
            'Last-Modified': mtime,
        });

        const readStream = fs.createReadStream(filePath);
        readStream.pipe(response);
    }
```

执行上述脚本后发现发现，请求并没有执行到服务端，而是使用了强缓存，为什么会这样呢？这是因为浏览器默认启用了一个启发式缓存。如果一个请求没有设置 Expires 和 Cache-Control，但是响应头有设置 Last-Modified 信息，这种情况下浏览器会触发启发式缓存，它的一个缓存时间是用 Date - Last-Modified 的值的 10% 作为缓存时间。
![image](https://user-images.githubusercontent.com/51777605/167260375-9ed79e33-36cf-4ae1-97aa-ce77c1fb2967.png)

我们要达到 304 的效果，不走强缓存直接走协商缓存，修改我们的响应，设置 Cache-Control：max-age=0 ，修改后再次执行结果如下：

```jsx
response.writeHead(200, {
    'Content-Type': 'text/javascript',
    'Last-Modified': mtime,
    //'Cache-Control': 'max-age=0', // 忽略启发式缓存
});
```
![image](https://user-images.githubusercontent.com/51777605/167260398-03986f49-3eed-40f2-bf1c-945298877788.png)

Tip: 除了启发式缓存外，针对内联 base64 图片会直接缓存在内存中，如果 base64 图片较大也会导致网页加载慢，可以使用 webpack loader，根据文件大小决定要不要做成内联 base64。

### Etag 和 If-None-Match

在了解 Etag 之前我们先考虑一下 Last-Modified 有什么弊端。

1. Last-Modified 根据文件修改时间来判断，单位是秒。加入我们操作是毫秒级别，Last-Modified 颗粒度就不满足我们现在的诉求。
2. 服务器资源确实被修改了，但实质的内容并没有发生变化，这个时候 Last-Modified 也会更新，但这种情况我们并不希望浏览器去重新加载资源。在这些特殊场景下，我们就需要使用 Etag。Etag 优先级高于 Last-Modified。

Etag 是根据文件内容是否修改来进行判断，通过 Hash 算法生成对应的 Etag 值。如果你对 Etag 想有更深的理解，可以阅读本系列中的 Etag 文章。

有了上面知识储备后，我们梳理整个 HTTP 缓存流程如下：
![image](https://user-images.githubusercontent.com/51777605/167260426-eafadb90-18b8-413d-9e9c-05094ffb757c.png)

### HTTP 代理服务

代理其实就是一台服务器，这台服务器本身不生产内容，提供“代理服务”。

代理的种类

匿名代理：完全“隐匿”了被代理的机器，外界看到的只是代理服务器

透明代理：不修改请求和响应的代理

正向代理：靠近客户端，代表客户端向服务器发送请求（VPN）
![image](https://user-images.githubusercontent.com/51777605/167260445-4df0433c-ebae-4824-a204-8e85863b2391.png)
反向代理：靠近服务器端，代表服务器响应客户端的请求(Nginx、CDN)
![image](https://user-images.githubusercontent.com/51777605/167260591-2b7d3af6-e9ab-480a-88d3-7fd870a595a4.png)
代理的作用
1. 负载均衡：合理分散外部流量到各个服务器，提高资源利用率和性能
2. 健康检查：使用“心跳机制”检查服务器是否异常，发生异常踢出集群，提高稳定性
3. 内容缓存：暂存、复用服务器响应

### 缓存代理服务

s-maxage: 使用公共服务器缓存，设置后忽略 Expires、max-age 指令

proxy-revalidate: 与 must-revalidate 相似，区别是代理服务器回源服务器校验，客户端不必回源服务器校验

no-transform: 禁止对资源做优化处理

only-if-cached：只接受代理服务器缓存，不接受源服务器响应。如果代理上没有缓存或者缓存过期，给客户端返回504（Gateway Timeout）

源服务器缓存控制

![image](https://user-images.githubusercontent.com/51777605/167260616-8e9943e0-a471-4077-894a-bee42c470490.png)
客户端缓存控制

![image](https://user-images.githubusercontent.com/51777605/167260630-297160e9-06e5-4b9c-a6e4-7a0aeb1a537c.png)
