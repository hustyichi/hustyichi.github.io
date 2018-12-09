---
layout: post
title: 'Less与Sass使用'
subtitle:   "less and sass"
date:       2018-12-09 13:22:00
author:     "Bryan"
header-mask: 0.3
catalog:    true
tags:
    - frontend
    - css
---

## 基础概念

有前端开发的小伙伴都或多或少接触过Sass和Less，由于CSS功能比较弱，代码的复用性比较弱，为了更方便地编写CSS，程序员们想到可以更方便的文件设计样式，然后再转换为CSS。这种方式就成为CSS预处理。而Less和Sass就是目前最流行的CSS预处理文件。

这两种方式实现的功能类似，都具备如下所示的基础功能增强：

- 变量，CSS不支持变量，导致相同的数值需要复制粘贴，采用Less或Sass就可以指定变量，重复的地方直接使用变量即可
- 混合，CSS无法实现样式代码的复用，使用Less和Sass可以复用Class的样式
- 继承，CSS无法实现样式的继承，而Less和Sass可以更好地继承另一个Class的样式
- 嵌套，Css中嵌套的指定比较麻烦，而且不直观，使用Less和Sass可以改善这种情况
- 运算，Css中不支持数值的运算，使用Less和Sass可以支持比较丰富的数学运算

## Less 语法

下面对Less相关功能的语法进行介绍：

#### 变量

Less使用`@` 字符指定变量，使用方式如下所示：

```less
@color: #4D926F;

h2 {
  color: @color;
}
```

#### 混合

Less可以将一个定义好的Class引入另一个Class中，从而达到样式的复用，同时支持参数的调用，类似其他语言中方法调用的效果，代码如下所示：

```less
.rounded-corners (@radius: 5px) {
  border-radius: @radius;
  -webkit-border-radius: @radius;
  -moz-border-radius: @radius;
}

#header {
  .rounded-corners;
}
#footer {
  .rounded-corners(10px);
}
```

在上面的代码中，`#header` 和`#footer` 都复用了`rounded-corners` 的样式，其中`#header` 使用的是默认值参数`5px`， 而`#footer` 使用指定的参数`10px` ，最终转换为CSS代码如下所示：

```less
#header {
  border-radius: 5px;
  -webkit-border-radius: 5px;
  -moz-border-radius: 5px;
}
#footer {
  border-radius: 10px;
  -webkit-border-radius: 10px;
  -moz-border-radius: 10px;
}
```

#### 继承

Less如果希望在Class中继承另一个Class的样式，可以采用与混合功能一样的实现方式，直接将被继承的样式直接加入即可，代码如下所示：

```less
.rounded-corners (@radius: 5px) {
  border-radius: @radius;
  -webkit-border-radius: @radius;
  -moz-border-radius: @radius;
}

.content {
    .rounded-corners;
    font-size: 20px;
}
```

#### 嵌套

在原始的CSS中，每个嵌套方式需要一一指定，导致写起来很繁琐，而且不直观，在Less中可以直接按照层次写即可。代码如下所示：

```less
#header {
  h1 {
    font-size: 26px;
    font-weight: bold;
  }
  p { font-size: 12px;
    a { text-decoration: none;
      &:hover { border-width: 1px }
    }
  }
}
```

而转换为CSS后，对应的代码如下所示：

```less
#header h1 {
  font-size: 26px;
  font-weight: bold;
}
#header p {
  font-size: 12px;
}
#header p a {
  text-decoration: none;
}
#header p a:hover {
  border-width: 1px;
}

```

可以看到在Less中，可以直接在`{}` 中增加样式即可表示层次关系，而且层次关系可以循环嵌套下去，而CSS中则相当繁琐，需要一层一层指定下去。

#### 运算

在Less中，可以直接对数值进行运算，大大方便开发者。代码如下所示：

```less
@the-border: 1px;
@base-color: #111;

#header {
  color: @base-color * 3;
  border-right: @the-border * 2;
}
```

## Sass 语法

Sass包含两种文本格式，一种是`.sass` 格式的，采用python类似的严格缩进，不使用`{}` 和 `；` 作为分隔符。一种是`.scss`，采用`{}` 和`；` 进行分隔，是标准的CSS语法。`.scss` 是Sass最新的默认格式。后续介绍也都以此格式进行介绍。

#### 变量

Sass使用`$` 字符指定变量，代码如下所示：

```scss
$color: #4D926F;

h2 {
  color: $color;
}
```

#### 混合

Sass中可以将一些公共样式进行复用，也支持指定参数。在使用中，需要使用`@mixin` 参数指定复用的样式，并使用`@include` 进行引用，代码如下所示：

```scss
@mixin rounded-corners ($radius: 5px) {
  border-radius: $radius;
  -webkit-border-radius: $radius;
  -moz-border-radius: $radius;
}

#header {
  @include rounded-corner();
}
#footer {
  @include rounded-corners(10px);
}
```

#### 继承

Sass中可以继承另外一些样式的代码，在Sass中可以使用`@extend` 继承样式，代码如下所示：

```scss
.rounded-corners {
  border-radius: 5px;
  -webkit-border-radius: 5px;
  -moz-border-radius: 5px;
}

.content {
    @extend .rounded-corners;
    font-size: 20px;
}
```

可以看到Less中混合和继承使用相同的实现方式，而Sass中继承和混合采用不同关键字进行实现，看起来功能比较类似，比较两者的区别可以参考[Gloria的博客](https://www.w3cplus.com/preprocessor/sass-mixin-or-extend.html) 

#### 嵌套

对于嵌套的语法，Sass与Less基本一致，实现的代码如下所示：

```scss
#header {
  h1 {
    font-size: 26px;
    font-weight: bold;
  }
  p { font-size: 12px;
    a { text-decoration: none;
      &:hover { border-width: 1px }
    }
  }
}
```

可以看到，与Less语法一致。

#### 运算

进行通用运算，基本的算法与Less也相同，代码如下所示：

```scss
$the-border: 1px;
$base-color: #111;

#header {
  color: $base-color * 3;
  border-right: $the-border * 2;
}
```

可以看到，基本的运算操作基本一致

## 总结

最后比较下来，Less和Sass都对CSS的痛点进行了修复，在一般场景下也都能很好地满足需求。从语法层面上看起来，Less的语法相对简单，而Sass的语法设计更严谨，也更复杂一些。因此，一般情况下，项目选择其中一种进行CSS的预处理都Ok，但是从市场上的风向来看，Sass看起来更受欢迎一些，之前一直使用Less的Bootstrap也切换到了Sass，因此，如果新项目的话，建议还是跟着市场走选择Sass，老项目的话也不必急着切换，Less也是够用的。