
NaviLib 文件夹和 Storage 文件夹无权限，需要事先拷贝

## 编译

vs 里以解决方案打开会强制连着 sdk 一起编译。

### 编译 ECDIS

```sh
cd E:\ECDIS
msbuild ECDIS.vcxproj /p:Configuration=Release /p:Platform=x64 /m /v:m
```

### 编译 SDK

用 vs，因为编译完sdk有设置生成后事件拷贝到别的目录

## Run

in developer powershell for VS 2022:

```sh
E:\ECDIS\NaviLib\Windows\x64\Release\ECDIS.exe
```

## 登录验证

只位于 `xnChartManageWidget.cpp` 的安装海图前的登录验证，拉取之后注释掉

```cpp
if (!xnUserMan::GetInstance()->AbtainPermission(this, USER_PERM_CHART_IO))
	return;
```

| 行号                           | 所在函数                                             | 说明          |
| ---------------------------- | ------------------------------------------------ | ----------- |
| xnChartManageWidget.cpp:1952 | InstallExchangeSet()                             | 安装交换集       |
| xnChartManageWidget.cpp:2005 | InstallCJExchangeSet()                           | 安装长江(CJ)交换集 |
| xnChartManageWidget.cpp:2386 | InstallChartCell(const std::string& productID)   | 安装海图单元      |
| xnChartManageWidget.cpp:2439 | InstallS100Dataset(const std::string& productID) | 安装 S100 数据集 |

弹登录窗的语句在 `xnUserMan::AbtainPermission` 内部，未登录时会弹出 `xnLoginDlg`

```cpp
xnLoginDlg loginDlg(parent);
if (QDialog::Accepted == loginDlg.Exec())
{
	return AbtainPermission(parent, permission);
}
else {
	return false;
}
```

## 瓦片资源请求

通过样式 JSON 文件记录的瓦片服务器 ip 向服务器请求资源，以下 ip 以拷贝的源文件为准，实际部署要改成自己虚拟机的 ip，虚拟机后台开着就行

挂载目录：

-  `mbgl-dev\ecdis-sdk-2\resources` 
- `ECDIS\Storage\ReadWrite\MapData`

| 端口   | 用途                  | 示例                                                                                                |
| ---- | ------------------- | ------------------------------------------------------------------------------------------------- |
| 8081 | 字体，精灵图              | glyphs（第 2 行）和 sprite（第 1075 行）                                                                   |
| 8082 | 矢量瓦片（.pbf），即主要的瓦片服务 | http://192.168.1.68:8082/tiles/senc_L13-15/{z}/{x}/{y}.pbf、natural_earth、Track、Route、NaviWarn 等图层 |
| 8083 | 栅格瓦片（.png）          | http://192.168.1.68:8083/tiles/raster-arcgisimage/{z}/{x}/{y}.png                                 |

### 192.168.1.68

路径：`E:\ECDIS\Storage\ReadOnly\Render\styles\style-L.json`

| 端口   | 处数  | 内容                                           |
| ---- | --- | -------------------------------------------- |
| 8081 | 2   | glyphs、sprite                                |
| 8082 | 15  | 各矢量瓦片图层（senc_L*、natural_earth、Track、Route 等） |
| 8083 | 1   | raster-arcgisimage 栅格瓦片                      |

### 10.8.7.227

路径：`Storage\ReadOnly\Render\styles\`

| 组              | 文件                                                                    | 该 IP 条目                                                                       |
| -------------- | --------------------------------------------------------------------- | ----------------------------------------------------------------------------- |
| enc 海图样式（12 个） | `style-enc_plainbnd_*` 和 `style-enc_symbolbnd_*` 的 day/dusk/night 全组合 | 每个文件 5 条：senc_L3-8、senc_L9-13、hdf5_S-102 三个瓦片源（8082）+ sprite + glyphs（8081）   |
| xn 样式（4 个）     | style-xn_common / style-xn_day / style-xn_dusk / style-xn_night       | 每个文件 5 条：senc_xn_L5-12_13 ×2、senc_xn_patch_L5-12（8082）+ sprite + glyphs（8081） |
| 用户数据（1 个）      | style-userdata.json                                                   | 6 条全在 8082：Waypoint、Route、Track、PlotPoint、PlotLine、PlotSurface                |
