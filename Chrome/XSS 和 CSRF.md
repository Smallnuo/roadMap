## Cookie

Cookie 应用的三个方面：

1. 会话状态管理（用户登录状态、购物车或需要记录的信息）
2. 个性化设置（用户自定义主题）
3. 浏览器行为跟踪（跟踪分析用户行为）

SameSite 属性值

在理解 SameSite 各个属性的概念之前，我们需要了解[同源与同站](https://web.dev/i18n/zh/same-site-same-origin/)是什么。

同源：协议 + Host + 端口

同站：协议 + (eTLD + 1)

[eTLD](https://publicsuffix.org/list/)：有效的TLD（顶级域名）

```text
// 跨域跨站
www.taobao.com 和 [www.baidu.com](http://www.baidu.com/)
// 跨域同站
www.a.taobao.com 和 [www.b.taobao.com](http://www.b.taobao.com/)
// 跨站
a.github.io 和 b.github.io
```

SameSite ：显示说明 Cookie 在跨站请求时会不会被发送

Chrome80 之前 SameSite 默认是 None，所以淘系很多业务都采用跨站共享 Cookie，Chrome80 将 SameSite 默认值修改为 Lax，这样有效的避免 CSRF 攻击，但对于之前很多跨站共享 Cookie 的业务造成了重大影响。如果我们想继续跨站共享 Cookie，可以将 SameSite 属性改为 None，但于此同时也需要使用 https 来传输协议，也就是 Cookie 增加 Secure。

```jsx
Set-Cookie: promo_shown=1; SameSite=None;Secure
```

![image](https://user-images.githubusercontent.com/51777605/166679976-5f0d0346-ff38-4dfd-bafd-3e0ff413efef.png)

Priority：设置 Cookie 优先级，有 Low、Medium、High 3种取值

目前 Chrome 对 Cookie 的删除管理如下：

1. 优先级为 Low 的非 secure Cookie
2. 优先级为 Low 的 secure Cookie
3. 优先级为 Medium 的非 secure Cookie
4. 优先级为 Medium 的 secure Cookie
5. 优先级为 High 的非 secure Cookie
6. 优先级为 High 的 secure Cookie

## 什么是 XSS

XSS（Cross-Site Scripting） 是一种代码注入攻击，黑客通过巧妙的方法注入**恶意指令代码到网页**, 恶意代码与正常代码混在一起，浏览器无法分辨脚本可行性，**恶意脚本被执行**。黑客注入恶意脚本的目的是什么呢？

- 窃取 Cookie 信息：document.cookie 获取 Cookie 信息后，在其他电脑上模拟用户登录，然后进行转账等行为；
- 监听用户行为：使用 addEventListener 监听用户键盘行为，获取用户账号密码信息；
- 修改 DOM 伪造登录窗口，欺骗用户输入用户名和密码；

通过上面几个例子的介绍，我们可以了解到如果页面中插入了恶意脚本，等同于在黑客面前没有秘密，黑客可以利用所掌握的信息做很多意想不到的事情。

## XSS分类

### 存储型 XSS

![image](https://user-images.githubusercontent.com/51777605/166680113-cd4178d8-b961-4aac-87bd-c88dc235822f.png)

## 反射型 XSS 攻击

![image](https://user-images.githubusercontent.com/51777605/166680161-8c015aa6-3883-400c-bdad-50ca3f040fbb.png)

### 基于 DOM 的 XSS 攻击

![image](https://user-images.githubusercontent.com/51777605/166680221-e9c33555-c1a6-4c64-9924-3e1af2016a6b.png)

**存储型 XSS 和 反射型 XSS 都需要与服务器通信，属于服务端安全攻击，基于 DOM XSS 攻击不牵涉服务器，属于前端 JavaScript 自身安全漏洞。需要注意的是反射型 XSS 攻击不会被服务器存储，但存储型 XSS 攻击会被服务器存储**。

## 如何阻止 XSS 攻击

1. 服务器对输入内容进行过滤或转码
2. DOM 结构与数据分离， DOM结构维护在客户端
3. 转义 HTML
4. Cookie 设置 HttpOnly 属性: 禁止 JavaScript 读取某些敏感 Cookie，攻击者完成 XSS 注入后也无法窃取此 Cookie
5. 充分利用 CSP

> CSP（Content Security Policy）是一种内容安全策略，通过配置可以告诉浏览器那些内容可以被加载执行，而那些又不可以。如果你想了解更多关于 CSP 的知识，可以查看以下链接：
>

[内容安全策略( CSP ) - HTTP | MDN](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/CSP)

1. React 防范 XSS 攻击
- 尽可能使用 JSX 呈现页面内容，React 在背后为我们做了一些处理，防止 XSS 攻击；
- 勿滥用 Refs，避免借助 Refs 做一些危险的 DOM 操作；
- 使用 dangerouslySetInnerHTML API 时，确保渲染前先对数据进行处理，可以通过工具函数实现恶意脚本过滤。

## 什么是CSRF

CSRF(Cross-Site request forgery)又称“跨站请求伪造”。黑客引诱用户打开黑客网站，在网站中利用**用户已登录成功状态向被攻击网站**发起跨站请求。上面的说法还是比较晦涩难懂的，我们以 David 为例进行了解。

![image](https://user-images.githubusercontent.com/51777605/166680361-6fd3b904-c529-416e-94e7-5ed9d91961a9.png)

## CSRF 攻击类型

### GET 类型的 CSRF

浏览器自动发起获取图片资源时，如果浏览器未做判断会认为这是一个转账请求。

```html
<!DOCTYPE html>
<html>
  <body>
    <h1>黑客的站点：CSRF攻击演示</h1>
    <img src="https://time.geekbang.org/sendcoin?user=hacker&number=100">
  </body>
</html>
```

### POST 类型的 CSRF

构建一个自动提交转账请求的隐藏表单，当用户打开该站点时自动执行表单提交转账请求。

```html
<!DOCTYPE html>
<html>
<body>
  <h1>黑客的站点：CSRF攻击演示</h1>
  <form id='hacker-form' action="https://time.geekbang.org/sendcoin" method=POST>
    <input type="hidden" name="user" value="hacker" />
    <input type="hidden" name="number" value="100" />
  </form>
  <script> document.getElementById('hacker-form').submit(); </script>
</body>
</html>
```

### 链接类型的 CSRF

引诱用户点击图片下方的链接，发起请求。

```html
<div>
  <img width=150 src=http://images.xuejuzi.cn/1612/1_161230185104_1.jpg> </img> </div> <div>
  <a href="https://time.geekbang.org/sendcoin?user=hacker&number=100" taget="_blank">
    点击下载美女照片
  </a>
</div>
```

## 如何阻止 CSRF 攻击

1. Origin/ Referer 属性验证来源站点

CSRF 攻击来自第三方站点，我们可以通过验证来源站点辨别是否是正常的请求。两者都表示来源站点，区别是 Origin 属性只包含域名信息，不包含具体 URL 路径。一般情况下推荐使用 Origin 进行处理。

```text
// Referer
http://dev.insight.dtstack.cn/aiworks/model/modelList

// Origin
http://dev.insight.dtstack.cn
```

2. Token

**Token 是什么？**

Token 我们也称之为令牌，用于向服务端确认身份。Token 由3部分组成：

- header：指定了签名算法
- payload：可以指定用户 id，过期时间等非敏感数据
- Signature: 签名，服务端根据 header 知道它该用哪种签名算法，再用密钥根据此签名算法对 head + payload 生成签名，这样一个 token 就生成了。

![image](https://user-images.githubusercontent.com/51777605/166680548-59108ea6-91c4-4bda-8e03-ec455cad25a1.png)

**Access Token(身份认证)**

- 访问资源接口（API）时所需要的资源凭证
- 简单 token 的组成： uid(用户唯一的身份标识)、time(当前时间的时间戳)、sign（签名）

认证流程：

1. 用户通过用户名和密码发送请求；
2. 服务器端程序验证；
3. 服务器端程序返回一个**带签名**的`token` 给客户端；
4. 客户端储存`token`，比如放在 cookie 里或者 localStorage 里，每次访问`API`都携带`Token`到服务器端；
5. 服务端验证`token`，校验成功则返回请求数据，校验失败则返回错误码。

![image](https://user-images.githubusercontent.com/51777605/166680634-13491fc3-c771-4dd9-ada8-29b0adc7628c.png)

**CSRF Token(预防 CSRF 攻击)**

上述流程中解决了 Access Token 过期更新的问题，refresh Token 过期会如何处理呢？refresh Token 一般时间较长，refresh Token 过期后让用户再次登录认证就好啦。

在介绍 CSRF Token 之前，我们先来看一个说法，**Token 可以避免 CSRF 攻击？**Token 是如何做到避免 CSRF 攻击的呢？一般我们会将服务端返回的 Token 存放在 Cookie 中，但这里的 Cookie 仅是存储但概念，Token 一般会在请求头中添加 Authorization ****中，格式如下：

```jsx
'Authorization': 'Bearer ' + ${token}
```

上诉方式就避开了浏览器自动将 Cookie 添加这一特性，攻击者就无法冒用用户身份发起请求了。但 Token 存储在 Cookie 中是并不安全的，攻击者可以结合 XSS 攻击获取到用户的 Cookie。

CSRF Token 认证流程与 Access Token 一致，不同的是 Token 不存放在 Cookie 中，存放在服务器的 Session 中。我们在发送请求时会将对应的 Token 添加上发送给服务端。

- 静态 HTML 结构

遍历整个 DOM 树，对 DOM 中的 a 标签、form 标签加入 CSRF Token

- 动态 HTML 结构

    ```text
    // GET
    http://url?csrftoken=tokenvalue
    
    // POST
    // 在表单最后加上获取 CSRF Token 值
    <input type=”hidden” name=”csrftoken” value=”tokenvalue”/>
    ```


****Refresh Token（刷新Access Token）****

Token 一般情况下我们都会设置有效期，但是多久的有效期合适呢？如果时间过短，频繁的用户登录操作体验也不是很好，所以我们利用 ****Refresh Token 来刷新 Access Token。****如果时间过长会降低 Token 的安全性，给黑客提供了解密的时间。

我们根据时序图看一下 Access Token 与 Refresh Token 之间的关系：

1）登录

![image](https://user-images.githubusercontent.com/51777605/166680749-91d12eba-815e-4cc5-9876-27f0bc8ff281.png)

2）业务请求

![image](https://user-images.githubusercontent.com/51777605/166680798-6e46abfb-ada8-4255-98c2-f5e04cd189b9.png)

3）Access Token 过期，刷新 Access Token

![image](https://user-images.githubusercontent.com/51777605/166680855-6d7e9c76-b2e8-4124-878a-c3f3d62b523e.png)

3. 利用 Cookie 的 SameSite 属性
