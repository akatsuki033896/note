
`xnRouteDesignElement` 为航线对象，继承自 `xnElement`

- **数据**：`QVector<xnRouteNode*> m_routeNodes`： 航点经纬度、安全距离、转向半径、航程类型（RL/GC）、ETA/ETD/Stay/Speed 等
- **显示**：`m_waypnts / m_legLins / m_wholins`（S52 海图要素类型为 `QList<GsS52Raz>`），把航线挂到海图 cell 上渲染
- **运行**：监控状态、航线检测结果（GsCollision）、历史平均航速统计、RTZ 格式航线（`m_rootRtzNode`）

## 接口

大部分内容查看 `Elements/xnRouteDesignElement.h` 注释，获取经纬度时返回 `xnLatLon` 类型

## 函数和类的成员

绘制安全距离线：

```cpp
void RenderYawLine(QPainter* painter, xnViewPort* viewPort);

void RenderYawLine(QPainter* painter, xnViewPort* viewPort,
    const QList<GsS52Raz>& waypnts, const QList<GsS52Raz>& leglins, const QVector<xnRouteNode*>& routeNodes);
    
void CalculateCentralAngle(xnLatlon F1, xnLatlon O, xnLatlon F2, double& startAngle, double& spanAngle);

QPoint CalculateCoursePos(QPainter* painter, QPoint homepos, const QString& course, double legdeg);

QPoint CalculateSpeedPos(QPainter* painter, QPoint homepos, const QString& speed, double legdeg);
```

计算包围盒：跨180°经度手工处理

```cpp
void CalculateBBOX();
```

创建航点：

前三个都用 `#if 0` 暂时屏蔽了

```cpp
GsS52Raz NewWaypoint(xnRouteNode* node, int id);
GsS52Raz NewLegLin(GsS52Raz waypnt1, GsS52Raz waypnt2, xnRouteNode* node);
GsS52Raz NewWholin(GsS52Raz leglin, xnRouteNode* node);
GsS52Raz GetWaypoint(int index);
GsS52Raz GetLeglin(int index);
GsS52Raz GetWholin(int index);
```

航线检测：

`GsCollision` 类型返回碰撞状态

```cpp
//航线检测
bool Detection();
void DelDetection();
bool IsDetected();
unsigned int GetDetectionResult() const { return m_collisionFlag; }
GsCollision GetGsCollision() const { return m_collision; }
```

引入geos，这边在干什么没看懂

```cpp
//用于航线途径图幅
CoordinateArraySequence* CoordSequenceFromGeodesicLine(const std::pair<Coordinate, Coordinate>& coordinatePair);
CoordinateArraySequence* CoordSequenceFromRhumbLine(const std::pair<Coordinate, Coordinate>& coordinatePair);
std::vector<Geometry*>* CreateGeodesicBuffer(const CoordinateArraySequence* coordSequence, double distance);

GsBBox m_outlineBox;
std::vector<Geometry*>* m_buffers;  //数据由m_outlineBuffer托管，本地不做析构
Geometry* m_outlineBuffer;			//航线安全区域轮廓
bool m_bUpdateOutlineBuffer;        //是否需要更新buffer
const GeometryFactory* m_geomFactory;
```

`CoordinateArraySequence`，`Coordinate`，`Geometry`，`GeometryFactory` 都是 GEOS 的类。

---

构造函数，可以从空构造，也可也传入另一个航线来构造。
其中 `m_geomFactory` 在类构造时：

```cpp
m_geomFactory = geos::geom::GeometryFactory::getDefaultInstance();
```

析构时：

```cpp
if (m_outlineBox)
{
    ooc_delete((Object)m_outlineBox); // ooc.h
    m_outlineBox = nullptr;
}
if (m_outlineBuffer)
{
    m_geomFactory->destroyGeometry(m_outlineBuffer);
    m_outlineBuffer = nullptr;
}
```

`Init()` 函数提取航线的设置中初始化，构造函数最后也执行这个函数

`std::vector<Geometry*>* xnRouteDesignElement::CreateGeodesicBuffer`：使用Geographiclib
