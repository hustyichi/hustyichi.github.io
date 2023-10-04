---
layout: post
title: 'medical-labelme 改造方案概述'
subtitle:   "Overview of the medical-labelme transformation program"
date:       2023-10-04 20:12:00
author:     "Bryan"
header-mask: 0.3
catalog:    true
tags:
    - python
    - DICOM
    - medical-labelme
---

## 背景信息
在前面的文章中介绍了基于 Labelme 改造的 [medical-labelme](https://hustyichi.github.io/2023/09/23/medical-labelme/)，通过第一阶段的改造，最终在保留了对 Labelme 的向前兼容的情况下有限支持了 DICOM 文件的标注，最终发布了 [v0.1.0](https://github.com/hustyichi/medical-labelme/releases/tag/v0.1.0)。

为了解决 v0.1.0 的对 DICOM 多帧图像支持不足的问题，本次从底层开始进行深层次的改造，但是也被迫放弃了与 Labelme 的完全兼容。考虑到 Labelme 输出的结构与单帧图像绑定较深，对 DICOM 这种多帧图像的支持不足，最终总是需要调整的。早日调整避免后续的向前不兼容的阵痛。

本次改造的最大底层变化就是使用 `ImageLabel` 替代了原有的 `LabelFile` 结构，带来如下所示的好处：

1. 原生支持多帧图像的存储，避免了帧切换需要重复解析文件的问题，效率大幅提升；
2. 对多帧的标注更加符合生产环境的需要，避免多帧 DICOM 不同帧的标记需要独立存储，多帧的标注输出也更容易解析；

另外修复了 Labelme 中存在的一些隐藏的问题，同名文件标注错乱问题，修复后发布 [v0.2.0](https://github.com/hustyichi/medical-labelme/releases/tag/v0.2.0)。

## 底层数据结构优化
本次改造主要的变化是去除了对底层数据结构 `LabelFile` 的依赖，先分析下 `LabelFile` 在其中承担的基础职责，主要包含两部分：

1. 提供图像解析的基础能力，将图像文件解析为 bytes 数据；
2. 提供加载标注文件并还原为标注后图像的能力；
3. 提供保存图像数据至标注文件的能力；

原有的 `LabelFile` 存在如下所示的问题：

1. 提供的图像解析能力没有拓展性，直接返回 bytes 数据，导致无法支持多帧数据；
2. 不具备持续状态维持的能力，导致帧的切换只能依赖上层依赖多次调用导致重复解析；
3. 加载标注文件与加载图像采取完全不同逻辑，没有建立统一逻辑；
4. 标注文件的保存完全依赖上层业务提供数据组装能力，单个职责由不同模块支持，缺乏内部封闭性；

针对现有问题，简单的改造无法根治这些问题，因此直接提供了新的数据结构 `ImageLabel`，用于替换 `LabelFile`，并增加辅助基础结构 `FrameLabel`，两者承担不同角色：

1. `FrameLabel` 提供对单帧数据的支持，内部维护图像帧中的状态，包括图像帧的数据以及图像帧上的标记信息；
2. `ImageLabel` 提供对图像文件与标记帧的支持，内部维护图像加载后的状态，包括图像帧列表的数据以及标记文件的信息；

#### FrameLabel 实现
`FrameLabel` 的主要承担图像帧数据存储以及帧上标注管理的能力，关键点在于标注的管理，下面重点介绍这部分的改造

 `LabelFile` 在实际标注时使用 `Shape` 结构表示标记数据，但是加载标记文件时只会还原为 dict 数据，导致处理逻辑不一致。因此首先增强了 `Shape` 结构，将格式化与还原的能力加入其中，提供 `Shape.format()` 用于格式化，提供 `make_shape` 方法用于利用格式化数据还原 `Shape` 对象，这部分的实现在 `labelme/shape.py` 对应的文件中。

`FrameLabel` 的实现就比较简单了，内部存储 `Shape` 列表，从而提供增删的能力，并可以进行必要的格式化能力，方便用于将帧数据存储至标记文件中，对应的实现如下：

```python
def format(self):
    return {
        "frame": self.frame,
        "shapes": [shape.format() for shape in self.shapes],
        "imageWidth": self.image_pil.width,
        "imageHeight": self.image_pil.height,
    }
```

#### ImageLabel 实现
`ImageLabel` 主要提供图像文件与标记帧的支持，相对 `LabelFile` 具备轻量切换帧的能力，主要的实现方案也很简单:

1. 使用 `FrameLabel` 列表维持文件对应的多个帧信息；
2. 使用 `current_frame` 维持当前帧标记，切换帧时只需要修改对应的标记，然后获取对应的帧图像数据即可。

通过这部分的改造，之前帧切换时需要重复加载的问题就得到了解决。


