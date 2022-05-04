阅读本文前，请先阅读前文[数据流前篇]( )

## 回顾以及引入问题

在之前的设计中，我们使用 useReducer 获得了一个强制更新函数(`forceComponentUpdateDispatch`)，然后在`store.subcribe`回调函数中执行

```jsx
export const connect = (mapStateToProps, mapDispatchToProps) => (
  WrappedComponent
) => (props) => {
  const { ...wrapperProps } = props;
  const context = useContext(ReactReduxContext);
  const { store } = context; // 解构出store
  const state = store.getState(); // 拿到state
  //使用useReducer得到一个强制更新函数
  const [, forceComponentUpdateDispatch] = useReducer((count) => count + 1, 0);
  // 订阅state的变化，当state变化的时候执行回调
  store.subscribe(() => {
    forceComponentUpdateDispatch();
  });
  // 执行mapStateToProps和mapDispatchToProps
  const stateProps = mapStateToProps?.(state);
  const dispatchProps = mapDispatchToProps?.(store.dispatch);
  // 组装最终的props
  const actualChildProps = Object.assign(
    {},
    stateProps,
    dispatchProps,
    wrapperProps
  );
  return <WrappedComponent {...actualChildProps} />;
};
```

上述代码已经实现了 store 中数据改变时，对应使用 connect 包裹的组件能够获得对应数据，但是存在一个更新顺序的问题。

