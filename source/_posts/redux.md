---
title: "Redux 原理"
date: 2018-03-5
tags:
  - react
  - redux
---

# Redux 原理

## 前言

最开始接触 react 技术栈的时候，最不容易懂的就是 redux 部分。很多新的名词 store、action、reducer、state、dispatch、middleware 等等。网上找到的 redux 教程很难看明白，虽然大概晓得它是做什么的，又有什么作用，不过其中的原理还是很难理解。
随着 redux 的使用，以及对其中的源码的学习，后来发现其实原理真的很简单。

整个 redux 原理其实是内部维护一个 state 对象,每当 dispatch 一个 action，通过 reducer 函数匹配 action.type 处理,从而得到新的 state,再利用订阅发布模式, 通知所有注册的监听回调。

当然可能看不太明白，随着之后源码的阅读，再回头看会恍然大悟。

不过其中几点需要注意：

- redux 和 react 没有任何关系，redux 是单纯的状态管理器，可以和任何框架配合使用，比如 vue。
- redux 之所以能和 react 配合使用，是因为 react-redux 提供了将 redux 和 react 连接起来，之后会介绍 react-redux

## createStore

首先介绍 createStore。 createStore 主要是用来创建 store, redux 的三大原则第一条就是单一数据源，所以整个 App 应该只创建一个 store。

createStore 接收的第一个参数是 reducer 函数，第二个是可选的 preloadSsate,第三个是可选参数 enhancer。

其中 reducer 可以是普通的函数也可以是使用 combineReducers 生成的函数。preloadState 不用多少是前置的 state 初始值。enhancer 是用来加强 redux 的，其中 redux 自带的 applyMiddleware 就是一个 enhancer。

再看一下 createStore 的返回，它返回一个 Object,也就是 store

```js
return {
  dispatch,
  subscribe,
  getState,
  replaceReducer,
  [$$observable]: observable
};
```

接下来看一下 createStore 内部实现。

### dispath

dispath 方法就是发出一个 action,而这个 action 会被 reducer 函数接收并处理，生成新的 state,然后通知所有注册的监听回调。

```js
// 接收一个action参数，action 必须是一个plain object 并且有type字段
function dispatch(action) {
  if (!isPlainObject(action)) {
    throw new Error("Action must be plain object.");
  }

  if (action.type === undefined) {
    throw new Error("Action may not have an undefined type property");
  }
  // 通过createStore 内部isDispatching变量控制，只有reducer函数处理完action之后才允许接收新的action
  if (isDispatching) {
    throw new Error("you may not call dispath while the reducer is executing.");
  }

  try {
    isDispatching = true;
    currentState = currentReducer(currentState, action); // reducer 处理action,生成新的state
  } finally {
    isDispatching = false;
  }

  // 更改完state, 通知所有注册的listener回调
  const listeners = (currentListeners = nextListeners);
  for (let i = 0; i < listeners.length; i++) {
    listeners[i]();
  }
  return action;
}
```

### subscribe

前边 dispatch 修改 state 之后，就会通知所有注册的监听，subscribe 就是用来注册一个监听的。

```js
// 接收一个回调函数
function subscribe(listener) {
  if (typeof listener !== "function") {
    throw new Error("Expected listener must be a function.");
  }
  // 当reducer 处理action 的时候，不允许注册
  if (isDispatching) {
    throw new Error(
      "you may not call subscribe while the reducer is executing."
    );
  }

  // 这里需要注意的一点是
  // 这里维护了两个listeners数组： currentListeners 和 nextListeners。当subscribe时，先通过ensureCanMutateNextListeners函数拷贝一份当前的listeners数组（currentListeners）到 nextListeners。然后修改subscribe 修改的是nextListeners。当dispath 的时候再将nextListeners赋值给currentListeners。
  // 虽然有点绕，但原因很简单：为了保证dispath中遍历 listeners 数组的时候不会受到 subscribe 和 unsubscribe 的影响。
  let isSubscribed = true;
  ensureCanMutateNextListeners(); // 拷贝listeners数组
  nextListeners.push(listener);

  // 返回当前监听的注销函数， 调用此函数可注销当前的监听
  return function unsubscribe() {
    if (!isSubscribed) {
      return;
    }

    if (isDispatching) {
      throw new Error(
        "you may not call unsubscribe while the reducer is excecuting."
      );
    }
    isSubscribed = false;
    // 同上 拷贝listeners数组
    ensureCanMutateNextListeners();
    const index = nextListeners.indexOf(listener);
    nextListeners.splice(index, 1);
  };
}
```

