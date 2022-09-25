## 前言

>💡 为什么数据流管理重要？React 的核心思想为：UI=render(data)，data 就是所谓的数据，render 是 React 提供的纯函数，所以 UI 展示完全由数据层决定。

在本文中，会简单介绍 react 中的数据流管理，从自身的 context 到三方库的 redux 的相关概念，以及 redux 附属内容丐版实现。

在正文之前，先简单介绍**数据**和**状态**的概念。React 是利用可复用的组件来构建界面，组件本质上有限状态机，能够记住当前组件的状态，根据不同的状态变化做出相关的操作。在 React 中，把这种状态定义为`state`。通过管理状态来实现对组件的管理，当`state`发生改变时，React 会自动去执行相应的操作。

而数据，它不仅指 server 层返回给前端的数据，React 中的状态也是一种数据。当数据改变时，我们需要改变状态去引发界面的变更。

## React 自身的数据流方案

### 基于 Props 的单向数据流

React 是自上而下的单向数据流，容器组件&展示组件是最常见的 React 组件设计方案。容器组件负责处理复杂的业务逻辑和数据，展示组件负责处理 UI 层。通常我们会把展示组件抽出来复用或者组件库的封装，容器组件自身通过 state 来管理状态，setState 更新状态，从而更新 UI，通过 props 将自身的 state 传递给展示组件实现通信

<img width="770" alt="image" src="https://user-images.githubusercontent.com/38368040/155837019-dc9ea4b9-0cac-4c97-be7f-b6e9cd3f9a23.png">


对于简单的通信，基于`props`串联父子和兄弟组件是很灵活的。

但是对于嵌套深数据流组件，A→B→C→D→E，A的数据需要传递给 E 使用，那么我们需要在 B/C/D 的`props`都加上该数据，导致最为中间组件的 B/C/D 来说会引入一些不属于自己的属性

### 使用 Context API 维护全局状态

Context API是 React 官方提供的一种组件树全局通信方式

`Context`基于生产者-消费者模式，对应 React 中的三个概念: **React.createContext**、**Provider**、**Consumer**。通过调用`createContext`创建出一组`Provider`。`Provider`作为数据的提供方，可以将数据下发给自身组件树中的任意层级的`Consumer`，而 **Consumer 不仅能够读取到 Provider 下发的数据还能读取到这些数据后续的更新值**

```jsx
const defaultValue = {
    count: 0,
    increment: () => {}
};

const ValueContext = React.createContext(defaultValue);

<ValueContext.Provider value={this.state.contextState}>
    <div className="App">
        <div>Count: {count}</div>
        <ButtonContainer />
        <ValueContainer />
    </div>
</ValueContext.Provider>

<ValueContext.Consumer>
    {({ increment }) => (
        <button onClick={increment} className="button">increment</button>
    )}
</ValueContext.Consumer>
```

[16.3之前的用法](https://codesandbox.io/s/context-use-before-16-3-318qr2?file=/src/App.js)，[16.3之后的 createContext 用法](https://codesandbox.io/s/context-use-after-16-3-j566ro?file=/src/App.js:581-643)，[useContext 用法](https://codesandbox.io/s/context-use-hooks-2l55gw?file=/src/App.js)

Context工作流的简单图解：

<img width="770" alt="image" src="https://user-images.githubusercontent.com/38368040/189881707-b630ef3d-3be5-404b-9098-ebeb8d36a2d9.png">

在v16.3之前由于各种局限性不被推荐使用

-   代码不够简单优雅：生产者需要定义`childContextTypes`和`getChildContext`，消费者需要定义`ChildTypes`才能够访问`this.context`访问到生产者提供的数据
-   数据无法及时同步：类组件中可以使用`shouldComponentUpdate`返回 false 或者是`PureComponent`，后代组件都不会被更新，这违背了Context 模式的设置，导致生产者和消费者之间不能及时同步

在 v16.3 之后的版本中做了对应的调整，即使组件的`shouldComponentUpdate`返回 false，它仍然可以”穿透”组件继续向后代组件进行传播，更改了声明方式变得更加语义化，使得 Context 成为了一种可行的通信方案

但是 Context 也是通过一个容器组件来管理状态的，但是`Consumer`和`Provider`是一一对应的，在项目复杂度高的时候，可能会出现多个`Provider`和`Consumer`，甚至一个`Consumer`需要对应多个`Provider`的情况

当某个组件的业务逻辑变得非常复杂时，代码会越写越多，因为我们只能够在组件内部去控制数据流，这样导致 Model 和 View 都在 View 层，业务逻辑和UI实现都在一块，难以维护

所以这个时候需要真正的数据流管理工具，从 UI 层完全抽离出来，只负责管理数据，让 React 只专注于 View 层的绘制

## Redux

Redux 是 **JS 应用**的状态容器，提供可预测的状态管理

Redux 的三大原则

-   单一数据源：整个应用的 state 都存储在一棵树上，并且这棵状态树只存在于唯一的 store 中
-   state 是只读的：对 state 的修改只有触发 action
-   用纯函数执行修改：reducer 根据旧状态和传进来的 action 来生成一个新的 state (类似与 reduce 的思想，接受上一个 state 和当前项 action，计算出来一个新值)

Redux 工作流

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5995abe444484e359928b4cd64d19a87~tplv-k3u1fbpfcp-watermark.image?)

