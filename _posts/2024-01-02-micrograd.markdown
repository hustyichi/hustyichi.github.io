---
layout: post
title: '从 0 手撸一个 pytorch'
subtitle:   "Implementing a simple version of pytorch from 0"
date:       2024-01-02 09:20:00
author:     "Bryan"
header-mask: 0.3
catalog:    true
tags:
    - python
    - Machine learning
---

## 背景介绍
最近抽空看了下 Andrej Karpathy 的视频教程 [building micrograd](https://www.youtube.com/watch?v=VMj-3S1tku0)，教程的质量很高。教程不需要任何前置机器学习基础，只需要有高中水平的数学基础即可。整个教程从 0 到 1 手撸了一个类 pytorch 的机器学习库 [micrograd](https://github.com/karpathy/micrograd)，核心代码不到 100 行。虽然为了简化没有实现复杂的矩阵运算，但是对于理解 pytorch 的设计思想有很大帮助。

## 动手实践
为了验证 micrograd 的可用性，先基于 micrograd 实现了简单的线性回归算法。

首先构造出数据集，我使用随机数作为 x，通过线性回归确定结果后增加必要的噪声，对应的构造方法如下所示：

```python
import numpy as np

def get_train_dataset(num_samples, noise):
    x = np.random.rand(num_samples)

    y = 4 * x + 3 + np.random.normal(0, noise, num_samples)

    return x.tolist(), y.tolist()
```
可以看到最终期望的结果为 `y = 4 * x + 3`

接下来实现训练流程，线性回归的模型的初始值都使用随机值，持续跟踪训练过程中损失值与对应参数的变化，实现如下所示：

```python
import numpy as np
from micrograd.engine import Value

def zero_grad(w, b):
    w.grad = 0
    b.grad = 0

def step(w, b, learning_rate):
    w.data -= learning_rate * w.grad
    b.data -=  learning_rate * b.grad

def train_loop():
    dataset_x, dataset_y = get_train_dataset(10, 0.01)
    learning_rate = 0.1
    w = Value(np.random.rand())
    b = Value(np.random.rand())
    epoch = 40

    print(f"Init w {w.data}, b {b.data}")
    for idx in range(epoch):
        loss = 0

        for x, y in zip(dataset_x, dataset_y):
            x_value, y_value = Value(x), Value(y)
            y_pred = x_value * w + b
            current_loss = (y_value - y_pred) ** 2
            loss += current_loss.data

            zero_grad(w, b)
            current_loss.backward()
            step(w, b, learning_rate)

        print(f"Epoch {idx} got loss: {loss}, w {w.data}, b {b.data}")

```

上面的实现中使用 `zero_grad()` 方法重置参数的梯度，使用 `step()` 方法实际更新模型参数，训练流程就实现在 `train_loop()` 中。最终结果如下所示：

![micrograd](/img/in-post/micrograd/micrograd.png)

可以看到经过 40 轮训练后，损失值从最初的 55.69 下降至 0.0016，而参数 w, b 也接近期望的目标。从实践结果来看，micrograd 确实能实现简单模型的训练。

通过上面的实践来看，micrograd 最核心的就是 `Value`，按照 Andrej Karpathy 的说法，不到 100 行实现的 `Value` 就已经完成的 pytorch 中的 Tensor 90% 的功能了，除了这部分核心功能之外，pytorch 更多的是做了效率上的优化。

#### 流程梳理
在机器学习中，模型训练都是基于 [梯度下降](https://zh.wikipedia.org/wiki/%E6%A2%AF%E5%BA%A6%E4%B8%8B%E9%99%8D%E6%B3%95) 来更新模型的。模型训练的过程一般分为前向传播和反向传播：

- 前向传播会根据训练数据确定对应的损失值，对应于上面的实现如下：

```python
x_value, y_value = Value(x), Value(y)
y_pred = x_value * w + b
current_loss = (y_value - y_pred) ** 2
```

前向传播就是根据模型确定预测值 `y_pred`, 基于 MSE 确定损失值 `(y - y_pred)^2`。前向传播相对容易理解。

- 反向传播就是根据确定的损失值进行模型参数的调整，从而降低损失值，对应的实现就是：

```python
zero_grad(w, b)
current_loss.backward()
step(w, b, learning_rate)
```
上面最核心的功能就是调用 `current_loss.backward()` 确定各个参数对应的梯度，然后在 `step()` 方法中对参数的值进行更新。

参数更新的方案是相对明确，就是减去梯度与学习率之积实现。因此主要关注如何确定参数的梯度。梯度的计算存在如下所示的关注点：

1. 数学运算各个元素对应的梯度如何计算，这部分就是微积分中导数的计算；
2. 链式法则；
3. 复杂模型中包含上亿参数，如何确定参数各自的梯度；

## 实现细节
micrograd 最核心的实现位于 [engine.py](https://github.com/karpathy/micrograd/blob/master/micrograd/engine.py)，主要关注 `Value` 类的实现。

#### 初始化过程
关注初始化过程可以看到 Value 中包含的元素，实现如下：

```python
def __init__(self, data, _children=(), _op=''):
    self.data = data
    self.grad = 0
    self._backward = lambda: None
    self._prev = set(_children)
    self._op = _op # the op that produced this node, for graphviz / debugging / etc
```

初始化阶段可以看到 Value 中最重要的两个参数，`data` 保存的是元素中的原始数据，`grad` 保存的是当前元素对应的梯度。

`_backward()` 方法保存的是反向传播的方法，用于计算反向传播的梯度

`_prev` 保存的是当前节点前置的节点，比如 `y = w * x` 中节点 `y` 对应的 `_prev` 保存的是 `w` 和 `x`。通过不断的获取 `_prev` 节点，即可还原完整的运算链路。


#### 数学运算支持
`Value` 中支持了不同的数学运算，首先以加法为例，实现如下所示：

```python
def __add__(self, other):
    other = other if isinstance(other, Value) else Value(other)

    # 加法运算得到结果，同样是 Value 元素

    out = Value(self.data + other.data, (self, other), '+')

    # 加法反向传播函数

    def _backward():
        self.grad += out.grad
        other.grad += out.grad
    out._backward = _backward

    return out

```

前向传播计算的实现比较简单，直接基于 `data` 进行计算，通过加法运算生成了结果 `out`。同时将参与运算的元素 `self` 和 `other` 保存至 `self._prev` 中，方便还原运算链路。

`out` 对应的反向传播的方法 `_backward()` 是基于链式法则实现。举例如下：

c = a + b

那么 `∂l/∂a = ∂l/∂c * ∂c/∂a`，而 `∂c/∂a = 1`，因此 `∂l/∂a = ∂l/∂c`，因此加法中元素的梯度就等于其结果的梯度。

那么为什么实现是 `self.grad += out.grad` 而不是 `self.grad = out.grad` 呢，因为单个元素涉及多个运算链路时，梯度是不同链路确定的梯度之和。

这个也带来一个隐患，每次重新计算梯度之前，需要将原有的梯度重置为 0。对应于上面的 `zero_grad()` 的实现。了解 pytorch 应该也会注意到 pytorch 训练过程中也存在类似情况。

同样来查看乘法运算，对应的实现如下：

```python

def __mul__(self, other):
    other = other if isinstance(other, Value) else Value(other)
    out = Value(self.data * other.data, (self, other), '*')

    def _backward():
        self.grad += other.data * out.grad
        other.grad += self.data * out.grad
    out._backward = _backward

    return out
```

主要关注反向传播的实现，可以看到同样是链路法则的推演，举例如下：

c = a * b

那么 `∂l/∂a = ∂l/∂c * ∂c/∂a`，而 `∂c/∂a = b`, 因此 `∂l/∂a = ∂l/∂c * b`, 因此就可以理解上面的实现了。

#### 反向传播
通过上面的运算过程可以看到，通过不断保存其前置元素至 `self._prev` 中，可以构建出完整的运算链路图。而在运算过程中，元素反向传播计算的梯度的方法 `_backward()` 也被确定。因此反向传播就是从后往前调用 `_backward()` 来实现的：

```python

def backward(self):

    topo = []
    visited = set()
    # 根据前置元素的关系构建拓扑排序的元素列表，保证最终调用时是从后往前的

    def build_topo(v):
        if v not in visited:
            visited.add(v)
            for child in v._prev:
                build_topo(child)
            topo.append(v)
    build_topo(self)

    # 最后元素的梯度为 1, 依次计算前置元素的梯度

    self.grad = 1
    for v in reversed(topo):
        v._backward()
```

最终反向传播就是调用 `_backward()` 即可确定各个元素的梯度。


## 总结
通过上面的流程可以很容易理解机器学习模型训练框架的设计方案，这一套流程也完全适用于 pytorch，可以帮助更好地理解 pytorch 的训练流程。整体总结下实现思路：
1. 前向传播过程中会逐层计算运行结果，并确定结果与运算元素梯度之前的关系，在结果元素梯度确定后就可以确定运算元素的梯度；
2. 反向传播就是按照从后往前依次确认各个元素的梯度，方便后续根据梯度更新元素对应的值；
