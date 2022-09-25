## 前端路由

在 Web 前端单页面应用 SPA(Single Page Application)中，路由是描述 URL 和 UI 之间的映射关系，这种映射是单向的，即 URL 的改变会引起 UI 更新，无需刷新页面

### 如何实现前端路由

实现前端路由，需要解决两个核心问题

1. 如何改变 URL 却不引起页面刷新？
2. 如何监测 URL 变化？

在前端路由的实现模式有两种模式，hash 和 history 模式，分别回答上述两个问题

#### hash 模式

1. hash 是 url 中 hash(#) 及后面的部分，常用锚点在页面内做导航，改变 url 中的 hash 部分不会引起页面的刷新
2. 通过 [hashchange](https://developer.mozilla.org/en-US/docs/Web/API/Window/hashchange_event) 事件监听 URL 的改变。改变 URL 的方式只有以下几种：通过浏览器导航栏的前进后退、通过`<a>`标签、通过`window.location`，这几种方式都会触发`hashchange`事件

<img width="735" alt="image" src="https://user-images.githubusercontent.com/38368040/166693975-8cd51c55-744d-4ffd-81f4-c39764181bb0.png">

#### history 模式

1. history 提供了 `pushState` 和 `replaceState` 两个方法，这两个方法改变 URL 的 path 部分不会引起页面刷新
2. 通过 [popstate ](https://developer.mozilla.org/en-US/docs/Web/API/Window/popstate_event)事件监听 URL 的改变。需要注意只在通过浏览器导航栏的前进后退改变 URL 时会触发`popstate`事件，通过`<a>`标签和`pushState`/`replaceState`不会触发`popstate`方法。但我们可以拦截`<a>`标签的点击事件和`pushState`/`replaceState`的调用来检测 URL 变化，也是可以达到监听 URL 的变化，相对`hashchange`显得略微复杂

<img width="763" alt="image" src="https://user-images.githubusercontent.com/38368040/166694070-a31bfbc6-cfce-4928-bf8b-7111e3e6181a.png">

### JS实现前端路由

#### 基于 hash 实现

由于三种改变hash的方式都会触发`hashchange`方法，所以只需要监听`hashchange`方法

```jsx
// 页面加载完不会触发 hashchange，这里主动触发一次 hashchange 事件，处理默认hash
window.addEventListener("DOMContentLoaded", onLoad);
// 监听路由变化
window.addEventListener("hashchange", onHashChange);
// 路由变化时，根据路由渲染对应 UI
function onHashChange() {
    switch (location.hash) {
        case "#/home":
            routerView.innerHTML = "This is Home";
            return;
        case "#/about":
            routerView.innerHTML = "This is About";
            return;
        case "#/list":
            routerView.innerHTML = "This is List";
            return;
        default:
            routerView.innerHTML = "Not Found";
            return;
    }
}
```

[hash实现demo](https://codesandbox.io/s/vanilla-hash-vpfcbq?file=/index.html)

#### 基于 history 实现

因为 history 模式下，`<a>`标签和`pushState`/`replaceState`不会触发`popstate`方法，我们需要对`<a>`的跳转和`pushState`/`replaceState`做特殊处理。

- 对`<a>`作点击事件，禁用默认行为，调用`pushState`方法并手动触发`popstate`的监听事件
- 对`pushState`/`replaceState`可以重写 history 的方法并通过派发事件能够监听对应事件

```jsx
var _wr = function (type) {
    var orig = history[type];
    return function () {
        var e = new Event(type);
        e.arguments = arguments;
        var rv = orig.apply(this, arguments);
        window.dispatchEvent(e);
        return rv;
    };
};
// 重写pushstate事件
history.pushState = _wr("pushstate");

function onLoad() {
    routerView = document.querySelector("#routeView");
    onPopState();
    // 拦截 <a> 标签点击事件默认行为
    // 点击时使用 pushState 修改 URL并更新手动 UI，从而实现点击链接更新 URL 和 UI 的效果。
    var linkList = document.querySelectorAll("a[href]");
    linkList.forEach((el) =>
        el.addEventListener("click", function (e) {
            e.preventDefault();
            history.pushState(null, "", el.getAttribute("href"));
            onPopState();
        })
    );
}
// 监听pushstate方法
window.addEventListener("pushstate", onPopState);
// 页面加载完不会触发 hashchange，这里主动触发一次 popstate 事件，处理默认pathname
window.addEventListener("DOMContentLoaded", onLoad);
// 监听路由变化
window.addEventListener("popstate", onPopState);
// 路由变化时，根据路由渲染对应 UI
function onPopState() {
    switch (location.pathname) {
        case "/home":
            routerView.innerHTML = "This is Home";
            return;
        case "/about":
            routerView.innerHTML = "This is About";
            return;
        case "/list":
            routerView.innerHTML = "This is List";
            return;
        default:
            routerView.innerHTML = "Not Found";
            return;
    }
}
```

[history 实现 demo](https://codesandbox.io/s/vanilla-history-o88eo7?file=/index.html)

## react-router的理解

![image](https://user-images.githubusercontent.com/38368040/155311375-9942ba4d-91c5-4320-a84d-1e6715ab1b43.png)

在 v4 之后，我们直接从`react-router-dom`中引入`BrowserRouter`/`HashRouter` 。`BrowserRouter`/`HashRouter`又分别使用了`react-router`提供的 Router 组件和 history 提供的`createBrowserHistory`/`createHashHistory`方法。

## react-router v3/v4/v6的应用

[v3](https://codesandbox.io/s/router-demo-v3-wqhjd)、[v4](https://codesandbox.io/s/router-demo-v4-7xj3f)、[v6](https://codesandbox.io/s/router-demo-v6-dpv1m)

<img width="903" alt="image" src="https://user-images.githubusercontent.com/38368040/166693234-9664b7a0-89cf-4d8c-ba10-30afae53c8bb.png">

## history

在上文中说到，`BrowserRouter`使用history库提供的`createBrowserHistory`创建的`history`对象改变路由状态和监听路由变化。

❓那么 history 对象需要提供哪些功能讷？

- 监听路由变化的`listen`方法以及对应的清理监听`unlisten`方法
- 改变路由的`push`方法

```jsx
// 创建和管理listeners的方法
export const EventEmitter = () => {
    const events = [];

    return {
        subscribe(fn) {
            events.push(fn);
            return function () {
                events = events.filter((handler) => handler !== fn);
            };
        },
        emit(arg) {
            events.forEach((fn) => fn && fn(arg));
        }
    };
};
```

### BrowserHistory

```jsx
const createBrowserHistory = () => {
    const EventBus = EventEmitter(); //{subscribe,emit}
    // 初始化location
    let location: ILocation = {
        pathname: window.location.pathname || "/"
    };
    // 路由变化时的回调
    const handlePop = function () {
        const currentLocation = {
            pathname: window.location.pathname
        };
        EventBus.emit(currentLocation); // 路由变化时执行回调
    };
    // 定义history.push方法
    const push = (path) => {
        const history = window.history;
        // 为了保持state栈的一致性
        history.pushState(null, "", path);
        // 由于push并不触发popstate，我们需要手动调用回调函数
        location = { pathname: path };
        EventBus.emit(location);
    };

    const listen = (listener) => EventBus.subscribe(listener);

    // 处理浏览器的前进后退
    window.addEventListener("popstate", handlePop);

    // 返回history
    const history: IHistory = {
        location,
        listen,
        push
    };
    return history;
};
```

上述代码实现简单版本的 history，只有监听路由变化的`listen`/`unlisten`方法以及改变路由的`push`方法，详细的[BrowserHistory源码](https://github.com/remix-run/history/blob/v4.9.0/modules/createBrowserHistory.js#L38)

### HashHistory

```jsx
const createHashHistory = () => {
    const EventBus = EventEmitter();
    const location: ILocation = {
        pathname: window.location.hash.slice(1) || "/"
    };

    // 路由变化时的回调
    const handlePop = () => {
        const currentLocation = {
            pathname: window.location.hash.slice(1)
        };
        EventBus.emit(currentLocation); // 路由变化时执行回调
    };

    // 不用手动执行回调，因为hash改变会触发hashchange事件
    const push = (path: string) => (window.location.hash = path);

    const listen = (listener: Function) => EventBus.subscribe(listener);

     // 监听hashchange事件
    window.addEventListener("hashchange", handlePop);

    // 返回的history上有个listen方法
    const history: IHistory = {
        location,
        listen,
        push
    };
    return history;
};
```

和`BrowserHistory`一样，`hashHistory`也是极简版，详细的[hashHistory源码](https://github.com/remix-run/history/blob/v4.9.0/modules/createHashHistory.js#L57)

## react-router丐版

<img width="1084" alt="image" src="https://user-images.githubusercontent.com/38368040/166694742-45312731-48f0-483b-a315-126355a64654.png">

### Router

Router 接受一个 history 属性，用`history.listen`创建监听者，使用 context 传递 history 和 location 数据

```jsx
export default class Router extends React.Component {
    constructor(props) {
        super(props);
        this.state = {
            location: props.history.location // 将history的location挂载到state上
        };
            // 之所以要在这里做监听是因为还没 didMount
        this.unlisten = props.history.listen((location) => {
            this.setState({ location });
        });
    }
    componentDidMount() {}
    componentWillUnmount() {
        this.unlisten();
    }
    render() {
        const { history, children } = this.props;
        const { location } = this.state;
        return (
            <RouterContext.Provider
                value={{ history, location }}
            >
                {children}
            </RouterContext.Provider>
        );
    }
}
```

[Router源码](https://github.com/remix-run/react-router/blob/v5/packages/react-router/modules/Router.js)

### BrowserRouter/HashRouter

只是给 Router 组件传递 history 属性

**BrowserRouter**

```jsx
class BrowserRouter extends React.Component {
    history = createBrowserHistory();
    render() {
        return <Router history={this.history} children={this.props.children} />;
    }
}
```

**HashRouter**

```jsx
class HashRouter extends React.Component {
    history = createHashHistory();
    render() {
        return <Router history={this.history} children={this.props.children} />;
    }
}
```

[BrowserRouter源码](https://github.com/remix-run/react-router/blob/v5/packages/react-router-dom/modules/BrowserRouter.js)/[HashRouter源码](https://github.com/remix-run/react-router/blob/v5/packages/react-router-dom/modules/HashRouter.js)

### Route

Route可以接收`component`/`render`/`children`，但是它们渲染的优先级是不一样的。

v4/v5三个优先级不同

直接使用`Route`组件时，每个`Route`组件都会被渲染，会根据路由规则进行判断是否需要把组件渲染出来，目前代码中使用的正则来做匹配

```jsx
export default class Route extends React.Component<IProps> {
    render() {
        return (
        <RouterContext.Consumer>
            {(context) => {
                const pathname = context.location.pathname;
                const {
                    path,
                    component: Component,
                    exact = false,
                    render,
                    children
                } = this.props;
                const props = { ...context };
                const reg = pathToRegExp(path, [], { end: exact });
                // 判断url是否匹配
                if (!reg.test(pathname)) return null;
                if (children) return children;
                if (Component) return <Component {...props} />;
                if (render) return render();
            }}
        </RouterContext.Consumer>
        );
    }
}
```

[Route源码](https://github.com/remix-run/react-router/blob/v5/packages/react-router/modules/Route.js)

### Link

在 Link 中，我们使用`<a>`标签来做跳转，但是 a 标签会使页面重新刷新，所以需要阻止 a 标签的默认行为，使用 context 中 history 的 push 方法

```jsx
export default class Link extends React.Component<IProps> {
    render() {
        const { to, children } = this.props;
        return (
            <RouterContext.Consumer>
                {(context) => {
                    return (
                        <a
                            href={to}
                            onClick={(event) => {
                                // 阻止a标签的默认行为
                                event.preventDefault();
                                context.history.push(to);
                            }}
                        >
                            {children}
                        </a>
                    );
                }}
            </RouterContext.Consumer>
        );
    }
}
```

[Link源码](https://github.com/remix-run/react-router/blob/v5/packages/react-router-dom/modules/Link.js#L75)

### Switch

Route 组件的功能是只要 path 匹配上当前路由就渲染组件，也就意味着如果多个 Route 的 path 都匹配上了当前路由，这几个组件都会渲染，例如`/home/1`能够匹配上`/home/1`和`/home`，所以需要一个组件来控制匹配上一个 Route 就返回，所以 Switch 组件诞生了。

它的功能就是即使多个 Route 的 path 都匹配上了当前路由，也只渲染第一个匹配上的组件。

要实现该功能，把 Switch 的 children 拿出来循环，找出第一个匹配的 child，记录下当前的 child ，把其他的 child 全部干掉

```jsx
export default class Switch extends React.Component {
    render() {
        return (
        <RouterContext.Consumer>
            {(context) => {
                const location = context.location;
                let element, match; // 两个变量记录第一次匹配上的子元素和match属性
                React.Children.forEach(this.props.children, (child) => {
                    // 先检测下match是否已经匹配到了, 如果已经匹配过了，直接跳过
                    if (!match && React.isValidElement(child)) {
                        element = child;
                        const { path, exact } = child.props;
                        const reg = pathToRegExp(path, [], { end: exact });
                        if (reg.test(location.pathname)) match = true;
                    }
                });
                // <Switch>组件的返回值只是匹配上元素的拷贝，其他元素被忽略了
                // 如果一个都没匹配上，返回null
                return match ? React.cloneElement(element, { location }) : null;
            }}
        </RouterContext.Consumer>
        );
    }
}
```

[Switch源码](https://github.com/remix-run/react-router/blob/v5/packages/react-router/modules/Switch.js#L12)

到现在 react-router 的核心组件以及 API 都实现完成，[线上demo](https://codesandbox.io/s/mini-react-router-0mukx7)

## 总结

在本文中，从前端路由入手，分析了原生的 hash/history 的路由实现，react-router 底层依赖和上层使用，实现了简版的 react-router

需要注意的是，hash 模式下三种改变 url 的方法都会触发 `hashchange` 事件，而 history 模式下只有浏览器前进后退会触发`popstate`，`pushState`/`replaceState`以及`<a>`标签都不会。`<a>`标签的默认行为会触发页面刷新，所以在实现路由时需要用`e.preventDefault`阻止默认行为。