![connect](https://user-images.githubusercontent.com/38368040/161436788-83925945-4c39-4257-9285-74675e8e6754.png)

在前文中，我们提及到 React 是单向数据流，props 都是父组件传递给子组件的。一旦我们引入了 redux 后，假设父子组件都会引用了同一个变量`count`，子组件根本不会从父组件拿该参数，而是直接从 redux 中读取，这使得 React 的原本**父→子**的单向数据流被打破了。

再说到更新问题，在 React 中，如果一个共同变量变化了，那必然是父组件先更新，再把数据传给子组件做更新。但是 redux 里，数据变成 **redux→父**，**redux→子**，父子组件完全根据 redux 的数据做独立更新，不能完全保证父组件先更新，子组件再更新。react-redux 为了保证更新顺序引入了一个监听者类`Subscription`

## Subscription类

`Subscription`需要做什么？[线上代码](https://codesandbox.io/s/upgrade-mini-react-redux-45h0df?file=/src/mini-react-redux/Subscription.js)

1. 实现发布订阅，处理所有state的回调
2. 需要判断当前连接 redux 的组件是否为第一个连接 redux 的组件，如果当前组件就是连接 redux的根组件，它`state`回调直接注册到 redux store；同时创建一个`Subscription`实例(`subscription`)并且通过`context`传递给子级
3. 如果当前组件不是根组件，说明已经有组件注册到了 redux store 了，那在子组件中可以拿到通过`context`传递的`subscription`(由于是父组件的监听类又称为`parentSub`)，那么当前子组件的回调会注册到`parentSub`上。并且会新建一个`Subscription`实例，在`context`上继续传递，那么当前组件的子组件回调会注册到当前组件的`Subscription`实例上
4. 当`state`变化了，根组件注册到 redux store 的回调会更新根组件，根组件会手动更新子组件的回调，子组件的回调执行更新子组件，子组件会执行`subscription`上注册的回调，触发孙子组件更新...这样子就实现了一层一层的组件更新，保证了**父→子**的更新顺序

```jsx
export class Subscription {
  constructor(store, parentSub) {
    this.store = store;
    this.parentSub = parentSub;
    this.listeners = [];
    this.handleChangeWrapper = this.handleChangeWrapper.bind(this);
  }
  //当前组件注册
  addNestedSub(listener) {
    this.listeners.push(listener);
  }
  //通知监听者
  notifyNestedSub() {
    this.listeners.forEach((listener) => listener());
  }

  // 回调函数的包装
  handleChangeWrapper() {
    if (this.onStateChange) {
      this.onStateChange();
    }
  }

  //注册回调函数
  //如果没有parentSub，说明是根组件注册到store上
  //如果有，就注册到父组件的监听类上
  trySubscribe() {
    this.parentSub
      ? this.parentSub.addNestedSub(this.handleChangeWrapper)
      : this.store.subscribe(this.handleChangeWrapper);
  }
}
```

[Subscription源码](https://github.com/reduxjs/react-redux/blob/v7.2.0/src/utils/Subscription.js#L74)

## 对应改造

### Provider

在我们使用 redux 的时候，`Provider`始终是我们的根组件，所以需要给`Provider`创建一个`Subscription`实例再通过`context`传递下去，[线上代码](https://codesandbox.io/s/upgrade-mini-react-redux-45h0df?file=/src/mini-react-redux/Provider.tsx)

```jsx
export const Provider = (props) => {
  const { store, children, context } = props;

  // 传给子组件的context{store,subscription}
  const contextValue = useMemo(() => {
    const subscription = new Subscription(store);
    // 注册回调函数，通知子组件
    subscription.onStateChange = subscription.notifyNestedSubs;
    return { store, subscription };
  }, [store]);

  const previousState = useMemo(() => store.getState(), [store]);

  useEffect(() => {
    const { subscription } = contextValue;
    // 添加监听者
    subscription.trySubscribe();
    // 如果state发生改变，通知监听者
    if (previousState !== store.getState()) {
      subscription.onStateChange();
    }
  }, [contextValue, previousState, store]);

  const Context = context || ReactReduxContext;
  return <Context.Provider value={contextValue}>{children}</Context.Provider>;
};
```

[Provider源码](https://github.com/reduxjs/react-redux/blob/v7.2.0/src/components/Provider.js#L6)

### Connect

在之前的版本中，connect 是直接注册到 store 上，那现在就应该注册在父级的`subscription`上，在自己更新完成之后，再去通知自己的子级做更新。

还有就是我们需要重写`context`中的`subscription`，因为当前组件拿到的`subscription`是属于它父级的，而当前组件的子级需要的`subscription`是当前组件创建的，我们需要重写`context`中的`subscription`，所以我们的`connect`返回的组件需要用`Context.Provider`包裹一下。[线上代码](https://codesandbox.io/s/upgrade-mini-react-redux-45h0df?file=/src/mini-react-redux/connect.tsx)

```jsx
export const connect = (mapStateToProps, mapDispatchToProps) => (
  WrappedComponent
) => (props) => {
  const { ...wrapperProps } = props;
  const context = useContext(ReactReduxContext);
  const { store, subscription: parentSub } = context; // 解构出store
  const subscription = new Subscription(store, parentSub); // 创建当前组件的subscription
  // 保存上一次的值
  const lastChildProps = useRef();
  //使用useReducer得到一个强制更新函数
  const [, forceComponentUpdateDispatch] = useReducer((count) => count + 1, 0);

  // 获取传递给组件的props
  const childPropsSelector = (store, wrapperProps) => {
    const state = store.getState();
    // 执行mapStateToProps和mapDispatchToProps
    const stateProps = mapStateToProps?.(state);
    const dispatchProps = mapDispatchToProps?.(store.dispatch);
    return Object.assign({}, stateProps, dispatchProps, wrapperProps);
  };

  //对比state，处理回调
  const compareStateForUpdate = () => {
    const newChildProps = childPropsSelector(store, wrapperProps);
    if (isEqual(newChildProps, lastChildProps.current)) return;
    lastChildProps.current = newChildProps;
    forceComponentUpdateDispatch();
    subscription.notifyNestedSubs();
  };
  const actualChildProps = childPropsSelector(store, wrapperProps);

  useLayoutEffect(() => {
    lastChildProps.current = actualChildProps;
  }, [actualChildProps]);

  // 使用subscription注册回调
  subscription.onStateChange = compareStateForUpdate;
  subscription.trySubscribe();

  //重写contextValue，把自己的subscription传递下去
  const overWriteContextValue = {
    ...context,
    subscription
  };

  return (
    <ReactReduxContext.Provider value={overWriteContextValue}>
      <WrappedComponent {...actualChildProps} />
    </ReactReduxContext.Provider>
  );
};
```

[connect源码](https://github.com/reduxjs/react-redux/blob/v7.2.0/src/connect/connect.js#L46)

## 总结
在本文中，提出了上一篇文章中`connect`实现的问题，由于 Redux 的引入使得 React 原本的数据流遭遇破坏。通过引入`Subscription`类实现发布订阅模式，来保证父**父→子**的一个更新顺序。数据发生改变时，从根组件开始通知自己的子组件，子组件通知其子组件，这样来保证更新顺序。

