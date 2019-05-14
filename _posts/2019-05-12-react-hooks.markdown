---
layout: post
title: 'React hooks 入门'
subtitle:   "React hooks"
date:       2019-05-12 22:45:00
author:     "Bryan"
header-mask: 0.3
catalog:    true
tags:
    - react
    - frontend
---

## 背景介绍

React 使用过程中，一个朴素的思想是代码的复用。通过拆分为基础的组件，React 已经可以方便地实现组件的复用。但是在 React 中，纯粹的逻辑复用是很困难的，因为在 React 中，各种逻辑代码是散步在 React 的各个生命周期方法中。因为这个现象，React 中的状态管理是很不清晰的。为了解决逻辑复用的问题，也为了更好地管理 React 中的状态，Hooks 就横空出世了。

## Hooks 初探

首先选择一个最简单例子看看 Hooks 的魅力，我们选择了一个简单的带有状态管理的小例子，具体代码如下所示：

```react
class Example extends React.Component {
  constructor(props) {
    super(props);
    this.state = {
      count: 0
    };
  }

  render() {
    return (
      <div>
        <p>You clicked {this.state.count} times</p>
        <button onClick={() => this.setState({ count: this.state.count + 1 })}>
          Click me
        </button>
      </div>
    );
  }
}
```

使用过 React 的应该比较容易理解，最终会展示的内容包含一个文本以及一个按钮，按钮点击会更新文本的内容。但是可以看到，如此简单的状态管理都需要维护在不同的方法中，需要在 `constructor()` 方法中设置初始状态，在 `render()` 中增加方法更新状态。而且为了管理，内容必须为`React.component` 

采用 Hooks 后代码如下所示：

```react
import { useState } from 'react';

function Example() {
  const [count, setCount] = useState(0);

  return (
    <div>
      <p>You clicked {count} times</p>
      <button onClick={() => setCount(count + 1)}>
        Click me
      </button>
    </div>
  );
}
```

这段代码与初始的代码基本是等价的，通过 Hooks 提供的 `useState()` 方法返回一个状态 `count` ，一个修改状态的方法 `setCount()` ，后续就可以直接使用了。

对比可以看到，使用 Hooks 之后，相关的状态逻辑可以在单个方法中进行管理，而且也不再需要写成 `React.component` 组件，而是作为一个逻辑状态管理方法，复用起来也更便利。

## 基础 Hooks

React 官方提供了一些基础的 Hooks， 可以帮助我们比较好地优化代码，下面对列举实用的基础 Hooks：

#### useState

`useState` 是最基础的 Hooks 了，`useState` 是一个基础的状态管理的 Hooks，将原本分散的初始化状态，获取状态，更改状态统一在一个方法中了。这样便于将单个状态相关的内容统一维护，复用起来也更轻松一些。具体如下所示：

```react
const [count, setCount] = useState(0);
```

上面的代码初始化状态为 0， 返回两个值 `count` 和 `setCount()` ， 一个用于获取最新的状态，一个是用于更新状态。后续与此状态相关的代码都维护在同一个方法中了。使用十分简单，但是效果也很明显，使用之后开发者慢慢就会按照状态组织代码了，复用起来也会更方便

#### useEffect

 前面的 `useState` 虽然很方便，但是有一个痛点依旧无法解决。就是在当前状态的管理依赖外部世界时，比如在之前我们会在 `componentDidMount()` 初始化网络请求，而会在 `componentWillUnmount()`  中执行组件卸载的清理，这也是之前将状态管理分拆到不同方法进行管理的重要原因。对于这个问题，Hooks 又提供了一个怎样的解决方案呢，可以从一个例子看到：

```react
class FriendStatus extends React.Component {
  constructor(props) {
    super(props);
    this.state = { isOnline: null };
    this.handleStatusChange = this.handleStatusChange.bind(this);
  }

  componentDidMount() {
    ChatAPI.subscribeToFriendStatus(
      this.props.friend.id,
      this.handleStatusChange
    );
  }

  componentWillUnmount() {
    ChatAPI.unsubscribeFromFriendStatus(
      this.props.friend.id,
      this.handleStatusChange
    );
  }

  handleStatusChange(status) {
    this.setState({
      isOnline: status.isOnline
    });
  }

  render() {
    if (this.state.isOnline === null) {
      return 'Loading...';
    }
    return this.state.isOnline ? 'Online' : 'Offline';
  }
}
```

在上面的例子中，是一个获取好友在线状态的例子，在初始化时执行 `ChatAPI.subscribeToFriendStatus()` 订阅好友状态，在组件卸载时通过 `ChatAPI.unsubscribeFromFriendStatus()` 取消订阅。由于方法是与生命周期相关，直接使用 `useState()` 无法解决此问题，此时使用 `useEffect()` 就可以完美解决。改造后的等价代码如下：

