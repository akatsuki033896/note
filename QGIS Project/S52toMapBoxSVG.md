
读取 S-52 电子海图显示库以及 S-100 / S-101 符号资源，将其中的点符号、面图案、线样式逐一解析并转换为 DAY / DUSK / NIGHT 三种配色的svg，，并生成可用于 Mapbox / MapLibre / QGIS 渲染的样式资源（s101_style.json）

## Quick Start

### S52 PresLib to SVG

转换 `Preslib_e4.0.0.dai` 显示库文件为 `./output` 目录内 `.svg` 图片，分为 DAY / DUSK / NIGHT，`cjpl.dai` 分为 CJ_DAY / CJ_DUSK / CJ_NIGHT

```sh
cd S52toMapboxSVG
python main.py
```

## S101 submodule

### `convert_s100_svg`

将 S-100 标准 SVG（用 CSS 类名 sCHMGD/fCHMGD 引用颜色）转换为内联实际色值的标准 SVG

```sh
python S101/convert_s100_svg.py -i symbols -o output -p day --verbose
```

| **参数**         | **默认**       | **说明**                        |
| -------------- | ------------ | ----------------------------- |
| -i / --input   | S101/symbols | 输入 SVG 目录                     |
| -o / --output  | S101/output  | 输出目录（自动建 day/dusk/night 子目录）  |
| -c / --css-dir | S101/css     | CSS 配色文件目录                    |
| -p / --profile | S101/all     | 配色方案：day / dusk / night / all |
| -v / --verbose | -            | 打印每个文件的转换明细                   |

Output：

- 对应配色方案目录内的 S101 的 svg
- 预览页 preview_day/dusk/night.html、index.html
- S101 与 S57 差异对比页 diff.html

### `convert_svg_viewbox.py`

对 S-101 SVG 进行 viewBox 重算与 mm→px 单位转换

```py
# 更改配置路径
INPUT_DIR = r"E:\QGISdata\test\s52样式转换mapbox\S101\Output\test\night"
OUTPUT_DIR = r"E:\QGISdata\test\s52样式转换mapbox\S101\Output\test\ChangeOutput\night"
S57_DIR = r"E:\QGISdata\test\s52样式转换mapbox\S101\Output\dayS57"
DPI = 96          # mm→px 的 DPI
SCALE = 1.5      # 输出尺寸缩放倍数
```

## dataset

| name               | description             |
| ------------------ | ----------------------- |
| PresLib_e4.0.0.dai | IHO S-52 显示库（全球）        |
| cjpl.dai           | 长江内河符号显示库               |
| Pslb_4.0.0.dai     | 备用显示库                   |
| s101_style.json    | Mapbox/MapLibre 规范的样式文件 |

> [!IMPORTANT]
> s101_style.json 是符合 Mapbox/MapLibre 规范的样式文件，定义了图层
> (layers)、过滤器(filter)、字形(glyphs)等。转换得到的 SVG 图标作为 sprite
> 资源配合该样式使用，实现海图符号在 Mapbox 系渲染引擎中的可视化

## script

| name                     | description                                                           | usage                               |
| ------------------------ | --------------------------------------------------------------------- | ----------------------------------- |
| `Amplify_svg.py`         | 依据真实 path/transform 几何（含矩阵）重算外接框并重设 viewBox，修复 viewBox 与内容不符          | 改 path 后运行                          |
| `change_to_xchar_w_h.py` | 把 SVG 的 width/height 单位(mm/pt/px)统一转整型 px，并放大 1.35 倍                  | 改 path 后运行                          |
| `check_dai_inst.py`      | 统计 .dai 的 INST 中 SY()/LC()/AP() 引用，核对是否都有对应定义，列出缺失项                   | 改顶部 DAI_FILE 后运行                    |
| `rename.py`              | 按照规范重命名                                                               | 默认 svg 地址为 `./output` 文件夹           |
| `w_h_int.py`             | 把 SVG 的 width/height 单位(mm/pt/px)统一转整型 px（系数 1.0）                     | 改顶部目录后批量运行                          |
| `xn_将图片铺满.py`            | 用 S-52 尺寸改写 X-Chart 图例 SVG 的 width/height                             | 改 x_chart_dir_path/s52_dir_path 后运行 |
| `批量翻转SVG 文件.py`          | 批量翻转 SVG 方向（默认 `flip_y=True`），仅改 `<g>` 的 transform，不增删元素，只处理以 L 开头的文件 | 改 TARGET_DIRECTORY 后运行              |
| `提取objectclass所有attr.py` | S-57 物标/属性提取、AML 附加层与小类.dai 生成                                        | 按需放开注释并配置路径                         |
| `查看s57物标名.py`            | 比对 S57ObjectClasses 与 S57Attributes，找出缺失属性并从 CSV 补回 ATTR 文件           | 改 ATTR_FILE/OBJ_FILE/excel_file 后运行 |
| `移除rect.py`              | 用正则删除 SVG 中所有自闭合 `<rect .../>`（多用于清理 S-101 夜间输出）                      | 改 TARGET_FOLDER 后运行                 |
| `转换颜色表.py`               | 颜色表解析原型，可指定单个 ID（如 501518）调试输出                                        | 改顶部 path 与目标 ID 后运行                 |

## Appendix: 矢量指令速查

| **指令** | **含义** | **说明**                          |
| ------ | ------ | ------------------------------- |
| SP     | 设置颜色   | 后接令牌变量号，经 SCRF/PCRF/LCRF 映射到颜色表 |
| SW     | 设置线宽   | 值 × 0.3                         |
| ST     | 设置透明度  | 0/1/2/3 → 100%/75%/50%/25%      |
| PU     | 抬笔移动   | 设定下一段起点                         |
| PD     | 落笔绘制   | 给出坐标序列；空坐标表示画点(圆)               |
| PM     | 路径模式   | 0=开始子路径, 1=继续, 2=结束             |
| CI     | 绘制圆    | 以最后 PU 点为圆心                     |
| FP     | 填充多边形  | 对当前闭合路径填色                       |
| EP     | 描边多边形  | 对当前闭合路径描边                       |
| SC     | 引用子符号  | SC(符号名,方向)，方向仅 0/1/2            |
