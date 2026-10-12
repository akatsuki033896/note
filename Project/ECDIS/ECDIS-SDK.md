
GsBBox
GsS52Raz
GsWholin
GsLeglin
GsPOS
GsCoord
GsGeometry
GsCollision

>[!question]
>S52符号是怎么显示的？

## GS_BOOL

`typedef int GS_BOOL` 模拟 C++ 的 bool

## GsBBox

包围盒类，以坐标表示包围盒，类除了接口还有计算交集的函数

```c
ClassMembers(GsBBox, Base)
	double minx;	// the minimum x-coordinate
	double maxx;	// the maximum x-coordinate
	double miny;	// the minimum y-coordinate
	double maxy;	// the maximum y-coordinate
EndOfClassMembers;
```

## GsS52Raz

S52符号类

### Examples

用S-52 符号化引擎画出一个虚拟物标

1. 构造一个要素(Feature)：点类型，OBJL="cursor"，属性 cursty=1（实心样式 A）
2. 把要素构造成S-52并调用 `GsS52RazResolveCS()` 解析
3. 设置成 Qt 的光标

```cpp
// examples/GsCanvasWidget.cpp
// 960-989
{
	GsBaseFeature geoData;
	GsBaseFeatureConstructorParams params;
	
	// 构造要素
	params.core = mCore;
	params.objType = POINT_T;

	// solid cursor, style A
	geoData = (GsBaseFeature)ooc_new(GsBaseFeature, (const void *)&params);

	GsCoordSequenceConstructorParams params1;
	params1.allocated = 1;
	params1.allocation_chunks = 0;
	params1.dims = 2;

	GsCoordSequence coordSeq = ooc_new(GsCoordSequence, &params1);
	GsCoordSequencePushBack(
		coordSeq,
		std::numeric_limits<double>::quiet_NaN(),
		std::numeric_limits<double>::quiet_NaN()
	);

	GsBaseFeatureSetGeometry(geoData, (GsGeometry)coordSeq);
	GsBaseFeatureSetOBJL(geoData, "cursor");
	GsBaseFeatureSetAttVal(geoData, "cursty", "1");
	
	// 构造 S52
	cursra01 = ooc_new(GsS52Raz, (const void *)geoData);
	GsS52RazResolveCS(cursra01);
}
	// 设置光标
	setCursor(GsGraphicsQtCreateCursor(mGraphicsQt, cursra01));
```

### GsS52RazResolveCS(GsS52Raz self)

符号化流程的总入口

```cpp
// GsS52Raz.cpp
void GsS52RazResolveCS(GsS52Raz self)
{
	GsBaseFeature geoData = GsS52RazBaseFeature(self); // 取出绑定要素

	if (geoData->defn == NULL)
		return ;
	
	// 缓存显示上限比例尺
	const char *scaminstr = GsBaseFeatureAttVal(geoData, "SCAMIN");
	geoData->scamin = (NULL == scaminstr) ? GS_INFINITY : atof(scaminstr);
	
	_linkLUP(self); // 查找Look-Up Table
	_resolveSMB(self); // 解析符号 + 展开条件符号
	
	GsS52RazAdjustDPRI(self); // 设置显示优先级
}
```

#### `GsS52RazGetS52PL()`

获取显示库，标准物标用 "IHO" 库，`OBJL≥10000` 的扩展物标用 "CJHDJ" 库

#### `GS_BOOL _linkLUP(GsS52Raz self)`

 取出S57 物标对象的物标名 `OBJL`，从查找表中查找符合的S52符号，获取 DPRI / RPRI/ DISC / LUCM 并赋值给自己。展示库从  `GsS52RazGetS52PL()` 获取。

| 代码       | 英文全称                                                           | 中文含义与核心作用                                                                                                                                                                                      |
| -------- | -------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| DPRI     | Display Priority                                               | 一个数值（例如从 0 到 9 或 12 等），用来决定当海图上不同要素（如陆地、等深线、航标、水深点）在同一位置重叠时，**谁浮在最上面显示**（数字越大越优先显示），以确保最重要的导航安全信息不会被次要信息遮挡                                                                                     |
| **RPRI** | **Radar Priority**  <br>(或 Radar Overlay Priority)             | **雷达叠显优先级**：决定海图要素与雷达图像（Radar/ARPA）重叠时，**谁显示在最上层**。例如，高雷达优先级的海图线（如安全等深线）会压在雷达回波上方，确保关键航行安全信息不被雷达杂波遮挡。                                                                                          |
| **DISC** | **Display Category**                                           | **显示类别**：决定该海图要素属于哪一个显示层级。S-52 规定了三大基础类别：  <br>1. **Base（基础显示）**：任何时候都不能关闭（如海岸线、安全等深线）。  <br>2. **Standard（标准显示）**：开机默认显示，满足基本航行需求。  <br>3. **Other / All（其他/全部显示）**：次要信息（如某些水深点），可根据船员需求手动开关。 |
| **LUCM** | **Lookup Table Common / Modification**  <br>(或 Lookup Comment) | **查找表通用/注释字段**：在 S-52 的“物标向符号转换”机制中，它属于**查找表（Lookup Table）**的一个控制参数或匹配说明。它用于关联特定的 S-57 物标属性（如属性组合），并指导系统调用对应的条件符号化程序（Conditional Symbology Procedure）。                                         |

Preslib.dai

```
"BOYSPP","","SY(BOYSPP01)","7","O","DISPLAYBASE","25010"
```

- `"BOYSPP"`：**S-57 物标名称**（Special Purpose Buoy，专用浮标）。
- `""`：**S-57 属性组合**（这里为空，表示所有该类浮标默认都用这个规则）。
- `"SY(BOYSPP01)"`：**S-52 符号指令**。告诉系统去 S-52 符号库里调用名为 `BOYSPP01` 的图形符号（Symbol）。
- `"7"`：**DPRI（显示优先级）**。该符号的显示层级为 7。
- `"O"`：RPRI **雷达优先级**。这里的 O 代表 Over-radar（压在雷达图层之上）。
- `"DISPLAYBASE"`：**DISC（显示类别）**。属于基础显示层（Display Base），任何时候不能被船员关闭。
- `"25010"`：**S-52 内部底图组编号**（View Group）。

#### `GS_BOOL _resolveSMB(GsS52Raz self)`

把 LUP 里的 `S52_CmdWL` 类型符号指令串解析成可执行的绘图命令序列，赋值给 `cmdFinal` 成员

#### `void GsS52RazAdjustDPRI(GsS52Raz self)`

提取当前 DPRI 和几何类型，DPRI 层级以链表表示

先从旧的链表 `disPrioLayers[DPRI][obj_t]` 中删除，如果是常用航点把新的显示层级加到链表尾部，如果是 waypnt 航点物标且属性为 trnrad 就在头部添加


## GsCoordinate / GsCoord

包含 `double` 类型的横坐标和纵坐标

含一个判断坐标相等的成员函数 `GsCoordEqual(const GsCoord self, const GsCoord other)`

