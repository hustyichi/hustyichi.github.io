---
layout: post
title: 'React与Knockout共存探索'
subtitle:   "React and knockout"
date:       2018-07-16 20:11:00
author:     "Bryan"
header-mask: 0.3
catalog:    true
tags:
    - react
    - knockout
    - frontend
---

## 基础介绍
前端的框架一直变化很快，一个框架存活没多久，后续的新框架又火起来了。我们也总是需要与时俱进，不断更新前端的框架

- #### Knockout
[knockout](http://knockoutjs.com/index.html) 是一个MVVM的Javascript前端库，数据与展示采用双向绑定，使用比较便利。但是也是由于过度灵活，如果稍有不注意，写出来的代码就比较难以维护。

- #### React
[React](https://reactjs.org/) 是目前最火的前端框架之一，由Facebook创造。采用单向的数据流，入门比较容易。

公司之前使用Knockout的框架，目前希望能紧跟时代，向React迁移，但是之前的代码完全废弃又会导致业务停滞。因此，中间阶段就会出现Knockout与React共生的问题，希望新的代码采用React，但是必须与原有的Knockout代码兼容。之前有一些的Knockout的代码组件，也希望能继续使用，避免大量代码的重写

## React in knockout
在现有的knockout框架下，如果希望引入新的react组件，可以使用ReactDOM.render()轻松实现。利用此方法可以将React的组件转换为HTML，插入指定的DOM节点。示例代码如下：

```
  <div id="test_details">
  </div>
```
在需要插入处增加div, 指定id为test_details，方便后续查找与插入。具体使用可以任意指定

```
export default class reactContent extends React.PureComponent {
  render() {
    return <span>测试内容</span>
  }
}
```
创建React组件，在render()函数中增加所需的渲染内容，测试仅仅包含<span>，后续内容会被插入前面的div中

```
    var Page = require("path/to/react/page");
    
    reactDom.render(react.createElement(Page), document.getElementById("test_details"));
```
利用react.createElement()方法利用reactContent创建React元素，利用reactDom.render()方法将react元素转换为HTML，并插入到id为test_details的DOM下。

## Knockout in React
在使用React框架时，如果希望引入已有的knockout组件，也是可以实现的。
#### html引入
在React中，html与js是混合在一起的，但是在knockout框架下，html与js是分离的。因此如果希望复用原有的knockout组件，那么首先需要引入knockout组件中的html代码。可以使用dangerouslySetInnerHTML方法来实现，在上层通过ref进行标记，方便后续进行绑定，代码如下所示：

```
const binding_block_view = require('path/to/knockout/page.html');
<div ref={'ref_flag'}>
  <div dangerouslySetInnerHTML={ {'__html': binding_block_view}}>
  </div>
</div>
```
#### js绑定
在knockout中，需要将html与对应的js进行绑定，在React中也可以使用ko.applyBindings方法进行绑定。在使用结束之后再解绑。具体代码如下：

```
const binding_block = require('path/to/knockout/page.js');
componentDidMount() {
    ko.applyBindings(this.binding_block, this.refs.ref_flag);
}
componentWillUnmount() {
    ko.cleanNode(this.refs.ref_flag);
}
```
如果knockout组件不使用react组件的数据，而且react重新render时不会影响原有knockout组件的DOM结构，那么这种方式就可以完美工作了。但是如果knockout组件本身需要使用react组件传入的数据，或者是重新render时会影响knockout组件的DOM结构，那么就需要进行额外的处理。

原因在于react改变数据重新render时，由于传入的数据可能不是observable的，那么knockout组件获得的值是不会更新的，可能会导致knockout组件的展示不能正常更新。为了解决这个问题，需要在重新render之后通知knockout组件更新数据。一种解决方案如下：

```
componentDidUpdate() {
  ko.cleanNode(this.refs.ref_flag);
  ko.applyBindings(this.binding_block, this.refs.ref_flag);
}
```
即在每次重新render之后，先将原有的绑定解除，然后重新绑定即可。

对于需要在react页面使用data-bind的情况，之前已经有文章介绍过可行的办法，就不赘述了。可以参看 [Leland Richardson的博客](http://www.intelligiblebabble.com/making-reactjs-and-knockoutjs-play-nice)。
