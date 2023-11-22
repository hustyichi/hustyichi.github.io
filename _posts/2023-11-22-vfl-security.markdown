---
layout: post
title: '深入探索 FATE 纵向联邦学习安全方案'
subtitle:   "In-depth exploration of FATE vertical federated learning security solution"
date:       2023-11-22 17:40:00
author:     "Bryan"
header-mask: 0.3
catalog:    true
tags:
    - Federated learning
    - python
    - Machine learning
---

## 背景介绍
在之前的文章[FATE 纵向联邦学习实现探索](https://zhuanlan.zhihu.com/p/667637041) 介绍了纵向联邦神经网络训练的完整流程，在 [深入探索 FATE 纵向联邦学习模型设计方案](https://zhuanlan.zhihu.com/p/668221265) 详细介绍了纵向联邦神经网络的模型设计方案，但是对于如何在模型训练过程中保证模型的安全性基本都没有深入展开。这篇文章是纵向联邦神经网络的最后一篇，补上了安全性这一环。

## 安全协议
在开始具体的内容的介绍前，需要先了解同态加密以及经典的同态加密实现方案 [Paillier](https://en.wikipedia.org/wiki/Paillier_cryptosystem)。

同态加密是指满足同态性质的加密算法，即用户可以对加密的数据直接进行数学运算，加密数据运算的结果解密后与明文运算的结果是相等的。图示如下所示：

![security](/img/in-post/vfl-security/security.png)

为啥同态加密这么实用呢？可以设想一下，大部分情况下我们都不希望原始数据被其他人知道，但是在多方协作情况下经常会需要数据进行联合操作，但是常规加密的数据没办法进行任何操作，加密数据操作后就办法还原了。而同态加密就解决了这个问题，不需要泄露原始信息，多方就进行了联合计算，计算后的结果也是有意义的。因此同态加密在联邦学习中应用十分广泛。

Paillier 是一个经典而使用广泛的实现方案，实现的步骤在也可以比较容易找到，在 Python 语言中存在一个使用比较多的库 [python-paillier](https://github.com/data61/python-paillier)，有兴趣的可以深入了解下。

这边不深入细节，只需要了解下 Paillier 初始化时会生成公钥与私钥，公钥用于加密数据，私钥用于解密数据。FATE 的同态加密使用的就是 Paillier 机制。FATE 中多方协作时，所有的参与方都可以进行加密，只有一方可以解密，从现有的实现来看，一般是 Host （不存在标签数据的参与方）具备解密能力。

另外同态加密有一个广泛使用的标记符，一般使用 `[]` 进行表示。比如原始数据为 `a`, 其同态加密数据一般表示为 `[a]`

## FATE 安全机制

FATE 在纵向联邦神经网络中的训练过程过程中，个人从是实际的执行流程进行总结，其中使用了下面三种安全机制，在了解了下面的三种安全机制后，就可以了解 FATE 复杂的纵向联邦神经网络训练流程了。

#### 同态加密数据保护
这个安全机制比较直观，参与方为了避免泄露明文的数据，使用同态加密对明文数据进行加密，传递给其他参与方进行操作，然后对操作后的结果进行解密，从而实现在安全性进行数据操作。对应的流程如下所示：

![tech1](/img/in-post/vfl-security/tech1.png)

#### 增减噪声数据保护
发起方在传递数据前，额外增加随机噪声数据，接收放收到数据后，执行必要的操作（预期不会执行乘除法），然后将处理后的数据发送回来，发起来将接收到的结果减去增加的噪声，即可得到正确的结果。对应的流程如下所示：

![tech2](/img/in-post/vfl-security/tech2.png)


#### 累积噪声参数保护机制

这个技术方案相对比较复杂，而且需要在模型训练的前向传播与反向传播都需要进行必要的操作。





## FATE 安全训练流程
在 [FATE 官方文档](https://fate.readthedocs.io/en/latest/zh/federatedml_component/hetero_nn/) 中介绍了纵向联邦神经网络的前向传播与反向传播的完整流程，下面分别对这个流程进行介绍


![hetero_nn_forward_propagation](/img/in-post/vfl-security/hetero_nn_forward_propagation.png)

![hetero_nn_backward_propagation](/img/in-post/vfl-security/hetero_nn_backward_propagation.png)


