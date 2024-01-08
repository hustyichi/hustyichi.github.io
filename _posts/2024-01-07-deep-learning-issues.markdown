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
深度学习的威力众所周知，但其模型训练过程常伴随着繁琐的挑战。许多人在深度学习的实践中都曾遇到过各种问题，这些问题使得理想模型的训练变得更加复杂。本文将以理论结合实践的方式，探讨深度学习训练过程中常见的问题及相应解决方案，主要涉及以下几个方面：

- 梯度消失与梯度爆炸
- 收敛速度缓慢
- 过拟合

## 梯度消失与梯度爆炸
**是什么（What）**

深度学习中模型参数的更新主要使用 [梯度下降法](https://zh.wikipedia.org/wiki/%E6%A2%AF%E5%BA%A6%E4%B8%8B%E9%99%8D%E6%B3%95)，而基于梯度下降法的公式简化如下 `w -= η * ∂loss/∂w`，其中 `η` 为学习率常量，`∂loss/∂w` 就是 `w` 参数对应的梯度。

可以看出，训练过程中参数变化多少主要由梯度决定。理想情况下，我们期望训练达到收敛之前梯度值处于合理范围内，不会过小导致参数基本没有更新，也不会太大导致参数反复震荡。但是事实上这两种情况都会出现，对应的就是梯度消失和梯度爆炸。

> 梯度消失指的是在反向传播过程中，梯度逐渐变得非常小，甚至接近于零。这样的情况下，权重更新几乎不会发生，导致神经网络无法学到有效的特征表示。

> 梯度爆炸是指在反向传播中，梯度变得非常大，导致权重更新过大，网络变得不稳定。这可能导致权重值趋向于无穷大，使网络无法收敛或者产生不稳定的行为。

**为什么（Why）**

梯度消失和梯度爆炸的存在与 [链式法则](https://zh.wikipedia.org/wiki/%E9%93%BE%E5%BC%8F%E6%B3%95%E5%88%99) 的存在有很大的关系。梯度反向传播是从输出层向输入层传递的，在传递过程中，梯度逐层相乘，如果每层的梯度都小于 1，那么在靠近输入层的地方，梯度可能会变得非常接近于零，此时就会出现梯度消失。

同理而言，各层梯度值都比较大，那么相乘导致最终输入层的梯度就会过大，此时就会出现梯度爆炸。

**怎么办（How）**

对于梯度消失与梯度爆炸的研究比较久了，因此存在着各种各样的解决方案：

- 权重初始化，比如使用 Xavier 初始化和 He 初始化方法；
- 批量归一化（Batch Normalization），通过对每层的输入进行标准化，提高了网络的数值稳定性；
- 梯度裁剪（Gradient Clipping），通过设定一个阈值，当梯度的模大于这个阈值时，会将梯度裁剪到这个阈值以内；
- 改变激活函数，选择非饱和激活函数，如 ReLU 以及其变体；
- 使用 ResNet 残差网络；

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

## 收敛速度慢

**是什么（What）**

模型收敛是指继续训练模型的损失值不再继续下降，权重参数的变化也比较小，此时模型进入稳定状态。收敛速度慢情况下模型达到稳定状态的耗时过长。

**为什么（Why）**

收敛速度慢的原因比较多，常见的原因如下:

- 学习率设置不当：学习率设置不当可能导致训练过程不稳定或者收敛速度过慢。学习率太大可能导致震荡，而太小可能导致收敛速度缓慢；
- 权重初始化不当：初始值设置得不合适，导致需要训练更多轮才能收敛；
- 模型过于复杂：复杂的模型需要更多的数据用于训练，也会消耗更长的时间才能收敛；

**怎么办（How）**

- 调整学习率，避免过大或过小的学习率，如果模型损失值过于缓慢下降，可以加大学习率，如果模型损失值开始震荡甚至增加，考虑减小学习率；
- 权重初始化，通过使用合适的初始化方法，避免初始值初始不当问题，比如使用 Xavier 或 He 初始化；
- 使用更简单的模型架构；
- 使用 ResNet 残差网络；

**动手实践**

直接使用上面的神经网络模型 MLP，使用下面的方法生成对应的数据集并进行模型训练:

```python
import matplotlib.pyplot as plt
import torch
import torch.nn as nn
import torch.optim as optim

def generate_dataset(num_samples):
    torch.manual_seed(42)
    X = torch.rand(num_samples, 1)
    y = 3 * X + 2 + 0.1 * torch.randn(num_samples, 1)
    return X, y

def plot_loss(losses, learning_rate):
    plt.plot(losses)
    plt.xlabel('Epochs')
    plt.ylabel('Loss')
    plt.title(f'Training Loss Curve ({learning_rate})')
    plt.show()

def train(num_epochs, learning_rate):
    losses = []

    model = MLP(3, nn.ReLU)
    criterion = nn.MSELoss()
    optimizer = optim.SGD(model.parameters(), lr=learning_rate)

    X_train, y_train = generate_dataset(100)

    for epoch in range(num_epochs):
        model.train()
        y_pred = model(X_train)
        loss = criterion(y_pred, y_train)
        optimizer.zero_grad()
        loss.backward()
        optimizer.step()

        losses.append(loss.item())

        print(f'Epoch [{epoch + 1}/{num_epochs}], Loss: {loss.item():.4f}')

    plot_loss(losses, learning_rate)
    return losses
```

首先基于较小的学习率 0.001 进行训练，调用 `train(500, 0.001)` 进行训练，可以看到损失值下降如下图所示：

![small](/img/in-post/deep-learning-issues/small.png)

可以看到接近 200 个 Epochs 时模型开始收敛，收敛略慢

接下来将学习率增加至 0.1，可以看到损失值如下所示：

![large](/img/in-post/deep-learning-issues/large.png)

可以看到学习率过大导致损失下降后又开始上升，最后持续震荡，无法收敛

将学习率调整为 0.01 时，可以看到损失值如下所示：

![mid](/img/in-post/deep-learning-issues/mid.png)

可以看到接近 20 个 Epochs 时模型就开始收敛。


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

为了构造过拟合的情况，定义了一个模型参数较多的模型，模型参数如下所示：

```python
import torch
import torch.nn as nn

class OverfittingModel(nn.Module):
    def __init__(self):
        super(OverfittingModel, self).__init__()
        self.fc1 = nn.Linear(20, 100)
        self.fc2 = nn.Linear(100, 1)

    def forward(self, x):
        x = torch.relu(self.fc1(x))
        x = torch.sigmoid(self.fc2(x))
        return x

```

接下来构造分类数据集，其中包含 1000 条数据，构造数据集如下所示：

```python
import torch
from sklearn.datasets import make_classification
from sklearn.model_selection import train_test_split

def generate_dataset():
    X, y = make_classification(n_samples=1000, n_features=20, n_classes=2, random_state=42)
    X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, random_state=42)
    X_train = torch.tensor(X_train, dtype=torch.float32)
    y_train = torch.tensor(y_train, dtype=torch.float32)
    X_test = torch.tensor(X_test, dtype=torch.float32)
    y_test = torch.tensor(y_test, dtype=torch.float32)
    return X_train, y_train, X_test, y_test
```

基于上面的数据集与模型，执行 400 轮模型训练，接下来查看模型在训练集与测试集上的准确率，对应的实现如下所示：

```python
def train(weight_decay=0):
    model = OverfittingModel()
    criterion = nn.BCELoss()
    optimizer = optim.Adam(model.parameters(), lr=0.01, weight_decay=weight_decay)
    X_train, y_train, X_test, y_test = generate_dataset()

    epochs = 400
    for epoch in range(epochs):
        outputs = model(X_train)
        loss = criterion(outputs, y_train.view(-1, 1))
        optimizer.zero_grad()
        loss.backward()
        optimizer.step()

        if (epoch+1) % 10 == 0:
            print(f'Epoch [{epoch+1}/{epochs}], Loss: {loss.item():.4f}')

    # 在训练集和测试集上评估模型

    with torch.no_grad():
        train_accuracy = ((model(X_train) > 0.5).float() == y_train.view(-1, 1)).float().mean().item()
        test_accuracy = ((model(X_test) > 0.5).float() == y_test.view(-1, 1)).float().mean().item()

        print(f'Training Accuracy: {train_accuracy:.4f}, Testing Accuracy: {test_accuracy:.4f}')
```

基于上面的训练过程调用 `train()` 最终结果如下所示：

![overfit](/img/in-post/deep-learning-issues/overfit.png)

可以看到最终训练时在测试集上表现良好，损失值下降至 0.0007，而且准确率为 100%，但是在测试集上表现很差，准确率只有 81.5%。这是明显的过拟合的现象。

为了解决上面的问题，通过设置权重衰减（weight decay）来实现 L2 正则化，实际调用的值为 `train(weight_decay=0.04)`，最终结果如下所示：

![overfit_fix](/img/in-post/deep-learning-issues/overfit_fix.png)

可以看到增加正则化后在 100 轮左右实现了收敛，在训练接上准确率下降至 88.0%，但是在测试集上准确率提升至 87.5%, 过拟合的问题得到了很大的缓解。

## 总结
本文总结了深度学习训练过程中常见的几种问题以及常规的解决方案，并通过实践看到如何确定是否存在相关问题，以及解决方案带来的效果改善，希望帮助大家建立直接的了解。

深度学习是一门持续发展的学科，正是建立在很多人的研究基础上，深度学习才得以持续发展，一个又一个训练难题得到妥善的解决，最终我们能从几层神经网络，增加至几十层，几百层，最终得到类似 chatGPT 这样的庞然大物。致敬在这个过程中做出贡献的广大先驱们。
