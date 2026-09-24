# 经纬度编辑

- [[#`xnLatlon`]]
- [[#`xnLatLonEdit`]]
- [[#`xnLatOrLonEdit`]]

## `xnLatlon` 

存储经纬度数据并格式化

数据模型：

```cpp
enum FORMAT
{
	DEGREE,  // dd.dddddd 格式;
	DEG_MIN,  // dd癿m.mmmm
	DEG_MIN_SEC   // dd癿m'ss.ss
};

double lat, lon;
```

除了Get / Set接口、重载运算符、字符串处理、计算距离还包含以下API

| API                                           | 描述           |
| --------------------------------------------- | ------------ |
| `GsPOS ToGsPOS()` / `FromGsPos(const GsPOS&)` | 与 `GsPOS` 互转 |
| `GetDistanceToLine`                           | 点到直线距离       |
| `GetDistanceToLineSegment`                    | 点到线段最近距离     |
| `isometric_lat_inv()`                         | 等量纬度反解       |

## `xnLatLonEdit` 

UI经纬度显示和编辑

## `xnLatOrLonEdit`

编辑经度或纬度

# Route

## 新建航线

### 操作栏点击新建航线

1. `xnVoyagePlannerWidget` 发起 → `xnMouseAction` 捕获海图鼠标事件
2. `xnRouteDesignElement::AppendPoint` 把 `xnRouteNode` 逐个堆进航线
3. `xnRouteMan` 统一管理并经 `xnDataBase::RouteElementToRoute` 转成 NaviLib 的 `Route` 由 NaviDataManage 存入 Navi.db（编辑中间态和 RTZ 扩展树则存 xn.db 的 routeTmp/rtzNode 表）
4. `xnRouteImportExport::ExportToRtz` 用 `QDomDocument` 把内存模型重新序列化成 RTZ 1.0/1.2 XML 写文件。

# xnfunlib

`xnFunLib.h` 为定义帮手宏的头文件

| 宏            | 描述                                                                                   |
| ------------ | ------------------------------------------------------------------------------------ |
| `mmin(a, b)` | 若 `a < b` 则 `a` 为真                                                                   |
| `mmax(a, b)` | 若 `a > b` 则 `a` 为真                                                                   |
| `xnU(str)`   | `QString::fromLocal8Bit(str)`，将字符串转为Qt内部统一使用的UTF-16，**代码任何在UI显示中文字符的都要使用这个宏，否则显示异常** |
| `xnT(str)`   | `QString::tr(str)` 调用翻译                                                              |

