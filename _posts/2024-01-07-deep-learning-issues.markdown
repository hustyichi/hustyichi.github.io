---
layout: post
title: '难以训练的深度学习'
subtitle:   "Difficult to train deep learning"
date:       2024-01-07 11:33:00
author:     "Bryan"
header-mask: 0.3
catalog:    true
tags:
    - python
    - Machine learning
---

## 背景介绍
接触过深度学习或多或少都知道，深度学习模型很强大，但是深度学习模型的训练也会更加麻烦，容易碰到各种各样的问题，导致难以训练出理想的模型。这篇文章使用理论结合实践的方式，介绍其中存在的常见问题与解决方案，主要涉及：

1. 梯度消失与梯度爆炸
2. 过拟合
3. 收敛速度慢
4. 模型加深与准确性下降

## 梯度消失与梯度爆炸
**是什么（What）**

深度学习中模型参数的更新主要使用 [梯度下降法](https://zh.wikipedia.org/wiki/%E6%A2%AF%E5%BA%A6%E4%B8%8B%E9%99%8D%E6%B3%95)，而基于梯度下降法的公式简化如下 `w -= η * ∂loss/∂w`，其中 `η` 为学习率常量，而 `w` 参数对应梯度就是 `∂loss/∂w`。

因此训练过程中参数变化多少主要由梯度决定。理想情况下，我们期望训练过程中梯度值是在合理范围内，不会过小导致参数基本没有更新，也不会太大导致参数反复震荡。但是事实上这两种情况都会出现，对应的就是梯度消失和梯度爆炸。

> 梯度消失指的是在反向传播过程中，梯度逐渐变得非常小，甚至接近于零。这样的情况下，权重更新几乎不会发生，导致神经网络无法学到有效的特征表示。

> 梯度爆炸是指在反向传播中，梯度变得非常大，导致权重更新过大，网络变得不稳定。这可能导致权重值趋向于无穷大，使网络无法收敛或者产生不稳定的行为。

**为什么（Why）**

梯度消失和梯度爆炸的存在与 [链式法则](https://zh.wikipedia.org/wiki/%E9%93%BE%E5%BC%8F%E6%B3%95%E5%88%99) 的存在有很大的关系。梯度反向传播是从输出层向输入层传递的，在传递过程中，梯度逐层相乘，因为每一层都需要计算上一层的梯度。如果这些梯度相乘的结果非常小，那么在靠近输入层的地方，梯度可能会变得非常接近于零，就会出现梯度消失。

同理而言，各层梯度值都比较大，那么相乘最终输入层的梯度就会过大，最终出现梯度爆炸。

**怎么办（How）**

对于梯度消失与梯度爆炸的研究比较久了，因此存在着各种各样的解决方案：

- 权重初始化，比如使用 Xavier 初始化和 He 初始化方法；
- 批量归一化（Batch Normalization），通过对每层的输入进行标准化，提高了网络的数值稳定性；
- 梯度裁剪（Gradient Clipping），通过设定一个阈值，当梯度的模大于这个阈值时，会将梯度裁剪到这个阈值以内；
- 改变激活函数，选择非饱和激活函数，如 ReLU 以及其变体；

**动手实践**

这边手工构造了一个类似 MLP 的神经网络模型，仅包含线性回归模型与激活函数，模型如下所示：

```python
import torch
import torch.nn as nn

class MLP(nn.Module):
    def __init__(self, num_layers, activation_func):
        super(MLP, self).__init__()

        layers = []
        for _ in range(num_layers):
            layers.append(nn.Linear(1, 1, bias=False))
            layers.append(activation_func())

        self.model = nn.Sequential(*layers)
        self.initialize()

    def forward(self, x):
        return self.model(x)

    def initialize(self):
        for m in self.modules():
            if isinstance(m, nn.Linear):
                nn.init.constant_(m.weight.data, 1.0)

```

接下来使用下面的方法进行训练并执行反向传播，使用激活函数为 `Sigmoid` 训练 30 层神经网络，对应如下所示：

```python
def train(num_layers, activation_func):
    model = MLP(num_layers, activation_func)
    output = model(torch.tensor([1.0]))
    loss = torch.sum(output)
    loss.backward()

    # 打印每一层的梯度

    for name, param in model.named_parameters():
        if param.requires_grad:
            print(f"{name} grad: {param.grad.item()}")

train(30, nn.Sigmoid)
```

反向传播后打印的梯度如下所示：

![prev](/img/in-post/deep-learning-issues/prev.png)

可以看到越靠近输出层的梯度越大，比如最后一层 `model.58.weight` 的梯度为 0.148，但是越靠近输入层梯度越小，比如第一层 `model.0.weight` 的梯度为 `2.99 * e-20`，基本接近 0，表现出梯度消失的特征。

上面的模型切换激活函数为 `ReLU` 后，执行的方法为 `train(30, nn.ReLU)` 对应的梯度如下所示:

![after](/img/in-post/deep-learning-issues/after.png)

可以看到始终保持在 1.0，梯度消失的现象没有再出现。

## 过拟合

**是什么（What）**

过拟合是指训练误差和测试误差之间的差距太大。一般是模型在训练集上表现很好，但在测试集上却表现很差。
模型对训练集"死记硬背"，没有理解数据背后的规律，泛化能力差。

**为什么（Why）**

过拟合的发生是因为模型过于复杂，以至于学习到了训练数据中的噪声和细节，而无法很好地泛化到新数据上。

**怎么办（How）**

在深度学习中解决过拟合的手段如下所示：

- 正则化：正则化是通过在损失函数中添加惩罚项来限制模型参数的大小，防止其过于复杂。常见的正则化方法包括L1正则化和L2正则化；
- Dropout：在训练过程中随机丢弃一些神经元，以减少模型对某些特定神经元的依赖，有助于防止过拟合；
- 更多的数据：提供更多的训练数据可以减轻过拟合，但是获取的数据的一般意味着更高的成本，所以一般使用数据增强手段增加数据；

**动手实践**




