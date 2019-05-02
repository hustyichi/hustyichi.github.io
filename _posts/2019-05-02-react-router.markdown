---
layout: post
title: 'React router 入门'
subtitle:   "Introduction to react router "
date:       2019-05-02 10:58:00
author:     "Bryan"
header-mask: 0.3
catalog:    true
tags:
    - react
    - frontend
---

## 基础介绍

react router 是一个强大的 react 组件，可以提供强大的前端路由功能，算是 react 全家桶中一个相当重要的组件。由于 react router 从 v4 开始与之前彻底不兼容，为了避免误会，本文主要是针对 v4 以上的版本，旧版本的入门教程可以参考 [阮一峰的教程]([http://www.ruanyifeng.com/blog/2016/05/react_router.html](http://www.ruanyifeng.com/blog/2016/05/react_router.html)) 。

## 基础使用

在具体介绍之前，我们可以参考的 react router 提供的一个简化版的基础例子来了解下具体的使用是怎样的。可看到基础的代码如下所示：

```react
import React from "react";
import { BrowserRouter as Router, Route, Link } from "react-router-dom";

function BasicExample() {
  return (
    <Router>
      <div>
        <ul>
          <li>
            <Link to="/">Home</Link>
          </li>
          <li>
            <Link to="/about">About</Link>
          </li>
        </ul>

        <hr />

        <Route exact path="/" component={Home} />
        <Route path="/about" component={About} />
      </div>
    </Router>
  );
}

function Home() {
  return (
    <div>
      <h2>Home</h2>
    </div>
  );
}

function About() {
  return (
    <div>
      <h2>About</h2>
    </div>
  );
}
```

最终执行的结果如下图所示：

![basic_example](/img/in-post/react-router/basic_example.png)

实现的功能是点击上面的`home` 链接，下面展示的是`<h2>Home</h2>` ，而点击 `About` 链接，最终展示的内容是 `<h2>About<h2>` 内容。

简单解释一下：无序列表中包含两个链接组件 `<Link>` ，此组件类似于 `<a>` 但是不会刷新当前页面，下面使用 `<Route>` 组件用于在匹配到特定的路径时渲染特定的内容。第一个 `Route` 组件会在匹配到 `/` 路径时，渲染 `Home()` 方法中的内容，而第二个 `Route` 会在匹配到 `/about` 路径时，渲染 `About()` 方法中的内容。从而实现上面介绍的功能。

## 基础组件

react router 中使用三种类型的基础组件，分别是：路由组件(router components) ，一般被称为 `router`， 路由匹配组件(route matching components)，一般被称为 `route` ， 导航组件(navigation components)。由于`router` 和 `route` 是 react router 项目中最核心的概念，而且中文不容易区分，下面的介绍中就直接使用英文缩写了。

而所有这些组件都可以从 `react-router-dom` 包中获取，因此理论上一般使用者不需要关心 react-router 的其他包。

#### routers

router 用于包装 route，route 必须被包含在 router 中。每个 router 会有自己独立的历史记录(history)。

router 分成两种类型，分别是`<BrowserRouer>` 和 `<HashRouter>` ，一般情况下，我们应该使用 `<BrowserRouter>` ，但是如果你建立的是一个静态网站，没有任何后端内容，可以考虑使用 `HashRouter`。

#### routes

route 用于匹配到特定的路径，如果匹配到则渲染相应的内容。此组件使用 path 参数指定匹配的路径，使用参数 component, render, children 指定渲染的内容。后续详细介绍相关的细节。

#### 导航组件

众所周知，前端使用 `<a>` 用于跳转，但是`<a>` 是跳转至一个新的网址，所有的内容都重新渲染，而 react router 中进行跳转时，可能只希望对真正有变化的部分进行渲染。这时候如何做呢 ？

react router 提供了 `<Link>` 组件，用于替代 `<a>` ，点击`<Link>` 跳转时，改变了网站的路径，此时不会导致前端内容全部重新渲染。

react router 同时提供了 `<NavLink>` 组件，此组件是一种特殊的`<Link>` 组件，此组件会在路径匹配时，将自己设定为 active 状态

如果你想要强制跳转，你可以渲染一个`<Redirect>` 组件。

## route 使用

#### 路径匹配

route 使用 path 参数指定匹配的路径，具体的路径匹配使用 [path-to-regexp](https://github.com/pillarjs/path-to-regexp) 进行。route 会将 path 参数指定的字符串转换为正则表达式并与当前路径进行匹配。

如果不了解具体的正则表达式参数也没有关系，默认情况下，会判断当前路径是否是 path 指定的字符串开头，如果符合则表示匹配。比如指定下面的 route：

```react
<Route path='/about' component={About}/> 
```

当前路径是 `/about` ,  `/about/me` 和 `/about/me/msg` 时，都能匹配到。

如果只想精确匹配到 path 指定的字符串，而不是 path 指定的字符串开头，那么可以使用 exact 参数，比如指定下面的 route:

```react
<Route exact path='/about' component={About}/> 
```

那么当前是 `/about` 时才能匹配到，当前路径是`/about/me` 和 `/about/me/msg` 时，不会被匹配到

###### 单一渲染

在 router 下存在 多个 route 时，如果多个 route 都匹配到当前路径时，多个 route 指定的渲染内容都会被渲染出来。但是大部分情况下，我们需要都是渲染第一个匹配 route，那么如何做呢 ？

此时可以考虑`<Switch>` 组件，`<Switch>` 内部的 route 只会渲染一个，如果有多个都与当前路径匹配，只有第一个匹配的 route 会被渲染出来，使用类似如下所示：

``` react
<Router>
  <Switch>
  	<Route exact path="/" component={Home} />
  	<Route path="/about" component={About} />
  	<Route path="/contact" component={Contact} />
  </Switch>
</Router>
```

#### 内容渲染

react 指定渲染内容时，可以使用 component, render 或 children。

- component 参数渲染

  用于渲染一个 react 组件，在路径匹配时渲染此组件的内容。一般情况下需要使用此参数指定渲染的内容。

- render 参数渲染

  通过一个内联方法指定渲染的内容，在路径匹配时渲染此内联方法指定的内容。官方建议，只应该在需要传递变量值至组件时使用 render 参数进行渲染。因为使用 component 参数渲染时传递参数可能会导致预期范围外的组件卸载和重新加载。比如：

  ```react
  // 建议做法
  <Route
    path="/about"
    render={props => <About {...props} extra={someVariable} />}
    />
  
  // 不建议做法
  <Route
    path="/contact"
    component={props => <Contact {...props} extra={someVariable} />}
    />
  ```

- children 参数渲染 

  用于渲染 react 元素，但是与 component 与 render 不同，children 参数指定的渲染内容在路径不匹配时也会被渲染。除非必要，一般不建议使用

## 总结

react 全家桶看起来越来越强大，也接管了越来越多的东西，连网址的跳转都被接管了。这一方便确实可以大大提升效率，但是在全家桶之外的复用变得比较难。但是如果入门 react 全家桶的坑，就不要犹豫了，都用起来吧。