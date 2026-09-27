# 基于 FFT 的高质量扫描图像去网纹工具

请帮我创建一个用于高质量扫描图像去网纹的图像处理项目。

## 项目背景

我有大量日本 CD 歌词册/内页的高分辨率扫描图片，扫描分辨率通常为 **600 DPI**。原始扫描图片存在明显的印刷网点、周期性半色调（Halftone）以及由扫描产生的 Moiré / 网纹。目前我在 Photoshop / ImageJ 中进行人工处理，大致流程是：

```text
原始扫描
    ↓
FFT / Fourier Spectrum
    ↓
观察频谱中的周期性亮点
    ↓
手动将对应的亮点涂黑
    ↓
Inverse FFT
    ↓
得到去网纹图像
```

实际测试发现，**直接在 FFT 频谱中针对周期性亮点进行 notch filtering 的效果明显优于普通 Gaussian Blur、Noise Reduction、Dust & Scratches 等空间域降噪方法**。

普通降噪会明显损失：日文小字体，细线，插画细节，纸张纹理，印刷边缘，因此，本项目的核心目标是：**尽可能保留原始扫描细节，只针对具有明显周期性的印刷网纹进行频域抑制。**

---

# 内容

实现一个：**Python + NumPy / OpenCV 的 FFT 自动去网纹命令行工具**

要求能够：

1. 读取 TIFF 图片
2. 进行 FFT
3. 生成 FFT Spectrum
4. 指定若干频谱峰值位置
5. 自动生成 Notch Filter Mask
6. 对指定频率进行抑制
7. Inverse FFT
8. 保存处理结果
9. 支持单张图片处理

---

# 技术栈

使用：Python 3 / NumPy / OpenCV / SciPy（如果 FFT 性能/内存管理需要）/ tifffile（用于高质量 TIFF 读写）

```text
numpy
opencv-python
scipy
tifffile
```

如果某个库实际上没有必要，请不要为了增加依赖而使用它。

---

# 4. 图像质量要求

1. 不要转换成jpeg，中间结果使用tiff
2. 尽量保留原始位深，不要无意义地降低位深。
3. 避免重复复制巨大数组，600 DPI 扫描图可能非常大。FFT 会产生非常大的中间数组，因此需要注意：

- float32 / float64 的选择
- complex 数组内存
- 临时数组数量
- 不必要的数据复制
- batch processing 时及时释放内存

请优先考虑 OpenCV `cv2.dft()` 或合适的 SciPy FFT 实现，而不是无脑创建大量 NumPy 临时数组。

---




# 8. 频率点设计

不要直接依赖绝对像素坐标。优先设计为：

```python
peaks = [
    (dx, dy),
    ...
]
```

即相对于 FFT 中心点的坐标。

# 11. peaks.json

请设计一个配置文件，例如：

```json
{
    "peaks": [
        [120, 35],
        [-120, -35],
        [80, 90],
        [-80, -90]
    ],
    "radius": 4
}
```

程序读取这个配置文件后自动生成 notch mask。

更推荐配置成：

```json
{
    "peaks": [
        {
            "dx": 120,
            "dy": 35,
            "radius": 4,
            "strength": 1.0
        }
    ]
}
```

其中：

```text
dx / dy     频率坐标
radius      notch 半径
strength    抑制强度
```

这样以后可以从：

```text
Hard Notch
```

扩展到：

```text
Soft Gaussian Notch
```

---

# 12. Hard Notch 和 Gaussian Notch

请同时设计接口，使以后可以支持：

### Hard Notch

```text
frequency = 0
```

直接删除。

### Soft Notch

根据距离逐渐衰减：

```text
中心       → 强抑制
边缘       → 弱抑制
远处       → 不影响
```

例如：

```text
mask = 1 - strength * exp(-distance² / (2*sigma²))
```

第一阶段默认使用 Hard Notch，因为我要先验证它与目前 ImageJ 手工处理的结果是否一致。

---

# 13. 彩色图像

不要一开始就设计得过度复杂。

第一阶段先支持：

```text
Grayscale
```

并提供：

```text
RGB → Luminance
```

的处理方案。

第二阶段再研究：

```text
RGB 三通道 FFT
```

以及：

```text
YCbCr / Lab
```

中的频域处理。

特别注意：

> 不要因为分别处理 RGB 三通道而产生明显色偏或彩色伪影。

---

# 14. 批处理

最终希望能够这样使用：

```bash
python batch_descreen.py \
    --input ./input \
    --output ./output \
    --config ./config/peaks.json
```

例如：

```text
input/
├── page001.tif
├── page002.tif
├── page003.tif
└── page004.tif
```

处理后：

```text
output/
├── page001_descreen.tif
├── page002_descreen.tif
├── page003_descreen.tif
└── page004_descreen.tif
```

要求：

- 不覆盖原始文件
    
- 保留文件名
    
- 出错时明确报告具体文件
    
- 尽量避免单张图片失败导致整个 batch 中断
    
- 处理完成后输出统计信息
    

---


# 18. 性能要求

600 DPI TIFF 可能非常大。

请在代码中关注：

- 内存占用
    
- FFT 时间
    
- 临时数组
    
- batch processing
    
- 多线程
    

不要为了代码“看起来简单”而复制整个大图很多次。

如果发现 NumPy FFT 对超大图像内存压力明显，可以评估：

```text
OpenCV DFT
SciPy FFT
float32
分块处理
```

但不要在第一版中为了优化而过度复杂化。

---

# 20. 开发原则

请遵循以下原则：

1. 先实现最小可用版本。
2. 不要开发 GUI。
3. 不要使用深度学习。
4. 不要使用普通 Gaussian Blur 作为主要去网纹方法。
5. 不要使用不可逆的过度降噪。
6. 尽可能保留原始图像信息。
7. 所有频域处理尽量可解释。
8. 重要参数放入配置文件。
9. 算法和 CLI 分离。
10. README 中说明 FFT、Notch Filter 和频率坐标的基本原理。
11. 对超大 600 DPI TIFF 注意内存问题。
12. 不要修改原始文件。
13. 第一阶段优先验证“固定 12 个频谱峰值 + 自动 Notch”是否能够复现人工 ImageJ 操作的效果。
