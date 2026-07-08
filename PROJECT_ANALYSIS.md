# 项目分析：CoColourPro

## 1. 项目概述

**ColorConsistency** 是一个基于 C++ / OpenCV 实现的图像拼接颜色一致性校正算法，主要用于解决多张拼接图像之间的色彩/亮度不一致问题。

该项目对应两篇论文：
- ICCV Workshop 2017: *Color Consistency Correction Based on Remapping Optimization for Image Stitching*
- ISPRS Journal 2019 (扩展版): *A Closed-Form Solution for Multi-View Color Correction with Gradient Preservation*

## 2. 目录结构

```
ColorConsistency/
├── README.md
├── Docs/demo_show.jpg              # 效果展示图
├── Data/
│   ├── Cache/ParamsPro.txt         # 图像邻接关系配置
│   └── Images/                     # 输入图像（10张tif全景图）
├── Source/
│   ├── CMakeLists.txt              # 顶层 CMake
│   ├── Unification/                # 主算法模块
│   │   ├── main.cpp                # 入口
│   │   ├── unification.h/.cpp      # ToneUnifier 核心类
│   │   └── correspondence.h/.cpp   # 直方图关键点匹配
│   ├── QuadProg/                   # 二次规划求解器
│   │   ├── QuadProg.h/.cpp         # Goldfarb-Idnani 对偶法
│   │   └── Array.h/.cpp            # 矩阵/向量封装
│   └── Utils/                      # 工具模块
│       ├── util.h/.cpp             # ROI、文件IO、结构差异计算
│       └── colorSpace.h/.cpp       # RGB↔YCbCr 颜色空间转换
└── *.rar 数据压缩包（CAMPUS/LUNCHROOM）
```

## 3. 核心算法流程

```
初始化
  ↓
加载邻接关系 (ParamsPro.txt) + 检测 ROI 掩膜
  ↓
提取相邻图像重叠区的颜色直方图关键点
  ↓
设置 B-Spline 控制点 + 计算插值系数
  ↓
对 Y/Cb/Cr 三个通道分别建立凸二次规划问题
  ↓
调用 Goldfarb-Idnani 对偶法求解
  ↓
用优化后的重映射曲线逐像素校正并输出拼接结果
  ↓
生成颜色差异与梯度保持评估报告
```

## 4. 关键技术点

### 4.1 能量函数设计

在 `unification.cpp:quadraticProgramming()` 中构造 QP 问题：

- **颜色一致性项（affinity）**：最小化相邻图像重叠区关键点经过映射后的差异
- **正则化项（regularization）**：让映射曲线尽量接近恒等映射，防止过度失真
- **梯度保持项（gradient preservation，仅亮度通道）**：保留主要梯度结构
- **对比度增强项（dynamic range extension）**：扩展亮度动态范围
- **硬约束（hard constraints）**：控制曲线斜率范围 `[0.3, 5]`，保证单调性与稳定性

### 4.2 曲线模型

- 采用 **B-Spline** 参数化颜色重映射曲线，每个通道每个图像设置 `PARAM_NUM = 6` 个控制点。
- 最终像素映射时进行 B-Spline 采样（`SAMPLE_FREQ_NUM = 15`）和线性插值。

### 4.3 二次规划求解

使用第三方的 `QuadProg++` 库（Goldfarb-Idnani active-set dual method）求解：

```cpp
QP::solveQuadraProgram(H, c, A.t(), -b, X);
```

其中 `H` 为 Hessian 矩阵，`c` 为一次项，`A/b` 为不等式约束。

### 4.4 颜色空间

- 在 `YCbCr` 空间进行处理，Y 通道单独处理以保留亮度一致性。
- `colorSpace.cpp` 提供完整的 RGB↔YCbCr 转换。

## 5. 输入数据格式

`Data/Cache/ParamsPro.txt` 每行格式：

```text
image_id  neighbor_num
neighbor_image_id ...
```

示例（10张图像，链式相邻）：

```text
0 1
1
1 2
0 2
...
```

相邻关系用于确定哪些图像之间存在重叠区域。

## 6. 运行方式