### getState

获取当前 state 的值

### replaceReducer

用于替换当前的 reducer 函数,不常用

### \$\$observable

observable 利用了一个第三方 [symbol-observable](https://github.com/tc39/proposal-observable)，对外暴露了一个 subscribe 方法，内部实际上是调用了上边的 subscribe 方法来注册监听。

## combineReducers

combineReducers 的作用主要是将多个 reducer 函数聚合起来，避免写一个庞大复杂的 reducer 函数。

### 参数

combineReducers 接收一个 object,其中每一个 key 对应一个 reducer 函数，而这里的每一个 key 都会对应 state 中的 key 值。这里就要注意在 createStore 接收的 preloadSsate 的结构就要和这里保持一致。

### assertReducerShape

首先跳过前边的类型检查，然后就是 assertReducerShape 函数，它的作用是对每一个 reducer 函数做一次校验，传递一个 ActionTypes.INIT 和一个随机字符串的 type,检测是否有合法的返回值。这里就要求每一个 reducer 函数都必须有初始值，并且必须能响应任意的 type 返回合法值。主要是利用 switch case 的 default 来返回 state。

代码直接阅读源码，这里就不贴出来了。

### combination

检测完每一个 reducer 函数之后，combineReducers 将返回一个 combination 函数，其实就是一个组合的 reducer 函数（combination 函数也将接收 state 和 action, 然后返回新的 state）。

首先是在 debug 模式下 利用 getUnexpectedStateShapeWarningMessage 函数对 reducers 和 state 进行校验。

- reducers 不能为空 object
- state 必须是一个 plain object
- reducers 和 state 结构必须一致（key 值）

之后将遍历 finalReducerKeys 将接收到的 action 交给每一个 reducer 处理生成 state（nextStateForKey）， 将 previousStateForKey 和 nextStateForKey 进行对比，利用 hasChanged 变量作为标记，只要有一个 reducer 函数响应之后的 state 发生了变化，就返回新的 state 对象，从而出发刷新。

```js
function combination(state = {}, action) {
  if (shapeAssertionError) {
    throw shapeAssertionError;
  }

  if (process.env.NODE_ENV !== "production") {
    const warningMessage = getUnexpectedStateShapeWarningMessage(
      state,
      finalReducers,
      action,
      unexpectedKeyCache
    );
    if (warningMessage) {
      warning(warningMessage);
    }
  }

  let hasChanged = false;
  const nextState = {};
  for (let i = 0; i < finalReducerKeys.length; i++) {
    const key = finalReducerKeys[i];
    const reducer = finalReducers[key];
    const previousStateForKey = state[key];
    const nextStateForKey = reducer(previousStateForKey, action);
    if (typeof nextStateForKey === "undefined") {
      const errorMessage = getUndefinedStateErrorMessage(key, action);
      throw new Error(errorMessage);
    }
    nextState[key] = nextStateForKey;
    hasChanged = hasChanged || nextStateForKey !== previousStateForKey;
  }
  return hasChanged ? nextState : state;
}
```

## bindActionCreators

bindActionCreators 将生成 action 的 actionCreator 函数和 dispath 封装了一下，主要是用来简化代码。

文件内部有一个 bindActionCreator 函数，它接收 actionCreator 和 dispatch, 主要是用来生成一个新的函数，调用时将 actionCreator 生成的 action dispath 触发出去。

bindActionCreators 则是将 actionCreators 对象 利用 bindActionCreator 函数转化，将 每一个 value 封装成一个新的函数。

## applyMiddleware

在 [createStore](./createStore.md) 中提到了 enhancer 增强器，applyMiddleware 就是 redux 内置的一个 enhancer。主要是利用中间件强化 store.dispatch.

applyMiddleware 接收一个中间件函数的数组，并且返回一个 enhancer 增强器。

主要利用接收到的 createStore 函数，先生成 store, 在遍历 middlewares 中间件 加工 store.dispatch，然后通过 [compose 函数](https://github.com/reduxjs/redux/blob/master/src/compose.js) 组合，最后返回 新的 store。

```js
const store = createStore(...args); // 生成 store
let dispatch = () => {
  throw new Error(
    "Dispatching while constructing your middleware is not allowed. " +
      "Other middleware would not be applied to this dispatch."
  );
};
const middlewareAPI = {
  getState: store.getState,
  dispatch: (...args) => dispatch(...args)
};
const chain = middlewares.map(middleware => middleware(middlewareAPI)); // 遍历中间件数组， 传入 middlewareAPI 参数， 并返回一个函数， 这个函数接收一个dispath 函数，并且返回 处理之后的dispatch 函数。

// 利用 compose 将每个中间件返回的函数组合起来
// 每一个 middleware函数返回的函数接收到的参数都是 上一个 返回函数的返回值（也就是dispatch函数）， 并且返回dispath 函数（将会被下一个 middleware函数返回的函数 所接收为参数）。 这样依次进行，最终返回一个被所有 middleware 返回函数 处理之后的 dispatch函数。
dispatch = compose(...chain)(store.dispatch);

return {
  ...store,
  dispatch
};
```

下边是一个 middleware 函数的例子

```js
function logger({ getState, dispatch }) {
  // 在 middlewares.map 阶段被返回的函数, 它将接收 compose 组合时上一个 函数的返回值（dispatch 函数），并返回一个 dispatch 函数
  return function wrapDispatchToAddLogging(next) {
    return function dispatchAndLog(action) {
      // 返回一个 dispatch 函数（及接收action 并处理），当调用 dispatch 是 这个函数也将被调用
      console.log("dispatching", action);
      let result = next(action);
      console.log("next state", store.getState());
      return result;
    };
  };
}
```

这里建议 看一下 react-thunk 的实现， 加深一下理解。

这里看一下 applyMiddleware 是怎么用的

```js
createStore(
    rootReducers,    //reducer
    preloadedState,
    applyMiddleware( //enhancer
        thunkMiddleware,
        createLogger
    )
)

// createStore 涉及到的部分源码 :

export default function createStore(reducer, preloadedState, enhancer) {
    if (typeof preloadedState === 'function' && typeof enhancer === 'undefined') {
        enhancer = preloadedState
        preloadedState = undefined
    }

    if (typeof enhancer !== 'undefined') {
        if (typeof enhancer !== 'function') {
            throw new Error('Expected the enhancer to be a function.')
        }
        // 在这里可以看到，如果第三个参数是函数也就是applyMiddleware
        return enhancer(createStore)(reducer, preloadedState)
    }

    if (typeof reducer !== 'function') {
        throw new Error('Expected the reducer to be a function.')
    }
    ...
}
// 于是在creatStore的时候传入了enhancer，那就会走到这里来，由enhancer的内部自己创建一个store,然后增强store的某一部分功能，返回出去。对于middleware来说，增强的就是dispatch的功能。

```

## compose

```js
/**
 * @param {...Function} funcs The functions to compose
 * @returns { Function }
 * For example
 * compose(a, b , c)
 * (...args) => a(b(c( ...args )))
 */
export default function compose(...funcs) {
  if (funcs.length === 0) {
    return arg => arg;
  }

  if (funcs.length === 1) {
    return funcs[0];
  }

  return funcs.reduce((result, fn) => (...args) => result(fn(...args)));
}
```

## 结束

redux 这里就结束了,接下来再看一下 react-redux 的代码。
