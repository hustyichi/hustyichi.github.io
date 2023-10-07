---
layout: post
title: 'medical-labelme 升级方案概述'
subtitle:   "Overview of the medical-labelme upgrade plan"
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
在前面的文章中介绍了基于 Labelme 改造的 [medical-labelme](https://hustyichi.github.io/2023/09/23/medical-labelme/)，通过第一阶段的改造，最终保持对 Labelme 的向前兼容的情况下有限支持了 DICOM 文件的标注，最终发布了 [v0.1.0](https://github.com/hustyichi/medical-labelme/releases/tag/v0.1.0)。

为了解决 v0.1.0 版本对 DICOM 多帧图像支持不足的问题，本次从底层开始进行深层次的改造，但是也被迫放弃了与 Labelme 的完全兼容。Labelme 标注输出的结构与单帧图像绑定较深，对 DICOM 这种多帧图像的支持不足，未来存在明显的可拓展性问题。考虑到最终肯定是需要调整的，早日调整避免后续的向前不兼容的阵痛。

本次改造的最大底层变化就是使用 `ImageLabel` 替代了原有的 `LabelFile` 结构，带来如下所示的好处：

1. 原生支持多帧图像的存储，避免了帧切换需要重复解析文件的问题，效率大幅提升；
2. 对多帧的标注更加符合生产环境的需要，避免多帧 DICOM 文件不同帧的标记需要独立存储，多帧图像的标注文件更容易解析；

另外修复了 Labelme 中存在的一些隐藏问题，比如同名文件标注错乱，测试后发布 [v0.2.0](https://github.com/hustyichi/medical-labelme/releases/tag/v0.2.0)。本文就是对其中所做的改造具体介绍。

## 改造介绍
#### 基础介绍
本次改造主要的变化是针对底层数据结构 `LabelFile`，先分析下 `LabelFile` 在 Labelme 中承担的职责：

1. 提供图像解析的能力，将图像文件解析为 bytes 数据；
2. 提供加载标注文件并还原为标注后图像的能力；
3. 提供保存图像与标注信息至标注文件的能力；

原有的 `LabelFile` 存在如下所示的问题：

1. 提供的图像解析能力没有拓展性，直接返回 bytes 数据，导致无法支持多帧数据；
2. 不具备状态维护的能力，导致帧的切换只能依赖应用层多次调用，重复加载文件导致效率不高；
3. 加载标注文件与加载图像逻辑没有统一，加载图像直接返回给应用层，加载标注文件会在对象内部维护相关状态；
4. 标注文件的保存完全依赖上层业务提供数据组装能力，单个职责需要不同模块支持，封装不够漂亮；

针对现有问题，简单的改造无法根治这些问题，因此直接提供了新的数据结构 `ImageLabel`，用于替换 `LabelFile`，并增加辅助基础结构 `FrameLabel`，承担角色如下：

- `FrameLabel` 提供对单帧数据的支持，内部维护图像帧中的状态，包括图像帧的数据以及图像帧上的标记信息；
-  `ImageLabel` 提供对图像文件与标记帧的支持，内部维护图像加载后的状态，包括图像帧列表的数据以及标记文件的信息；

#### FrameLabel 实现
`FrameLabel` 的主要承担图像帧数据存储以及帧上标注管理的能力，关键在于标注的管理，下面重点介绍这部分的改造

 `LabelFile` 在实际标注时使用 `Shape` 结构表示标记数据，但是加载标记文件时只会还原为 dict 数据，导致应用层需要承担必要的转换职责。

本次首先增强了 `Shape` 结构，将格式化与还原的能力加入其中，提供 `Shape.format()` 用于格式化，提供 `make_shape()` 方法用于利用格式化数据还原 `Shape` 对象，这部分的实现在 `labelme/shape.py` 对应的文件中。

```python
def format(self) -> dict:
    data = self.other_data.copy()
    data.update(
        dict(
            label=self.label.encode("utf-8") if PY2 else self.label,
            points=[(p.x(), p.y()) for p in self.points],
            group_id=self.group_id,
            description=self.description,
            shape_type=self.shape_type,
            flags=self.flags,
            frame=self.frame,
        )
    )
    return data
```

在此基础上，`FrameLabel` 的实现就比较简单了，内部管理 `Shape` 列表，提供必要的增删能力。同时提供统一的帧格式化的能力，方便上层 `ImageLabel` 使用统一的格式进行标注信息的输出，对应实现如下：

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

1. 使用 `FrameLabel` 列表维护图像文件对应的多帧信息；
2. 使用 `current_frame` 作为标记确定当前帧，切换帧时只需要修改对应的标记，然后获取对应帧的图像数据即可。

通过这部分的改造，帧切换时需要重复加载的问题就得到了解决。

之前图像的加载存在明显问题，直接返回图像 bytes 数据导致完全无法拓展至多帧情况。因此本次进行了图像加载的封装优化，将 DICOM 文件与普通图像的解析进行了统一封装，解析图像文件返回迭代器对象：

```python
def load_image(image_path: str):
    if is_dicom_file(image_path):
        yield from load_dicom_file(image_path)
    else:
        yield from load_common_image(image_path)
```

通过调用上面的方法，`ImageLabel` 会将完整的帧数据维护在对象内部，避免单次调用仅能获取一帧图像数据的问题。

之前加载图像与标注文件不统一的问题，本次也进行了必要优化，`ImageLabel` 始终会维护相关状态。加载图像是会加载图像本身的数据维护在对象内部，而加载标注文件则被封装为 加载图像 + 加载历史标注信息，保证整体的统一性。实现如下所示：

```python
# 加载图像

def load_image(self, image_path: str):
    frame = 0

    # 按帧加载图像，保存为 FrameLabel 列表进行管理

    for image_pil in utils.load_image(image_path):
        self.frame_labels.append(
            FrameLabel(
                image_path,
                image_pil,
                frame=frame,
                app_config=self.app_config,
            )
        )
        frame += 1

    self.image_path = image_path
    self.total_frame = len(self.frame_labels)

# 加载标注文件

def load_label_file(self, label_path: str):
    with open(label_path, "r") as f:
        data = json.load(f)
        relative_image_path = data.pop("imagePath")

    self.label_path = label_path
    image_path = osp.join(osp.dirname(label_path), relative_image_path)

    # 调用统一的加载图像的方法进行图像数据的加载

    self.load_image(image_path)

    # 加载历史标注数据

    frame_shapes_data = data.pop("frames", [])
    for frame_data in frame_shapes_data:
        frame_idx = int(frame_data.get("frame", -1))

        # 确定对应的图像帧，在对应帧上加载标注数据

        self.frame_labels[frame_idx].load(frame_data)


# 统一的文件加载方法

def load(self, file_path: str):
    if utils.is_supported_image(file_path):
        self.load_image(file_path)
    elif ImageLabel.is_label_file(file_path):
        self.load_label_file(file_path)
    else:
        raise LabelFileError("Not support file type")


```

通过上面的改造，`ImageLabel` 成为一个具备维护图像帧与标注状态的基础模块，业务层可以将大量散乱在各个组件中的状态封装在 `ImageLabel` 内部进行管理，并将所有的加载与输出的功能进行了统一管理。

## 总结
本文主要介绍了 medical-labelme v0.1.0 中为了保证向前兼容遗留的一些问题，以及 v0.2.0 中如何通过对基础模块 `LabelFile` 升级为  `ImageLabel` 解决现有问题，从而实现对多帧 DICOM 文件更好的支持，也为未来的持续拓展打下基础。
