---
layout: post
title: 'Airbnb React/JSX 编码规范'
subtitle:   "Airbnb React/JSX Style Guide"
date:       2019-06-23 22:51:00
author:     "Bryan"
header-mask: 0.3
catalog:    true
tags:
    - frontend

---

## 基础介绍

最近 react 的代码写的比较多，代码的编码规范并不明确，导致项目中出现不一样的命名风格。考虑了一下，还是需要统一一下前端的编码规范，搜索了 react 前端的编码规范，看到目前使用最多的应该是 Airbnb 的规范，于是我们也引入进来，有了统一的编码规范才能有统一的代码嘛，具体内容参考自 [Airbnb React/JSX Style Guide](https://github.com/airbnb/javascript/tree/master/react) 。

## 具体规范

#### 基本规则

- 在单个文件只包含一个 React 组件
  - 多个无状态，Pure 组件是允许的, [eslint](https://github.com/yannickcr/eslint-plugin-react/blob/master/docs/rules/no-multi-comp.md#ignorestateless)
- 始终使用 JSX 语法
- 不要使用 `React.createElement` ，除非是在通过非 JSX 文件初始化 app 的情况下
- [react/forbid-prop-types](https://github.com/yannickcr/eslint-plugin-react/blob/master/docs/rules/forbid-prop-types.md) 此规则会禁用 `arrays` 和 `objects` ，除非显式声明 `arrays` 和 `objects` 中包含的内容，使用 `arrayOf`, `objectOf`, 或 `shape`

#### Class vs `React.createClass` vs stateless

- 如果你有内部的 state 或 refs，倾向于使用 `class extends React.Component` 而不是 `React.createClass` 。  eslint: [`react/prefer-es6-class`](https://github.com/yannickcr/eslint-plugin-react/blob/master/docs/rules/prefer-es6-class.md) [`react/prefer-stateless-function`](https://github.com/yannickcr/eslint-plugin-react/blob/master/docs/rules/prefer-stateless-function.md)

  ```react
  // bad
  const Listing = React.createClass({
    // ...
    render() {
      return <div>{this.state.hello}</div>;
    }
  });
  
  // good
  class Listing extends React.Component {
    // ...
    render() {
      return <div>{this.state.hello}</div>;
    }
  }
  ```

  如果你没有使用 state 或 refs，倾向于使用普通的函数（不要选择箭头函数），而不是 classes

  ```react
  // bad
  class Listing extends React.Component {
    render() {
      return <div>{this.props.hello}</div>;
    }
  }
  
  // bad (依赖函数名推理，不鼓励)
  const Listing = ({ hello }) => (
    <div>{hello}</div>
  );
  
  // good
  function Listing({ hello }) {
    return <div>{hello}</div>;
  }
  ```

#### Mixins

- [Do not use mixins](https://facebook.github.io/react/blog/2016/07/13/mixins-considered-harmful.html)

  >  为什么? Mixins 会引入隐式依赖，导致命名冲突，导致雪球式的复杂度。在大多数情况下 Mixins 可以被更好的方法替代，如：组件化，高阶组件，工具模块等。

#### 命名

- 扩展名： React 组件使用 `.jsx` 的拓展名， eslint: [`react/jsx-filename-extension`](https://github.com/yannickcr/eslint-plugin-react/blob/master/docs/rules/jsx-filename-extension.md)

- 文件名： 文件名使用帕斯卡命名（所有单词首字母大写），比如 `ReservationCard.jsx`

- 引用命名： React 组件使用帕斯卡命名，组件实例使用驼峰命名（首个单词首字母小写，其他单词首字母大写）。eslint: [`react/jsx-pascal-case`](https://github.com/yannickcr/eslint-plugin-react/blob/master/docs/rules/jsx-pascal-case.md)

  ```react
  // bad
  import reservationCard from './ReservationCard';
  
  // good
  import ReservationCard from './ReservationCard';
  
  // bad
  const ReservationItem = <ReservationCard />;
  
  // good
  const reservationItem = <ReservationCard />;
  ```

- 组件命名：将文件名作为组件的名字。比如 `ReservationCard.jsx` 中应该有一个叫做 `ReservationCard` 的引用名。但是，对于组件包含一个文件夹，使用 `index.jsx` 作为文件名，使用文件夹名字作为组件名：

  ```react
  // bad
  import Footer from './Footer/Footer';
  
  // bad
  import Footer from './Footer/index';
  
  // good
  import Footer from './Footer';
  ```

- 高阶组件命名：在生成的组件中，应该使用高阶组件名字和传入的组件的名字的组合作为`displayName`。 比如，对于高阶组件 `withFoo()` ，当传入的组件为 `Bar` 时，应该生成一个 `displayName` 是 `withFoo(Bar)` 的组件。

  > 为什么呢？组件的 `displayName` 可以被开发工具或错误信息中使用，通过这个可以清楚表达组件关系的值可以帮助开发人员理解到底发生了什么

  ```react
  // bad
  export default function withFoo(WrappedComponent) {
    return function WithFoo(props) {
      return <WrappedComponent {...props} foo />;
    }
  }
  
  // good
  export default function withFoo(WrappedComponent) {
    function WithFoo(props) {
      return <WrappedComponent {...props} foo />;
    }
  
    const wrappedComponentName = WrappedComponent.displayName
      || WrappedComponent.name
      || 'Component';
  
    WithFoo.displayName = `withFoo(${wrappedComponentName})`;
    return WithFoo;
  }
  ```

- Props 命名：比如使用 DOM 组件的 prop 名字用于不同的目的

  > 为什么呢？开发人员预期像 `style`  和 `className` 这样的 props  意味着特定的事情。在你的 app 子集中改变这种 API 会让你的代码可读性和可维护性更差，而且也可能会导致 bug

  ```react
  // bad
  <MyComponent style="fancy" />
  
  // bad
  <MyComponent className="fancy" />
  
  // good
  <MyComponent variant="fancy" />
  ```

#### 声明

- 在命名组件中不要使用 `displayName`。 应该通过引用命名。

  ```react
  // bad
  export default React.createClass({
    displayName: 'ReservationCard',
    // stuff goes here
  });
  
  // good
  export default class ReservationCard extends React.Component {
  }
  ```

#### 代码对齐

- 对于 JSX 语法遵循下面的对齐规则，eslint: [`react/jsx-closing-bracket-location`](https://github.com/yannickcr/eslint-plugin-react/blob/master/docs/rules/jsx-closing-bracket-location.md) [`react/jsx-closing-tag-location`](https://github.com/yannickcr/eslint-plugin-react/blob/master/docs/rules/jsx-closing-tag-location.md)

  ```react
  // bad
  <Foo superLongParam="bar"
       anotherSuperLongParam="baz" />
  
  // good 有多行属性的话, 新建一行关闭标签
  <Foo
    superLongParam="bar"
    anotherSuperLongParam="baz"
  />
  
  // 如果一行可以显示的话，将组件和参数放在同一行
  <Foo bar="bar" />
  
  // 子元素按照常规方式缩进
  <Foo
    superLongParam="bar"
    anotherSuperLongParam="baz"
  >
    <Quux />
  </Foo>
  
  // bad
  {showButton &&
    <Button />
  }
  
  // bad
  {
    showButton &&
      <Button />
  }
  
  // good
  {showButton && (
    <Button />
  )}
  
  // good
  {showButton && <Button />}
  ```

#### 引号

- 对于 JSX 属性使用双引号（`"`）, 其他的 JS 均使用单引号（`'`）。eslint: [`jsx-quotes`](https://eslint.org/docs/rules/jsx-quotes)

  > 为什么呢？常规的 HTML 属性通常使用双引号代替单引号，因此 JSX 的属性也遵循了这个约定

  ```react
  // bad
  <Foo bar='bar' />
  
  // good
  <Foo bar="bar" />
  
  // bad
  <Foo style={{ left: "20px" }} />
  
  // good
  <Foo style={{ left: '20px' }} />
  ```

#### 空格

- 在自闭合标签中包含单个空格。 eslint: [`no-multi-spaces`](https://eslint.org/docs/rules/no-multi-spaces), [`react/jsx-tag-spacing`](https://github.com/yannickcr/eslint-plugin-react/blob/master/docs/rules/jsx-tag-spacing.md)

  ```react
  // bad
  <Foo/>
  
  // very bad
  <Foo                 />
  
  // bad
  <Foo
   />
  
  // good
  <Foo />
  ```

- 在 JSX 的引用括号中间不要插入空格。eslint: [`react/jsx-curly-spacing`](https://github.com/yannickcr/eslint-plugin-react/blob/master/docs/rules/jsx-curly-spacing.md)

  ```react
  // bad
  <Foo bar={ baz } />
  
  // good
  <Foo bar={baz} />	
  ```

#### Props 属性

- 对于 prop 的名字应该采用驼峰命名

  ```react
  // bad
  <Foo
    UserName="hello"
    phone_number={12345678}
  />
  
  // good
  <Foo
    userName="hello"
    phoneNumber={12345678}
  />
  ```

- 在 prop 值为 true 时可以省略这个值。 eslint: [`react/jsx-boolean-value`](https://github.com/yannickcr/eslint-plugin-react/blob/master/docs/rules/jsx-boolean-value.md)

  ```react
  // bad
  <Foo
    hidden={true}
  />
  
  // good
  <Foo
    hidden
  />
  
  // good
  <Foo hidden />
  ```

- `<img>` 标签总要包含 `alt` 属性。如果是[装饰性的图片 presentational image](https://www.w3.org/WAI/tutorials/images/decorative/) , `alt` 可以是空字符串，或者 `<img>` 设置 `role="presentation"`。eslint: [`jsx-a11y/alt-text`](https://github.com/evcohen/eslint-plugin-jsx-a11y/blob/master/docs/rules/alt-text.md) 

  ```react
  // bad
  <img src="hello.jpg" />
  
  // good
  <img src="hello.jpg" alt="Me waving hello" />
  
  // good
  <img src="hello.jpg" alt="" />
  
  // good
  <img src="hello.jpg" role="presentation" />
  ```

- 在 `<img>` 的 `alt` 属性中不要使用类似 "image", "photo", or "picture" 等单词。eslint: [`jsx-a11y/img-redundant-alt`](https://github.com/evcohen/eslint-plugin-jsx-a11y/blob/master/docs/rules/img-redundant-alt.md)

  > 为什么？屏幕阅读器已经强调了 `<img>` 是图片，不需要在 alt 中包含相关的信息

  ```react
  // bad
  <img src="hello.jpg" alt="Picture of me waving hello" />
  
  // good
  <img src="hello.jpg" alt="Me waving hello" />
  ```

- 只使用有效的，非抽象的 [ARIA roles](https://www.w3.org/TR/wai-aria/#usage_intro). eslint: [`jsx-a11y/aria-role`](https://github.com/evcohen/eslint-plugin-jsx-a11y/blob/master/docs/rules/aria-role.md)

  ```react
  // bad - not an ARIA role
  <div role="datepicker" />
  
  // bad - abstract ARIA role
  <div role="range" />
  
  // good
  <div role="button" />
  ```

- 在元素中不要使用 `accessKey` 。 eslint: [`jsx-a11y/no-access-key`](https://github.com/evcohen/eslint-plugin-jsx-a11y/blob/master/docs/rules/no-access-key.md)

  >为什么? 屏幕阅读器在键盘快捷键与键盘命令时造成的不统一性会导致阅读性更加复杂

  ```react
  // bad
  <div accessKey="h" />
  
  // good
  <div />
  ```

- 避免使用数组的索引作为 `key` 参数，更倾向于使用稳定的 ID。eslint: [`react/no-array-index-key`](https://github.com/yannickcr/eslint-plugin-react/blob/master/docs/rules/no-array-index-key.md)

  > 为什么？不使用稳定的 id 是一种 [反模式](https://medium.com/@robinpokorny/index-as-a-key-is-an-anti-pattern-e0349aece318) , 因为这样会影响性能，也会导致组件状态异常

  如果这个列表的顺序会改变的话，不推荐使用键的索引。

  ```react
  // bad
  {todos.map((todo, index) =>
    <Todo
      {...todo}
      key={index}
    />
  )}
  
  // good
  {todos.map(todo => (
    <Todo
      {...todo}
      key={todo.id}
    />
  ))}
  ```

- 对于非必须的参数 props，始终定义默认参数 defaultProps

  > 为什么？propTypes 是某种形式的文档，提供默认参数 defaultProps 意味着不需要假定太多。而且，这也意味着你的代码可以省略特定的类型检查。

  ```react
  // bad
  function SFC({ foo, bar, children }) {
    return <div>{foo}{bar}{children}</div>;
  }
  SFC.propTypes = {
    foo: PropTypes.number.isRequired,
    bar: PropTypes.string,
    children: PropTypes.node,
  };
  
  // good
  function SFC({ foo, bar, children }) {
    return <div>{foo}{bar}{children}</div>;
  }
  SFC.propTypes = {
    foo: PropTypes.number.isRequired,
    bar: PropTypes.string,
    children: PropTypes.node,
  };
  SFC.defaultProps = {
    bar: '',
    children: null,
  };
  ```

- 尽量少地传递扩展运算符

  >  为什么？除非你想给组件传递不必要的属性。对于 React v15.6.1 及以前的版本，你可以[传递无效的 HTML 属性给 DOM](https://reactjs.org/blog/2017/09/08/dom-attributes-in-react-16.html)

  例外情况：

  - HOCs that proxy down props and hoist propTypes

    ```react
    function HOC(WrappedComponent) {
      return class Proxy extends React.Component {
        Proxy.propTypes = {
          text: PropTypes.string,
          isLoading: PropTypes.bool
        };
    
        render() {
          return <WrappedComponent {...this.props} />
        }
      }
    }
    ```

  - 在 props 明确时可以使用扩展运算符。这在使用 Mocha 的 beforeEach 构造去测试组件时非常有用

    ```react
    export default function Foo {
      const props = {
        text: '',
        isPublished: false
      }
    
      return (<div {...props} />);
    }
    ```

    使用提醒：过滤掉不必要的 props。同时，使用 [prop-types-exact](https://www.npmjs.com/package/prop-types-exact) 去帮助预防 bug

    ```react
    // bad
    render() {
      const { irrelevantProp, ...relevantProps } = this.props;
      return <WrappedComponent {...this.props} />
    }
    
    // good
    render() {
      const { irrelevantProp, ...relevantProps } = this.props;
      return <WrappedComponent {...relevantProps} />
    }
    ```

#### Refs

- 始终使用 ref 回调。eslint: [`react/no-string-refs`](https://github.com/yannickcr/eslint-plugin-react/blob/master/docs/rules/no-string-refs.md)

  ```react
  // bad
  <Foo
    ref="myRef"
  />
  
  // good
  <Foo
    ref={(ref) => { this.myRef = ref; }}
  />
  ```

#### 括号

- 将超过一行的 JSX 标签包装在括号中。eslint: [`react/jsx-wrap-multilines`](https://github.com/yannickcr/eslint-plugin-react/blob/master/docs/rules/jsx-wrap-multilines.md)

  ```react
  // bad
  render() {
    return <MyComponent variant="long body" foo="bar">
             <MyChild />
           </MyComponent>;
  }
  
  // good
  render() {
    return (
      <MyComponent variant="long body" foo="bar">
        <MyChild />
      </MyComponent>
    );
  }
  
  // good, when single line
  render() {
    const body = <div>hello</div>;
    return <MyComponent>{body}</MyComponent>;
  }
  ```

#### Tags 标签

- 在没有子元素的情况下使用自闭合标签。eslint: [`react/self-closing-comp`](https://github.com/yannickcr/eslint-plugin-react/blob/master/docs/rules/self-closing-comp.md)

  ```react
  // bad
  <Foo variant="stuff"></Foo>
  
  // good
  <Foo variant="stuff" />
  ```

- 如果组件有多行属性，闭合标签另起一行。eslint: [`react/jsx-closing-bracket-location`](https://github.com/yannickcr/eslint-plugin-react/blob/master/docs/rules/jsx-closing-bracket-location.md)

  ```react
  // bad
  <Foo
    bar="bar"
    baz="baz" />
  
  // good
  <Foo
    bar="bar"
    baz="baz"
  />
  ```

#### 函数

- 在仅处理局部变量时使用箭头函数。在需要传递额外的数据到事件处理器时是十分方便的。要确保没有 [严重影响性能](https://www.bignerdranch.com/blog/choosing-the-best-approach-for-react-event-handlers/) ， 特别是传递箭头函数给可能是纯组件的自定义组件时，因为这可能会导致不必要的每次都重新渲染。

  ```react
  function ItemList(props) {
    return (
      <ul>
        {props.items.map((item, index) => (
          <Item
            key={item.key}
            onClick={(event) => doSomethingWith(event, item.name, index)}
          />
        ))}
      </ul>
    );
  }
  ```

- 在构造函数中为渲染的内容绑定事件处理方法。eslint: [`react/jsx-no-bind`](https://github.com/yannickcr/eslint-plugin-react/blob/master/docs/rules/jsx-no-bind.md)

  > 为什么？在 render 过程中每次调用 bind 都会创建一个新的方法。在 class 字段中不要使用箭头函数，因为这会让 [测试和调试变得富有挑战，并且严重影响性能](https://medium.com/@charpeni/arrow-functions-in-class-properties-might-not-be-as-great-as-we-think-3b3551c440b1) ，并且从概念上，class 字段是用于表示数据，而不是逻辑。

  ```react
  // bad
  class extends React.Component {
    onClickDiv() {
      // do stuff
    }
  
    render() {
      return <div onClick={this.onClickDiv.bind(this)} />;
    }
  }
  
  // very bad
  class extends React.Component {
    onClickDiv = () => {
      // do stuff
    }
  
    render() {
      return <div onClick={this.onClickDiv} />
    }
  }
  
  // good
  class extends React.Component {
    constructor(props) {
      super(props);
  
      this.onClickDiv = this.onClickDiv.bind(this);
    }
  
    onClickDiv() {
      // do stuff
    }
  
    render() {
      return <div onClick={this.onClickDiv} />;
    }
  }
  ```

- React 组件的内部方法不要使用下划线前缀

  > 为什么？下划线前缀在其他语言表示私有属性和方法。但是，和其他语言不同是，在 JavaScript 中没有私有变量的原生支持，所有东西都是共有的。不管你的意图是什么，加上下划线前缀不能让你的属性变为私有的，所有的属性（不管有没有加上下划线前缀）都应该被作为共有看待。深入讨论可以查看  [#1024](https://github.com/airbnb/javascript/issues/1024),  [#490](https://github.com/airbnb/javascript/issues/490)

  ```react
  // bad
  React.createClass({
    _onClickSubmit() {
      // do stuff
    },
  
    // other stuff
  });
  
  // good
  class extends React.Component {
    onClickSubmit() {
      // do stuff
    }
  
    // other stuff
  }
  ```

- `render()` 方法中一定要返回值。eslint: [`react/require-render-return`](https://github.com/yannickcr/eslint-plugin-react/blob/master/docs/rules/require-render-return.md)

  ```react
  // bad
  render() {
    (<div />);
  }
  
  // good
  render() {
    return (<div />);
  }
  ```

#### 生命周期

- 继承自 `class extends React.Component` 的生命周期函数：

  1. 可选的 `static` 方法
  2. `constructor`
  3. `getChildContext`
  4. `componentWillMount`
  5. `componentDidMount`
  6. `componentWillReceiveProps`
  7. `shouldComponentUpdate`
  8. `componentWillUpdate`
  9. `componentDidUpdate`
  10. `componentWillUnmount`
  11. *点击回调或事件处理器* 比如`onClickSubmit()` 或 `onChangeDescription()`
  12. *render 中的 getter 方法* 比如 `getSelectReason()` 或 `getFooterContent()`
  13. *可选的 render 方法* 比如 `renderNavigation()` 或 `renderProfilePicture()`
  14. `render`

- 怎样定义 `propTypes`, `defaultProps`, `contextTypes` 等

  ```react
  import React from 'react';
  import PropTypes from 'prop-types';
  
  const propTypes = {
    id: PropTypes.number.isRequired,
    url: PropTypes.string.isRequired,
    text: PropTypes.string,
  };
  
  const defaultProps = {
    text: 'Hello World',
  };
  
  class Link extends React.Component {
    static methodsAreOk() {
      return true;
    }
  
    render() {
      return <a href={this.props.url} data-id={this.props.id}>{this.props.text}</a>;
    }
  }
  
  Link.propTypes = propTypes;
  Link.defaultProps = defaultProps;
  
  export default Link;
  ```

-  `React.createClass` 的生命周期: eslint: [`react/sort-comp`](https://github.com/yannickcr/eslint-plugin-react/blob/master/docs/rules/sort-comp.md)

  1. `displayName`
  2. `propTypes`
  3. `contextTypes`
  4. `childContextTypes`
  5. `mixins`
  6. `statics`
  7. `defaultProps`
  8. `getDefaultProps`
  9. `getInitialState`
  10. `getChildContext`
  11. `componentWillMount`
  12. `componentDidMount`
  13. `componentWillReceiveProps`
  14. `shouldComponentUpdate`
  15. `componentWillUpdate`
  16. `componentDidUpdate`
  17. `componentWillUnmount`
  18. *点击回调或事件处理器* 比如`onClickSubmit()` 或 `onChangeDescription()`
  19. *render 中的 getter 方法* 比如 `getSelectReason()` 或 `getFooterContent()`
  20. *可选的 render 方法* 比如 `renderNavigation()` 或 `renderProfilePicture()`
  21. `render`

#### `isMounted`

- 不要使用`isMounted`。eslint: [`react/no-is-mounted`](https://github.com/yannickcr/eslint-plugin-react/blob/master/docs/rules/no-is-mounted.md)

  > 为什么？ [`isMounted` 是反模式](https://facebook.github.io/react/blog/2015/12/16/ismounted-antipattern.html) ，在 ES6 的 classes 中不可用。而且处在被官方废弃的路上。

 