1. 编译项目：需要安装 **OpenCV 2.4.9**（推荐），使用 CMake 生成 VS 工程
2. 修改 `Source/Utils/util.h:16` 的 `baseDir` 为实际 Data 目录绝对路径
3. 创建 `Data/Results/` 目录
4. 运行生成的 `Unification.exe`

## 7. 项目特点

| 优点 | 说明 |
|------|------|
| 全局最优 | 将颜色校正建模为凸二次规划，保证全局最优解 |
| 多图像支持 | 可处理多张有邻接关系的拼接图像 |
| 梯度保持 | 亮度通道额外约束，减少细节丢失 |
| 对比度保持 | 动态范围扩展项，避免结果发灰 |

## 8. 潜在问题与改进点

1. **硬编码路径**：`baseDir` 写死在代码中，不便于移植。
2. **Windows 依赖**：`util.cpp` 使用 `FindFirstFile` 等 Windows API，跨平台需重写。
3. **OpenCV 2.x 老旧**：使用 `cv.h`、`opencv2/nonfree/nonfree.hpp` 等已废弃模块，在新版 OpenCV 中可能无法编译。
4. **MFC 依赖**：CMake 中设置了 `CMAKE_MFC_FLAG 2`，但代码本身并未使用 MFC。
5. **内存管理**：部分 `new vector<double>[3]` 使用 `delete []` 释放，但 `Mat` 数据指针直接强转使用，需注意内存对齐。
6. **无命令行参数**：图像路径、参数文件都固定，灵活性不足。
7. **未处理参考图**：`fixedImgNos` 为空，当前实现未指定参考图像，所有图像都参与优化。

## 9. 编译环境

- 原始开发环境：**Visual Studio 2010 + Windows 8.1**
- 构建工具：CMake ≥ 2.8.3
- 依赖库：OpenCV 2.4.9
- 可选：OpenMP 并行加速

## 10. 前置条件：图像对齐

**图像几何对齐是使用本项目的前提条件**，该项目本身不负责图像配准，只处理颜色后处理。

### 10.1 为什么必须对齐？

1. **README 明确要求**：输入图像必须是 "aligned via image stitching algorithms"（如 PanoramaStudio 或 AutoStitching）。
2. **ROI 掩膜基于对齐结果生成**：
   - `Source/Utils/util.cpp:44-84` 的 `findBinaryROIMask()` 通过非黑色像素（3 通道）或 alpha ≠ 0 像素（4 通道）识别有效区域。
   - 这些有效区域就是几何对齐后每幅图像在公共画布上的投影范围。
3. **重叠区计算依赖对齐结果**：
   - `intersectROIsPro()` 计算相邻图像 ROI 索引的交集。
   - 只有在重叠区内才会提取像素、建立颜色对应关系、构造 QP 能量函数。
4. **邻接关系需外部指定**：
   - `Data/Cache/ParamsPro.txt` 需要人工或前序拼接程序提供图像之间的相邻关系。

### 10.2 完整的图像拼接流水线

```
拍摄多张图像
    ↓
特征提取与匹配（如 SIFT / SURF / ORB）
    ↓
几何配准与变换估计（如 RANSAC + 单应性矩阵）
    ↓
图像 warping 到公共坐标系，生成带黑边或 alpha 通道的对齐图像
    ↓
【本项目】颜色一致性校正
    ↓
图像融合（如 多波段融合 / 羽化）
    ↓
最终全景图
```

### 10.3 输入图像的形式

- 所有图像必须已经变换到同一尺寸画布上。
- 未覆盖区域应为纯黑色（RGB = 0,0,0）或 alpha = 0。
- 相邻图像之间必须存在真实的像素重叠，否则 `ParamsPro.txt` 中指定的邻接关系无效。

## 11. 总结

这是一个完整的学术研究型 C++ 项目，实现了图像拼接后处理中的颜色一致性校正。算法思路清晰：用 B-Spline 参数化重映射曲线，将颜色一致性、梯度保持、对比度约束统一为凸二次规划问题，求解后对每张图像逐像素校正。代码结构相对简洁，但受限于较老的 OpenCV 版本和 Windows 特定 API，若要在现代环境中复用，需要一定的迁移和现代化改造。

该项目在整个图像拼接流程中扮演**颜色后处理模块**的角色，前置条件是已完成几何对齐的多张图像。