**不可变性(Immutability)**

mutable 意为可改变的，immutability 意为用不可改变的

在 JS 的对象(object)和数组(array)默认都是 mutable，创建一个对象/数组都是可以改变内容

```js
const obj = { name: 'FBB', age: 20 };
obj.name = 'shuangxu';

const arr = [1,2,3];
arr[1] = 6;
arr.push('change');
```

改变对象或者数组，内存中的引用地址尚未改变，但是内容已经改变

如果想用不可变的方式来更新，代码必须复制原来的对象/数组，更新它的复制体

```js
const obj = { info: { name: 'FBB', age: 20 }, phone: '177xxx' }
const cloneObj = { ...obj, info: { name: 'shuangxu' } }

//浅拷贝、深拷贝
```

**Redux期望所有的状态都采用不可变的方式。**

### react-redux

react-redux 是 Redux 提供的 react 绑定，辅助在 react 项目中使用 redux

它的 API 简单，包括一个组件`Provider`和一个高阶函数`connect`

#### Provider

❓为什么`Provider`只传递一个`store`，被它包裹的组件都能够访问到`store`的数据呢？

Provider 做了些啥？

-   创建一个`contextValue`包含`redux`传入的`store`和根据`store`创建出的`subscription`，发布订阅均为`subscription`做的
-   通过`context`上下文把`contextValue`传递子组件

#### Connect

❓connect 做了什么事情讷？

使用容器组件通过`context`提供的`store`，并将`mapStateToProps`和`mapDispatchToProps`返回的`state`和`dispatch`传递给UI组件

组件依赖 redux 的`state`，映射到容器组件的`props`中，`state`改变时触发容器组件的`props`的改变，触发容器组件组件更新视图

```js
const enhancer = connect(mapStateToProps, mapDispatchToProps)
enhancer(Component)
```
#### react-redux丐版实现