```react
import { useState, useEffect } from 'react';

function FriendStatus(props) {
  const [isOnline, setIsOnline] = useState(null);

  function handleStatusChange(status) {
    setIsOnline(status.isOnline);
  }

  useEffect(() => {
    ChatAPI.subscribeToFriendStatus(props.friend.id, handleStatusChange);
    // Specify how to clean up after this effect:
    return function cleanup() {
      ChatAPI.unsubscribeFromFriendStatus(props.friend.id, handleStatusChange);
    };
  });

  if (isOnline === null) {
    return 'Loading...';
  }
  return isOnline ? 'Online' : 'Offline';
}
```

下面具体解释一下，`useEffect()` 可以执行与生命周期相关的代码，方法内部除返回方法相关的代码会在 DOM 更新时执行，也就是可以替代 `componentDidMount()` 和 `componentDidUpdate()` ，而方法的返回值对应的方法会在 React 组件卸载时执行，也就是可以替代 `componentWillUnmount()` 方法。

#### useContext

在 React 中，为了避免深层次的参数传递，一般会在父组件上创建 Context， 在子孙组件都可以获取到子组件，这样就避免了一层一层的参数传递。这个算是 React 提供的一个使用有用的基础功能。但是 Context 的使用比较不方便，使用 `useContext()` 就可以使这个过程变得更加容易。一个简单的使用例子如下所示：

```react
import * as React from 'react';

const Context = React.createContext(0);

function Display() {
  const context = React.useContext(Context);
  return <p>Current State: {context}</p>;
}

function Example() {
  const [state, setState] = React.useState(0);

  return (
    <div>
      <Context.Provider value={state}>
        <Display />
      </Context.Provider>
      <p>
        <button onClick={() => setState(v => v + 1)}>+</button>
        <button onClick={() => setState(v => v - 1)}>-</button>
      </p>
    </div>
  );
}
```

在上面的例子中可以看到，首先通过 `React.createContext()` 方法创建一个 Context 对象，然后通过 `Context.Provider` 设置 value 的值，将值传递给子组件 `<Display>` 。而在子组件中，就可以通过 React 提供的`useContext()` 方法获取 Context 的值。通过使用 `useContext()` 获取 Context 的值使用比较清晰而方便。

#### useMemo

在某些情况下，我们需要耗费大量的资源计算一个值，而这个值会受一些参数所影响，当参数没有变化时，这个值不会变化，当参数变化时，需要重新计算。为了提升效率，我们可以使用 `useMemo()` 这个 Hooks，此 Hooks 接受两个参数，一个是计算值的方法，一个是会影响值的参数。当任一参数变化时，才会重新计算，否则就不会重新计算。使用如下所示：

```react
const memoizedValue = useMemo(() => computeExpensiveValue(a, b), [a, b]);
```

上面的例子中，计算得到的值 memoizedValue 会受参数 a, b 的影响，当参数 a, b 不变时，不会重新计算memoizedValue 的值，而 a, b 任一参数改变时，都会触发重新执行 `computeExpensiveValue(a, b)` 方法重新计算 memoizedValue 的值。在避免重新计算上，`useMemo()` 还是比较方便的。

#### useCallback

`useCallback()` 方法与 `useMemo()` 类似， 也是避免重新计算，区别在于`useCallback()` 方法最后返回的是回调方法，此回调方法只有在输入发生更改时才会更改。使用如下所示：

```react
const callbackFunc = useCallback(() => doSomething(a, b), [a, b]);
```

最终在需要的地方可以直接调用`callbackFunc()` 方法，最终执行的就是`doSomething()` 部分的代码，可以认为是一个优化版的方法。

#### useRef

在之前的 React 使用中，我们经常会使用 ref 去找到特定的 DOM 元素，Hooks 中也提供了一个新的 `useRef()` 方法来帮助我们更加方便地使用 ref。具体的使用如下所示：

```react
function TextInputWithFocusButton() {
  const inputEl = useRef(null);
  const onButtonClick = () => {
    // `current` points to the mounted text input element
    inputEl.current.focus();
  };
  return (
    <>
      <input ref={inputEl} type="text" />
      <button onClick={onButtonClick}>Focus the input</button>
    </>
  );
}
```

可以看到上面的代码中，使用 `useRef` 创建了一个 ref 对象 inputEl ，后续在 input 元素中使用 `ref={inputEl}` 将 ref 对象 inputEl 与 input 元素映射上，后续就可以直接使用 `inputEl.current` 获取到 input 元素。使用过程简洁清晰。

## 总结

通过使用 Hooks，可以看到前端代码重用上不断提升，之前前端页面重复代码日子一去不复返了。Hooks 很吸引我的一点在于，Hooks 的代码是与之前的 React 代码完全兼容的，可以在任意时刻开始使用 Hooks，新页面可以使用 Hooks 写，老页面可以继续保留，完全不需要大面积的重写代码。与此相比，Python 在这方便就没那么友好了。有逻辑代码重用的同学们，不妨尝试起来吧。

