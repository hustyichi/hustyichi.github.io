---
layout: post
title: '新的医疗图像标注方案 medical-labelme'
subtitle:   "New medical image labeling project medical-labelme"
date:       2023-09-23 10:36:00
author:     "Bryan"
header-mask: 0.3
catalog:    true
tags:
    - python
    - DICOM
    - medical-labelme
---

## 背景信息
因为数据安全问题，医疗数据长期处于孤岛状态，为了解决相应的问题，隐私计算应运而生。但是如果希望将医疗影像用于机器学习算法训练，那么就需要将影像数据进行标注。

考虑到医疗影像中涉及大量的隐私信息，因此医疗影像的打标需要具备私有化部署的能力或离线运行的能力，常规图像的打标存在不少开源方案，比较热门的包括：

- [labelme](https://github.com/wkentaro/labelme) 11.1k star, 基于 QT 实现可视化
- [label-studio](https://github.com/HumanSignal/label-studio) 14.3k star, 基于 web 服务可视化，后端使用 Python
- [cvat](https://github.com/opencv/cvat) 10.1k star, 基于 web 可视化，后端使用 Python
- [labelImg](https://github.com/HumanSignal/labelImg) 20.6k star, 基于 QT 实现可视化，并入 label-studio，不再维护了

但是现有这些图像标注工具，基本都没有提供对医疗图像的 DICOM 的支持，因此都无法直接使用。对比了现有的几个项目，最终选择了持续更新的 labelme 项目作为基础，增加对医疗图像的支持，提供开箱即用的医疗图像标注工具 [medical-labelme](https://github.com/hustyichi/medical-labelme)。

## labelme 介绍
labelme 是起源于 [MIT](http://labelme.csail.mit.edu/Release3.0/) 的一个开源项目，目前 Github 上 star 11.1k 的项目 [labelme](https://github.com/wkentaro/labelme) 是由 wkentaro 维护的，medical-labelme 也是基于此项目进行扩展的。

![prev](/img/in-post/medical-labelme/prev.png)

labelme 是基于 QT 实现可视化，实现的功能比较轻量，导入的图像进行处理，输出 json 格式的标记文件。

labelme 的主界面位于 `labelme/app.py`，通过 `MainWindow` 类实现主要主界面的展示。

核心的组件为 `labelme/widgets/canvas.py`，主要实现画布的功能，其中基于 `QPixmap` 系统图像组件实现图像展示，并在图像上利用鼠标事件 `mousePressEvent` 持续确定用户点击的位置，最终形成自定义的标注区域。

## medical-labelme 介绍
labelme 对于常规图像的标注已经够用，但是本身不具备 DICOM 图像的支持能力，因此 medical-labelme 主要拓展了这部分能力。最终实现效果如下所示：

![new](/img/in-post/medical-labelme/new.png)

#### DICOM 解析
在 [之前的文章](https://hustyichi.github.io/2023/09/16/dicom/) 中对 DICOM 格式与解析进行了简要介绍，在 Python 中主要基于 [pydicom](https://pydicom.github.io/pydicom/stable/old/getting_started.html)，基于 pydicom 进行了 DICOM 图像的读取后，可以从 `pixel_array` 中读取对应的图像数据，此时会得到一个多维的 numpy 列表，一般格式为 `帧数 * 宽度 * 高度 * 通道数`

这个就会带来一个明确的问题，常规图像文件一般对应的都是单张图像，而 DICOM 实际可以对应多张图像，此时就需要进行额外处理。

#### 单帧 DICOM 支持
为了扩展 labelme 支持新图像格式，首先需要了解 labelme 是如何支持已有图像格式的，顺着文件导入的流程确定是通过 `labelme/label_file.py` 中的 `load_image_file()`实现的，原始的实现很简单，通过 `PIL.image` 加载图像，然后保存为字节数据。具体实现如下所示：

```Python
def load_image_file(filename):
    # 加载图像

    image_pil = PIL.Image.open(filename)

    image_pil = utils.apply_exif_orientation(image_pil)

    # 将原始图像数据转换为 IO 字节流

    with io.BytesIO() as f:
        ext = osp.splitext(filename)[1].lower()
        if PY2 and QT4:
            format = "PNG"
        elif ext in [".jpg", ".jpeg"]:
            format = "JPEG"
        else:
            format = "PNG"
        image_pil.save(f, format=format)
        f.seek(0)
        return f.read()
```

因此单帧的图像支持也比较简单，仅仅需要读取 DICOM 文件，之后获取对应的 `PIL.image` 加载后的对象即可。DICOM 读取的数据需要做必要的预处理，比如截距 tag 的处理与必要的正则化，后续才能与普通图像的处理流程一致，对应的实现如下所示：

```Python
def load_dicom_file(filename: str):
    # DICOM 文件中处理缩放截距，调整图像数据

    def handle_intercept(dicom_data: pydicom.FileDataset,
                         img_data: np.ndarray):
        if "RescaleIntercept" in dicom_data:
            img_data += int(dicom_data.RescaleIntercept)

        return img_data

    # 图像正则化，将原始数据转换为 0 ~ 255 的 rgb 数据

    def normalize_img(img_data: np.ndarray):
        min_, max_ = float(np.min(img_data)), float(np.max(img_data))
        normalize_img_data = (img_data - min_) / (max_ - min_) * 255
        trans_img_data = np.uint8(normalize_img_data)
        return trans_img_data

    #  读取 DICOM 图像

    dicom_data = pydicom.dcmread(filename)
    img = np.array(dicom_data.pixel_array).astype("float32")

    img = handle_intercept(dicom_data, img)
    img = normalize_img(img)

    image_pil = PIL.Image.fromarray(img)
    return image_pil

```

#### 多帧 DICOM 支持
多帧 DICOM 文件的则需要更复杂一些，因为原始的图像文件中只有只对应一张图片，因此 labelme 的解析 `load_image_file()` 直接返回图片对应的 IO 字节，对于多帧的情况，返回 IO 字节则对应的是多张图的 IO 字节数据，此时无法正确展示与解析。

为了解决这个问题，同时为了尽可能向前兼容，选择了扩展 `load_image_file()` 方法新增了额外参数 `frame`, 默认返回第一帧图像，这样返回值与之前就可以保持一致，用户可以在页面上多次调用 `load_image_file()` 获取不同帧的图像数据，并根据需要标记算法需要的关键帧。同时上层业务层需要获取帧总数才能保证最终不会超出现有帧数量，`load_image_file()` 需要返回总帧数。最终实现改造为如下所示：

```Python
def load_dicom_file(filename: str, frame: int = 0):
    # 预处理方法 handle_intercept 与 normalize_img 一样

    dicom_data = pydicom.dcmread(filename)
    img = np.array(dicom_data.pixel_array).astype("float32")

    # 获取 dicom 图像对应的全部帧数量，根据 frame 返回对应的帧的数据

    total_frame = utils.count_dicom_frame(dicom_data)
    if total_frame > 1:
        img = img[frame]

    img = handle_intercept(dicom_data, img)
    img = normalize_img(img)

    image_pil = PIL.Image.fromarray(img)
    return image_pil, total_frame


def load_image_file(filename: str, frame: int = 0):
    total_frame = 0

    # 根据类型使用不同的图像解析方式

    if utils.is_dicom_file(filename):
        image_pil, total_frame = LabelFile.load_dicom_file(filename, frame)
    else:
        image_pil = LabelFile.load_common_image_file(filename)
        total_frame = 1

    if not image_pil:
        return None, 0

    image_pil = utils.apply_exif_orientation(image_pil)

    # 将原始图像数据转换为 IO 字节流，与上面相同

```

实现了上面的能力后，应用层就可以获取总帧数，并可以通过调用 `load_image_file()` 并传递不同的 `frame` 实现帧切换。在 QT 页面上增加了对应的帧切换的按钮，在右下角增加了当前帧的信息，最终实现的效果如下所示：

![multi](/img/in-post/medical-labelme/multi.png)

在实际的标记输出中，也增加了标记帧的信息，这样模型训练时才能选择正确的关键帧。

## 总结
通过上面改造，使用尽可能少的改造，保证尽可能向前兼容的情况下，增加了对 DICOM 图像数据的支持。但是目前的版本是在很短时间内开发出来的简单版本，虽然基本可用，但是依旧存在不少优化空间，比如帧切换的过程中需要多次解析 DICOM 文件，效率不高，预期后续会对 medical-labelme 进行进一步的优化，从而更好地支持 DICOM 图像的标注。
