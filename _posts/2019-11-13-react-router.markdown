---
layout: post
title: 'React router 入门'
subtitle:   "Introduction to react router "
date:       2019-11-13 9:17:00
author:     "Bryan"
header-mask: 0.3
catalog:    true
tags:
    - react
    - frontend
---

## 背景介绍

react router 是一个强大的 react 组件，可以提供强大的前端路由功能，算是 react 全家桶中一个相当重要的组件。如果使用 react 技术栈实现 web 服务，那么 react router 是一个极好的选择。

在介绍 react router 之前，首先需要明确介绍的 react router 的版本。

react router 3.x 和 react router 4.x 拥有完全不一样的设计思路和相应的 API，react  router 3.x 更接近其他框架下的路由控制，属于静态路由，而 react router 4.x 则演变为动态路由。这两者思想上的差别导致了 API 与写法上的千差万别，关于动态路由的思想可以看 [react router 哲学](https://reacttraining.com/react-router/web/guides/philosophy) 。

而 react  5.x 是一个小版本更新，因此与 4.x 是完全兼容的。而为什么小版本更新会发布一个大版本号呢？原因是某版本依赖配置写错了，这一段笑料可以查看 [Michael Jackson 的博客](https://reacttraining.com/blog/react-router-v5/) 。在 react router 5.1 中终于出现了更重要的功能特性，支持了新的 Hooks 方法。

本文主要基于 react router 4.x 以后的版本，介绍基础的 react router 使用所需的框架知识，最后介绍 react router 5.1 中引入的 Hooks 方法。

## 基础使用

在具体介绍之前，我们可以参考的 react router 提供的一个简化版的基础例子来了解下具体的使用是怎样的。如果希望实际操作可以查看 [Basic demo](https://reacttraining.com/react-router/web/example/basic) ，可看到基础的代码如下所示：

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

实现的功能是点击上面的`Home` 链接，展示的是方法 `Home()` 中的内容 ，而点击 `About` 链接，最终展示的内容是 方法 `About()` 中的内容。

简单解释一下：无序列表中包含两个链接组件 `<Link>` ，此组件类似于 `<a>` ，可以进行路径跳转。下面使用 `<Route>` 组件用于在匹配到特定的路径时渲染特定的内容。匹配的路径通过 `path` 参数进行执行，渲染的通过 `component` 参数指定。

## 基础组件

react router 中使用三种类型的基础组件，分别是：

1. Routers ，路由组件，每个 react router 应用都应该有一个 Router 组件，用于包含所有 Route Matchers 组件，一般而言，Router 会放在所有渲染元素的顶层；
2. Route Matchers，路由匹配组件，用于指定匹配路径后渲染的内容；
3. Navigation，导航组件，用于进行路径跳转。

#### Routers

react router 应用的核心是 Router 组件，Router 组件用于包含 Route 组件。对于 Web 应用，可以使用 `BrowserRouer` 和 `HashRouter` ，这两者有比较明显的区别：

- `BrowserRouter` 是基于 html5-mode 实现的路由管理，访问的网址与看起来像常规的网址，对于用户来说看起来体验更好，但是服务端的支持，因为用户可能会在访问任意的路径，需要对不同路径的请求访问同样的页面，并利用 react router 去管理页面内容；
- `HashRouter` 是基于 hash-mode 实现的路由管理，这种路由管理将路径保存在 hash 部分，因此路径类似于 `http://example.com/#/your/page` ，由于 hash 中保存的路径不会被发送给服务器端，因此不需要服务端额外配置；

#### Route Matchers

路由匹配组件主要包含 Route 组件与 Switch 组件，下面分别介绍其用途：

###### Route

Route 组件用于指定匹配的路径与渲染的内容。当用户访问的路径与 Route 组件指定的路径匹配时，指定的渲染内容就会被渲染出来。在使用 Route 组件时，有下面两点需要注意：

1. Route 通过 `path` 参数指定匹配的路径，此路径是判断当前 URL 是否以 `path` 指定的路径开头。如果是以 `path` 指定的路径开头则渲染相应的内容；

   比如指定 `<Route path="/about" component={About} />` 则匹配到的路径不仅包括 `/about` ，也包含 `/about/me` ，`/about/you` 路径。

   如果需要仅仅匹配 `/about` 路径，可以增加 exact 参数，类似`<Route path="/about" exact component={About} />`  则仅仅匹配 `/about` 路径。

2. 多个 Route 的路径都满足匹配条件时，多个 Route 对应的内容都会被渲染出来；

###### Switch

刚刚介绍的 Route 中提到多个 Route 会在同时被匹配到时都渲染出来，这个在大部分情况都不能满足路由管理的需求。官方提供了 Switch 组件，此组件会包含多个 Route 组件，多个 Route 组件都与当前路径匹配时，只有第一个匹配的 Route 组件会被渲染出来；

#### Navigation

导航组件主要用于路径的跳转，主要包含三个有用的组件：

1. `Link` 此组件类似于 `<a>` ，但是不会导致数据的重新获取，只是在 react router 中进行路径跳转，使用类似如下所示：

   ```react
   <Link to="/">Home</Link>
   ```

2. `NavLink` 这是一种特殊的 `Link` 组件，会在当前路径匹配时处于 "active" 状态，比较适合用来做导航栏，使用类似如下所示：

   ```react
   <NavLink to="/react" activeClassName="hurray">
     React
   </NavLink>
   ```

   在上面的 `NavLink` 指定了 "active" 状态下的样式 `hurray` ，在路径匹配则会增加相应的样式；

3. `Redirect` 用于进行路径的重定向。如果在渲染内容时，需要进行重定向，可以渲染出一个 `Redirect` 组件，使用类似如下所示：

   ```react
   <Redirect to="/login" />
   ```

## Hooks API

随着 react router 5.1 的发布，react router 提供了 4 个全新的 Hooks 方法，下面分别进行介绍：

#### useHistory

useHistory Hooks 用于获取 [history 实例](https://reacttraining.com/react-router/web/api/history) ，从而用于动态进行导航跳转。可以调用 history 实例的方法，比如 `history.push()` 和 `history.goBack()` 等进行导航的管理。一个官方的例子如下所示：

```react
import { useHistory } from "react-router-dom";

function HomeButton() {
  let history = useHistory();

  function handleClick() {
    history.push("/home");
  }

  return (
    <button type="button" onClick={handleClick}>
      Go home
    </button>
  );
}
```

在上面的例子中，可以使用 `history.push()` 方法，在点击按钮之后就会跳转至路径为 `/home` 对应的页面

#### useLocation

useLocation Hooks 用于获取一个 [location 实例](https://reacttraining.com/react-router/web/api/location) ，用于获取当前用户访问的路径，以及对应的 query , hash 部分。一个简单的例子如下所示：

```react
function usePageViews() {
  let location = useLocation();
  
  React.useEffect(() => {
    console.log(location.pathname);
    console.log(location.search);
    console.log(location.hash);
  }, [location]);
}
```

在上面的例子中，使用 `location.pathname` 获取了当前的路径，使用 `location.search` 获取了当前的 query，使用 `location.hash` 获取当前的 hash。

#### useParams

useParams Hooks 用于获取 URL 中动态匹配参数。

比如希望匹配一个可变的路径，比如使用 `<Route path="/blog/:slug"` 通过变量 `slug` 去匹配类似 `/blog/...` 路径的网址，比如当前路径为 `/blog/abc` ，此时变量 `slug` 即为 `abc` ，在渲染的组件中即可使用此变量进行个性化内容的展示。

之前为了获取到此变量的值，可以在渲染内容时将变量显式传入。使用 useParams Hooks 则不需要显示传入，在需要的地方直接通过 useParams Hooks 直接获取到所有变量的键值对。一个例子如下所示：

```react
function BlogPost() {
  let { slug } = useParams();
  return <div>Now showing post {slug}</div>;
}

ReactDOM.render(
  <Router>
    <Switch>
      <Route exact path="/">
        <HomePage />
      </Route>
      <Route path="/blog/:slug">
        <BlogPost />
      </Route>
    </Switch>
  </Router>,
  node
);
```

在上面的例子中，存在变量 `slug` ，在渲染的内容方法 `BlogPost()` 方法中，通过 useParams Hooks 获取到变量键值对。并解构存储至变量 slug 中进行展示。

#### useRouteMatch

useRouteMatch 用于动态构建 Route，可以替代 Route。可以更灵活地构建 Route，特别适用于可变路径匹配的情况。使用 useRouteMatch Hooks 得到的是一个 [match 对象](https://reacttraining.com/react-router/web/api/match) 。 一个简单的对比样式如下所示：

```react
//before
function BlogPost() {
  return (
    <Route
      path="/blog/:slug"
      render={({ match }) => {
        // Do whatever you want with the match...
        return <div />;
      }}
    />
  );
}

// after
function BlogPost() {
  let match = useRouteMatch("/blog/:slug");

  // Do whatever you want with the match...
  return <div />;
}
```

两种代码是等价的，可以看到使用 useRouteMatch Hooks 会让代码更加直观。同时 Route 相关的可配置的参数如 `exact` ，`strict` 都是支持的。



## 总结

react 全家桶看起来越来越强大，也接管了越来越多的东西，路由的管理都被很好地管理起来了。这一方便确实可以大大提升效率，但是在全家桶之外的复用变得比较难。但是如果入门 react 全家桶的坑，就不要犹豫了，都用起来吧。