---
layout: post
title: 'React生命周期介绍'
subtitle:   "React lifecycle"
date:       2018-07-21 22:31:00
author:     "Bryan"
header-mask: 0.3
catalog:    true
tags:
    - react
    - frontend
---

## react生命周期
众所周知，react会有初始化的阶段，以及数据更新后的重新render阶段，在这些阶段中，会存在一些回调函数，react组件会在特定的阶段调用对应的回调函数。如果我们希望在特定的时间执行一些业务操作，就需要了解react提供的这些回调函数。

![react生命周期](/img/in-post/react-lifecycle/react.jpg)

#### constructor
构造函数，在构建组件的时候调用

#### componentWillMount
在组件挂载之前被触发，执行中在render之前触发，在此方法中调用setState()不会触发额外的渲染

#### componentDidMount
在组件挂载之后触发，此时子组件也挂载好了，在这个方法中可以使用refs。如果需要向远端获取数据，建议在这个方法中初始化网络请求。

#### componentWillReceiveProps
子组件的props来自于父组件，当父组件重新render的时候，就会触发子组件的更新，此时会调用子组件的componentWillReceiveProps方法，将新的props传递给子组件（即使父组件中props没有变化也会触发此方法）

#### shouldComponentUpdate
这个方法在父组件传递新的props或子组件自己调用setState时被触发，用于判断组件是否需要重新render。当返回true时，组件会重新render，默认情况下会返回true。具体业务中，可以根据数据判断是否有必要重新render，通过此方法可以避免不必要重复render，优化效率

#### componentWillUpdate
此方法在更新阶段的render之前被触发，用于做一些render之前的准备工作

#### componentDidUpdate
在更新阶段的render之后触发，与创建阶段的componentDidMount类似，注意，react组件只有初始化的时候会执行创建阶段的方法，而后在每一次更新state或props时，都只会执行更新阶段的方法，因此componentDidMount可能会多次执行

#### componentWillUnmount
在组件被卸载时会被调用，一般执行必要的清理工作

#### render
最后介绍最重要的render方法，render用于构建react的前端展示，注意，不要在render中添加setState更新状态，可能会导致死循环。

## react新生命周期
在react16.3开始，对react生命周期进行改变，移除了部分生命周期方法，同时增加了部分生命周期方法。下面是16.3的生命周期方法图示，具体图示来自于[react图表](http://projects.wojtekmaj.pl/react-lifecycle-methods-diagram/)

![新的react生命周期](/img/in-post/react-lifecycle/new-react.png)

#### getDerivedStateFromProps
getDerivedStateFromProps方法是用于替换之前的componentWillReceiveProps方法，都是在render之前触发，但是区别在于此方法在构建阶段和更新阶段都会触发，都是在于有了新的props时做出一些处理。在react16.4中，setState和forceUpdate方法都会触发此方法。

#### getSnapshotBeforeUpdate
此方法是在render方法之后，在react的VDOM更新到DOM之前，可以用于做出一些精细化的调整，比如在聊天应用的中的特定场景调整滚动条的位置。应用场景比较少。

#### UNSAFE_componentWillMount
原来的componentWillMount被弃用，在react16.3会有警告信息，如果希望忽略警告继续使用，那么可以使用UNSAFE_componentWillMount方法，但未来此方法会被废弃。原来一般在此方法中做出的操作，官方建议都移动至componentDidMount中进行处理

#### UNSAFE_componentWillReceiveProps
原始的componentWillReceiveProps被弃用，可以使用getDerivedStateFromProps进行替代

#### UNSAFE_componentWillUpdate
原始的componentWillUpdate已被弃用，此方法是在render之前触发的，用于在render之前做出一些准备工作，事实上官方不建议在此方法中执行setState或其他导致重新render的操作，因此，原始在此方法中做出的处理都可以移动到componentDidUpdate中，如果希望在DOM更新之前做出处理，那么可以移动的到getSnapshotBeforeUpdate方法中。

上面的介绍的方法都是基于react16.3和16.4进行介绍，参考了官方的一些介绍，如果希望紧跟官方，可以参考[官方文档](https://reactjs.org/docs/react-component.html#constructor)
