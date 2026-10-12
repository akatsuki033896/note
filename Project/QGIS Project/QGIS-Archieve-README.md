
# How to Use

- 18PC：复制到C盘文档
- ecdis-example-release-5.12.3：电子海图浏览器
- QGIS安装：安装包和配置
	- qgis配置.txt：设置->选项->系统->环境
	- xnmap.reg：注册表，配置xnmap文件夹使用
- qgis_python_库：
	- qgis3.16-apps-python37-lib：site-packages文件夹复制到QGIS安装目录 `./apps-Python37-lib`
	- Roaming-python-python37：site-packages文件夹复制到 `用户/AppData/Roaming/python/python37`
- QGisData：显示海图必要的文件，复制到C盘文档
- xnmap：海图文件 复制到C盘文档
- 电子海图浏览器_演示程序_Win6411：电子海图浏览器旧版本
- 项目：QGIS插件源码和数据处理项目源码
- QGIS_plugin_develop：复制到插件部署路径 `E:`
- 项目使用说明：插件的docx文档 / 使用说明文档，插件docx文档后面有典型操作流程
- 项目使用资料：插件使用的测试数据
- ENC海图：使用资料

# 项目

- 项目路径：`QGIS/QGIS项目/项目文件`
- `.` 为项目源码根目录

| 项目                        | 描述                                                                                     | 数据                                              |
| ------------------------- | -------------------------------------------------------------------------------------- | ----------------------------------------------- |
| DepthContourGenerator     | 从csv文件生成含有水深点的shapefile                                                                |                                                 |
| OgrS57ToGeoJs             | S57海图`.000` 转为shapefile和geojson                                                        | `QGIS/项目使用资料/s57toall`                          |
| S57全球海图数据汇总               | S57海图`.000` 数据提取成 `csv`                                                                | `QGIS/xnmap/S57Map/S63客户全球`                     |
| s52样式转换mapbox             | 根据 `.dai` 转换符合S-52标准的svg<br>**之后可能维护S-100版本（高优先级）（S-100海图有类似S-52的规定符号样式，进行了重大的升级和扩展）** | `./PresLib4.0.dai`                              |
| satellite_tile_crawler_ui | 卫星图爬虫                                                                                  |                                                 |
| shptos57                  | shapefile转为S57海图 `.000`                                                                | 数据来源：`./file`<br>输出目录：`./outfile/{s57name}.000` |
| VTSdataCreate             | 创建VTS数据                                                                                | `QGIS/项目使用资料/大数据分析VTS`                          |
| Xchart范围提取                | 根据geojson提取s57海图范围                                                                     | `.`                                             |
| 插卡式AIS                    | 将插卡式AIS整理成 `.docx`                                                                     | `QGIS/项目使用资料/插卡式AIS`                            |
| 海图结构数据化                   | 见源码名称                                                                                  | `QGIS/QGIS项目使用资料/海图数据结构化`                       |
| 解析generic                 | 统计 `.000` 海图测量控制点、坐标、物标                                                                | 任意`.000` 海图文件                                   |

# 插件

QGIS INFO

```sh
QGIS_PREFIX_PATH环境变量： C:/PROGRA~1/QGIS3~1.16/apps/qgis  
路径前缀： C:/PROGRA~1/QGIS3~1.16/apps/qgis  
插件路径： C:/PROGRA~1/QGIS3~1.16/apps/qgis/plugins  
程序包数据路径： C:/PROGRA~1/QGIS3~1.16/apps/qgis/.  
活动主题名称： default  
活动主题路径： C:/PROGRA~1/QGIS3~1.16/apps/qgis/./resources/themes\default\icons/  
默认主题路径： :/images/themes/default/  
SVG搜索路径： C:/PROGRA~1/QGIS3~1.16/apps/qgis/./svg/                C:/Users/VVVVV/AppData/Roaming/QGIS/QGIS3\profiles\default/svg/  
用户数据库路径： C:/PROGRA~1/QGIS3~1.16/apps/qgis/./resources/qgis.db  
认证数据库路径： C:/Users/VVVVV/AppData/Roaming/QGIS/QGIS3\profiles\default/qgis-auth.db
```

## xncompiler

发布：新诺海图
未加密：chart_unencrypted
加密：chart_encryption
产物：海图范围加密
mbtiles：S57（ECDIS使用）

## s57_unknown_feature_adder

objectclass文件路径：对象类，`QGIS/ecdis-example-release-5.12.3/Resources/S57ObjectClasses/zh-CN`

attributes：属性，`QGIS/ecdis-example-release-5.12.3/Resources/S57Attributes/zh-CN`

csv数据：.\test_file，`s57attributes.csv` 和 `s57objectclasses.csv` 也可以从 QGIS安装路径中 `\share\gdal` 找到

枚举：`.xls`，不给枚举就自己加

## S57toXinuoMap

障碍物转换-处理红叉数据，数据有问题

# 电子海图浏览器

ECDIS sdk 的example

切瓦片：`ecdis-example-release-5.12.3/example(2).exe`