[mini-react-redux](https://codesandbox.io/s/mini-react-redux-x8j5t0?file=/src/App.js:141-149)

**Provider**

```jsx
export const Provider = (props) => {
    const { store, children, context } = props;
    const contextValue = { store };
    const Context = context || ReactReduxContext;
    return <Context.Provider value={contextValue}>{children}</Context.Provider>
};
```

**connect**

```tsx
import { useContext, useReducer } from "react";
import { ReactReduxContext } from "./ReactReduxContext";

export const connect = (mapStateToProps, mapDispatchToProps) => (
    WrappedComponent
) => (props) => {
    const { ...wrapperProps } = props;
    const context = useContext(ReactReduxContext);
    const { store } = context; // 解构出 store
    const state = store.getState(); // 拿到 state
    //使用 useReducer 得到一个强制更新函数
    const [, forceComponentUpdateDispatch] = useReducer((count) => count + 1, 0);
    // 订阅 state 的变化，当 state 变化的时候执行回调
    store.subscribe(() => {
        forceComponentUpdateDispatch();
    });
    // 执行 mapStateToProps 和 mapDispatchToProps
    const stateProps = mapStateToProps?.(state);
    const dispatchProps = mapDispatchToProps?.(store.dispatch);
    // 组装最终的 props
    const actualChildProps = Object.assign(
        {},
        stateProps,
        dispatchProps,
        wrapperProps
    );
    return <WrappedComponent {...actualChildProps} />;
};

```

### redux Middleware

> “It provides a third-party extension point between dispatching an action, and the moment it reaches the reducer.” – Dan Abramov

`middleware`提供分类处理`action`的机会，在`middleware`中可以检查每一个`action`，挑选出特定类型的`action`做对应操作

<img width="770" alt="image" src="https://user-images.githubusercontent.com/38368040/189882071-28d47ffd-0fdd-4dd8-bcbc-e5bf41e57129.png">

[middleware示例](https://codesandbox.io/s/middleware-wn4pn?file=/src/App.js)

打印日志

```jsx
store.dispatch = (action) => {
    console.log("this state", store.getState());
    console.log(action);
    next(action);
    console.log("next state", store.getState());
};
```

监控错误

```jsx
store.dispatch = (action) => {
    try {
        next(action);
    } catch (err) {
        console.log("catch---", err);
    }
};
```

二者合二为一

```jsx
store.dispatch = (action) => {
    try {
        console.log("this state", store.getState());
        console.log(action);
        next(action);
        console.log("next state", store.getState());
    } catch (err) {
        console.log("catch---", err);
    }
};
```

提取 loggerMiddleware/catchMiddleware

```jsx
const loggerMiddleware = (action) => {
    console.log("this state", store.getState());
    console.log("action", action);
    next(action);
    console.log("next state", store.getState());
};
const catchMiddleware = (action) => {
    try {
        loggerMiddleware(action);
    } catch (err) {
        console.error("错误报告: ", err);
    }
};
store.dispatch = catchMiddleware
```

catchMiddleware 中都写死了调用的 loggerMiddleware，loggerMiddleware 中写死了 next(store.dispatch)，需要灵活运用，让 middleware 接受 dispatch 参数

```jsx
const loggerMiddleware = (next) => (action) => {
    console.log("this state", store.getState());
    console.log("action", action);
    next(action);
    console.log("next state", store.getState());
};
const catchMiddleware = (next) => (action) => {
    try {
        /*loggerMiddleware(action);*/
        next(action);
    } catch (err) {
        console.error("错误报告: ", err);
    }
};
/*loggerMiddleware 变成参数传进去*/
store.dispatch = catchMiddleware(loggerMiddleware(next));
```

middleware 中接受一个 store，就能够把上面的方法提取到单独的函数文件中

```jsx
export const catchMiddleware = (store) => (next) => (action) => {
    try {
        next(action);
    } catch (err) {
        console.error("错误报告: ", err);
    }
};

export const loggerMiddleware = (store) => (next) => (action) => {
    console.log("this state", store.getState());
    console.log("action", action);
    next(action);
    console.log("next state", store.getState());
};

const logger = loggerMiddleware(store);
const exception = catchMiddleware(store);
store.dispatch = exception(logger(next));
```

每个 middleware 都需要接受 store 参数，继续优化这个调用函数

```js
export const applyMiddleware = (middlewares) => (oldCreateStore) => (reducer, initState) => {
    //获得老的store
    const store = oldCreateStore(reducer, initState);
    //[catch, logger]
    const chain = middlewares.map((middleware) => middleware(store));
    let oldDispatch = store.dispatch;
    chain
        .reverse()
        .forEach((middleware) => (oldDispatch = middleware(oldDispatch)));
    store.dispatch = oldDispatch;
    return store;
    };
};

const newStore = applyMiddleware([catchMiddleware, loggerMiddleware])(
    createStore
)(rootReducer);
```

Redux提供了`applyMiddleware`来加载`middleware`,`applyMiddleware`接受三个参数，`middlewares`数组/`redux`的`createStore`/`reducer`

```jsx
export default const applyMiddleware = (...middlewares) => createStore => (reducer, ...args) => {
  //由createStore和reducer创建store
    const store = createStore(reducer, ...args)
    let dispatch = store.dispatch
    var middlewareAPI = {
        getState: store.getState,
        dispatch: (action, ...args) => dispatch(action, ...args)
    }
    //把getState/dispatch传给middleware，
    //map让每个middleware获得了middlewareAPI参数
    //形成一个chain匿名函数数组[f1,f2,f3...fn]
    const chain = middlewares.map(middleware => middleware(middlewareAPI))
    //dispatch=f1(f2(f3(store.dispatch)))，把所有  的middleware串联起来
    dispatch = compose(...chain)(store.dispatch)
    return {
        ...store,
        dispatch
    }
}
```

applyMiddleware 符合洋葱模型

<img width="770" alt="image" src="https://user-images.githubusercontent.com/38368040/156110747-48070fcb-e85a-4d2f-b97a-7aa57a2a523c.png">

## 总结

本文意在讲解 react 的数据流管理。从 react 本身的提供的数据流方式出发

1.  基于`props`的单向数据流，串联父子和兄弟组件非常灵活，但是对于嵌套过深的组件，会使得中间组件都加上不需要的`props`数据
1.  使用 Context 维护全局状态，介绍了 v16.3 之前、v16.3 之后、hooks，不同版本`context`的使用，以及 v16.3 之前版本的`context`的弊端。
1.  引入 redux，第三方的状态容器，以及 react-redux API(Provider/connect)分析与丐版实现，最后介绍了 redux 强大的中间件是如何重写 dispatch 方法

> 参考连接

-   [对 React 状态管理的理解及方案对比](https://github.com/sunyongjian/blog/issues/36)
-   [聊一聊我对 React Context 的理解以及应用](https://juejin.cn/post/6844903566381940744)
-   [redux middleware 详解](https://zhuanlan.zhihu.com/p/20597452)
-   [手写 react-redux](http://dennisgo.cn/Articles/React/React-Redux.html)
