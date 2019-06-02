---
layout: post
title: 'This 关键字'
subtitle:   "Keyword this"
date:       2019-06-02 14:39:00
author:     "Bryan"
header-mask: 0.3
catalog:    true
tags:
    - frontend
---

## 基础介绍

在 javascript 中，this 是一个经常会需要碰到的关键字，但是实话实说，在 javascript 中，this 关键字没有那么容易理解，一不小心就会被误用。周末无事，梳理了这个关键字，希望之后可以避免踩坑。

## 基础规则

this 关键字指向的位置看起来比较诡异，但是，只有一个最基础的原则就是：**this 关键字指向的是 this 所属的对象** 。虽然不同情况下看起来 this 指向不同，但是了解这个基础原则，就能解决大部分的问题了。下面具体来介绍不同的情况下的 this 指向：

#### 默认 this 指向全局对象

在 javascript 默认存在全局对象 window，如果没有在其他对象内部，默认的 this 就指向全局对象 window，可以通过下面的代码进行验证：

```javascript
this === window;  // true
var a = 12;
this.a;          // 12
window.a;        // 12
```

#### 默认函数内部 this 指向全局对象

前面介绍了，this 关键字指向 this 所属的对象。如果函数没有包含在其他类内部，那么此函数属于全局对象，此时 this 指向的是全局对象。可以通过如下所示的代码进行验证：

```javascript
var b = 34;
function myFunc() {
  this === window;     // true
  this.b;              // 34
}
myFunc();
```

#### 类对象中方法内的 this 执行类对象

对于类对象中的方法内部，如果使用了 this ，此时 this 指向的是类对象。可以通过如下所示的代码进行验证：

```javascript
var c = 56;
var obj = {
  c: 78,
  func: function() {
    this === window;  // false
    this.c;						// 78
  }
}
Obj.func();
```

理解上面这些足够解决一般情况下的 this 指针的问题了，对于动态变化的情况，则需要注意：**this 关键字指向的是实际执行时所属的对象** ，比如下面的代码：

```javascript
var obj = {
  d: 10,
  func: function(){
    return this.d;
  }
}
var obj2 = {
  d: 20
}
obj2.func = obj.func;
obj2.func();         // 20
```

上面的例子中，将 `obj.func()` 方法赋值给 `obj2.func` ，此时需要注意，this 指向是实际执行时所属的对象。因此在 `obj2.func()` 内部的 this 指向的是 obj2 对象，因此此时值为 20

因此可以看到 this 指向的是实际执行的环境中对象，可以理解为运行时的对象上下文。

## 补充情况

在上面的介绍中，已经介绍了基础的 this 指向规则，下面对一些额外的情况下进行介绍：

#### 严格模式下的 this 指向

在上面的介绍中可以看到，在全局的函数中，this 指向的是全局对象，这样容易导致在函数内部错误地修改了全局对象，为了避免这个问题，在严格模式下，函数内部的 this 指向的是 undefined，而不是全局对象 window。具体验证代码如下：

```javascript
function strictFunc(){
  "use strict";
  this === window;     // false
  this;								 // undefined
}
```

#### 构造函数 this 指向

javascript 与其他语言的类设计有一些区别，javascript 的设计不是基于类的概念，而是基于构造函数与原型链的。javascript 设计了构造函数用于生成类对象。

构造函数与普通函数的定义类似，但是生成类对象时，需要使用 `new` 生成对象。使用如下所示：

```javascript
var People = function() {
  this.name = 'yichi';
};
var person = new People();
person.name;           // 'yichi'
```

通过上面的示例可以看到，构造函数与普通函数类似，主要的区别在于通过 `new` 生成对象，那么构造函数与普通函数有什么不同呢 ？

构造函数中的 this 指向的是通过构造函数生成的对象，而普通函数默认指向的是全局对象 window。可以通过下面的代码进行验证：

```javascript
var name = 'global';
var People = function(){
  this === window; 
  this.name;
}
var person = new People();
// false
// undefined
```

最终执行构造函数生成对象时，结果为 false 和 undefined，可以看到构造函数生成对象时，构造函数中的 this 不会指向全局对象，而是构造函数生成的对象

## 结论

在 javascript 中，this 是一个比较常用，但是又比较容易被误用的关键字。被坑了一次没有深入了解，下次还会继续被坑，为了避免持续被坑，早点深入了解 javascript 中的一个又一个大坑吧。