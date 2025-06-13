---
title: "STEP、IGES、STL等3D模型转换为GLB"
date: "2022-11-26"
tags: ['3D']
categories: ['动手实践']
---

在用threejs渲染3D模型时，往往需要选择一个最适合的模型格式，通常都是使用GLB作为Web渲染模型。然而许多工业的模型往往都是以STEP或者IGES作为导出格式，这种格式对于目前主流的3D渲染库支持并不好，所以需要转换模型格式。本篇文章为个人通过查找总结的转换方法，虽然并不是最优解🙃。

## 各种3D模型格式

### STEP & IGES

简单来说，这两种格式都是CAD的一种文件标准，在工业上使用比较广泛，STEP比IGES出现得更晚一些，由于IGES格式的最新版本是96年发布的，现在多由更高效的STEP等新格式替代，不支持材质。
IGES 可以安装 [iges viewer](https://igsviewer.com/download.aspx) 免费工具查看。

但是，现行主流的web3d库，比如 three.js、Babylon 均不支持 STEP 和 IGES 模型，需要解决这个问题有两个思路：

- 深入了解格式含义，编写代码给对应库提交对应的解析方案（想法很好，但是实践不易）
- 将格式转换为适合web展示的格式，比如称为3d界JPG的 GLTF 格式（本文就是讲这个的）

### STL

STL格式更多出现在3D打印中，只能用来表示封闭的体或者面，且文件内部都用三角形表示，所以转换精度比较粗的话，看起来效果比较诡异，包括 Ascii 编码和二进制两种编码模式，一般采用二进制，因为体积相对较小，并且与STEP和IGES一样不支持材质
比如同一个模型（STEP大小：4.81M），转换精度不同可能就是如下两种效果
粗精度（Ascii编码：3.7M）；细精度（Ascii编码：63.3M，二进制编码：12.1M）

### GLTF

简单来说，就是最小化的把模型资源整理起来，称为3d模型界的JPG，支持材质贴图等，在各个Web3D库中得到了广泛支持，具体怎么加载这里就不赘述了，网上demo很多

Github上的格式介绍和相关技术汇总: https://github.com/mrdoob/three.js/pull/14308

GLTF的详细介绍中文资料: https://zhuanlan.zhihu.com/p/65265611

## 格式转换

整体思路如下：
1. 使用 pythonocc 将 STEP/IGES/Ascii的STL文件 统一转换为二进制模式的 STL，
2. 再使用 stl2gltf 将STL转换转换为 gltf 格式，
3. 最后使用 gltf-pipeline 将glb文件压缩输出即可

### 环境安装

- pythonocc环境

    Pythonocc是python的CAD，安装和使用都很方便
    - 下载并安装AnaConda：https://www.anaconda.com/distribution/#download-section
    - 创建pythonocc 环境：`conda create -n pythonocct -c dlr-sc -c pythonocc pythonocc-core=7.4.0rc1`

- gltf-pipeline环境

    - 下载并安装nodejs：https://nodejs.org/zh-cn/
    - 安装gltf-pipeline：`npm install -g gltf-pipeline`

### 实现转换（方法1）

1. pythonocc读取转换STP/IGS/STL

- 在 Anaconda Prompt 中执行进入环境命令

`activate pythonocct`

- STP转换为 STL 文件（StpConverter.py）

```python
import os
from OCC.Extend.DataExchange import read_iges_file,read_step_file,write_stl_file

input_file = 'temp2.stp'
output_file = 'out.stl'
if not os.path.exists(input_file):
    print('Input file need exists')
    exit()

shapes=read_step_file(input_file)
write_stl_file(shapes, output_file, 'binary', 0.03, 0.5)
```

- IGES转换为 STL 文件（IgsConverter.py）

```python
import os
from OCC.Extend.DataExchange import read_iges_file,read_step_file,write_stl_file

input_file = 'temp2.stp'
output_file = 'out.stl'
if not os.path.exists(input_file):
    print('Input file need exists')
    exit()

shapes=read_step_file(input_file)
write_stl_file(shapes, output_file, 'binary', 0.03, 0.5)

```

2. stl2gltf（将STL转换为GLTF）

将二进制模式的 STL 文件转换为 GLTF 文件，支持浏览器本地转换、Python脚本以及C++源码

- [stl2gltf.py](https://github.com/MyMiniFactory/stl2gltf/blob/c%2B%2B/stl2gltf.py)
- 执行转换命令`python stl2gltf.py out.stl out.glb -b`

### 实现转换（方法2）

使用在线转换网址得到GLB文件([3D模型在线转换](http://www.3dwhere.com/conv))

### GLB文件压缩

直接转换出来的 glb 文件可能比较大，对于WEB来说还是太大了，需使用 gltf-pipeline 进行文件压缩

- 执行压缩命令`gltf-pipeline -i out.glb -o out.glb -b -d`
- [glb/gltf格式模型文件压缩–gltf-pipeline相关参数说明](https://www.icode9.com/content-4-892291.html)

> 本文参考自：https://www.codeleading.com/article/26425971329/