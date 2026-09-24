
(2026.09.17)巨硬我真是日你全家，我在 mac 上用的 cmake 和 clang 从下载包到配置到运行 C++ API 例程全程不超过10分钟，我一天就在这边给你解决链接和兼容性问题得了...

官方因为 ABI 兼容问题建议生产环境用 C API，具体来说就是在 macos 上使用 g++ 编译会找不到符号出错，但是使用 clang 没问题。

GEOS 例程：[https://github.com/libgeos/geos/tree/main/examples](https://github.com/libgeos/geos/tree/main/examples)

---

## Introduction

GEOS 是一个 C/C++ 的地理空间计算机几何库，本身是 Java 的地理空间库 [JTS](https://github.com/locationtech/jts/) 的 C++ 实现，也是 QGIS, GDAL, Shapely 的依赖。

### Interface

支持 C 接口和 C++ 接口，C 接口提供ABI稳定性，C++ 不稳定但是可使用现代 C
++ 特性例如 `std::unique_ptr<Geometry*>`。接口使用可重入接口，线程安全。

### Capabilities

Spatial Model and Functions

- **Geometry Model**: Point, LineString, Polygon, MultiPoint, MultiLineString, MultiPolygon, GeometryCollection
- **Predicates**: Intersects, Touches, Disjoint, Crosses, Within, Contains, Overlaps, Equals, Covers（断言：交集、接触、不相交、交叉、内部、包含、重叠、相等、覆盖）
- **Operations**: Union, Distance, Intersection, Symmetric Difference, Convex Hull, Envelope, Buffer, Simplify, Polygon Assembly, Valid, Area, Length（操作：并集、距离、交集、对称差、凸包、缓冲区、简化、多边形组合、验证、面积、长度）
- **Prepared geometry** (using internal spatial indexes)（处理几何（预空间索引））
- **Spatial Indexes**: STR (Sort-Tile-Recursive) packed R-tree spatial index（R树空间索引）
- **Input/Output**: OGC Well Known Text (WKT) and Well Known Binary (WKB) readers and writers.（WKT 和 WKB 编码器及解码器）

## `geos::geom`

GEOS 的核心几何类所在的命名空间，GEOS 是不推荐 `#include <geos.h>` 或 `#include <geos/geom.h>` 的，最好是包含一个具体的头文件，然后在源码中定义命名空间

```cpp
#include <geos/geom/Geometry.h>
using namespace geos::geom;
```

https://libgeos.org/doxygen/cpp_iface.html

| 类                                   | 描述            |
| ----------------------------------- | ------------- |
| `geos::geom::Geometry`              | 所有几何类型的基类     |
| `geos::geom::GeometryFactory`       | 创建、销毁几何对象的工厂类 |
| `geos::geom::CoordinateSequence`    | 创建基础几何对象      |
| 几何对象向量（vectors of geometries）       | 创建几何集合        |
| `geos::geom::GeometricShapeFactory` | 构建特定几何形状对象    |
| `geos::geom::geosversion()`         | 获取 GEOS 版本字符串 |

### `Geometry`：几何对象类

`Geometry` 是 GEOS 的基本操作对象，可以为点 / 线 / 面等，这些分别对应了 `Point`，`LineString`, `Polygon`, `Coordinate` 等类，他们是 `Geometry` 的子类。

对于多个点、线、面，他们是 `GeometryCollection` 的子类。

![](https://libgeos.org/doxygen/classgeos_1_1geom_1_1Geometry.png)

#### 坐标

坐标是GEOS 的最小组成单位，一个坐标用 `Coordinate` 表示，多个坐标用 `CoordinateSequence` / `CoordinateArraySequence` 表示，想要构造点线面就要使用这些坐标。

`Coordinate` 作为基类，衍生出 `CoordinateXY`、`CoordinateXYZ`、`CoordinateXYM` 和 `CoordinateXYZM`

#### `CoordinateArraySequence`：坐标序列

https://libgeos.org/doxygen/classgeos_1_1geom_1_1CoordinateSequence.html#details

成员包括一个坐标数组和一个维度，`Coordinate` 类本质上只是包含 3 个 `double` 类型的变量来表示点，甚至是不满足 GEOS 上的最小可操作单位的。

```cpp
std::vector<Coordinate> vect;
mutable std::size_t dimension;
```

除了 `get/set` 函数，还能执行类似 STL 的操作例如增删改查，除了增加的函数是 `add()` 名字也基本上一样，不多介绍。

部分几何操作如下，此外还支持转为 `vector`，还有用于复制的`clone()` 函数，返回智能指针。

```cpp
bool 	isRing () const;
bool 	hasRepeatedPoints () const;
void 	toVector (std::vector< Coordinate > &coords) const; 
```

### `GeometryFactory` 

https://libgeos.org/doxygen/classgeos_1_1geom_1_1GeometryFactory.html

用于创建各种 `Geometry`  几何对象的工厂类，管理创建的几何对象的内存和空间参考系统。

#### `getDefaultInstance()`： 获取全局单例

```cpp
#include <geos/geom/GeometryFactory.h>

const GeometryFactory* gf = geos::geom::GeometryFactory::getDefaultInstance();
auto point = gf->createPoint(coord);
```

`getDefaultInstance()` 调用默认构造函数获取全局的工厂单例，整个程序都默认使用这个单例，不可以自己 `delete`。适合只需要使用默认工厂、不需要自己管理工厂生命周期的情况

![[Pasted image 20260922101208.png]]

创建一个工厂，然后使用它来创建各种要素。
#### `GeometryFactory::create()`：创建一个工厂

```cpp
int main()
{
    /* New factory with default (float) precision model */
    GeometryFactory::Ptr factory = GeometryFactory::create();
    Coordinate pt(3, 2);
    auto p = factory->createPoint(pt);
    std::cout << p->getGeometryType() << std::endl;
}
```

`Ptr` 类型是 `unique_ptr`，`create()` 会返回该类型的智能指针

```cpp
using Ptr = std::unique_ptr<GeometryFactory, GeometryFactoryDeleter>;
static GeometryFactory::Ptr create();
```

就是创建一个工厂，并把所有权交给一个智能指针来管理。

```cpp
{ 
	GeometryFactory::Ptr gf = GeometryFactory::create(); // 使用 factory 
}
// gf 被销毁
```

实际上两种创建方法都不需要自己手动管理生命周期，但是第一种写法返回裸指针 `const GeometryFactory*`，周期交给 GEOS 本身管理，第二种返回智能指针 `unique_ptr<GeometryFactory, ...>`，周期给 C++ 语义本身管理，第二种比较现代化

## `geos::io`：GEOS 的输入输出

`geos::io` 是负责 GEOS 读写的命名空间，主要可以进行 WKT / WKB 的读写

### WKT(Well-Known Text)

https://en.wikipedia.org/wiki/Well-known_text_representation_of_geometry

WKT 是一种开放标准纯文本标记语言，可以表示二维和三维的空间几何对象，比如点 / 线 / 多边形，还可以定义空间参考系统，便于在 GIS、地理空间数据库进行数据交换。以下是 2D 的 WKT 的文本形式范例，Well-Known Binary 解释参照 wikipedia

![](https://picgocloud.com/m/66e7a1a4-7b21-4dae-a524-3032b3b0c96f.png)

![](https://picgocloud.com/m/ed79dc98-c107-49d7-b9ea-e38bda78227e.png)

查看 WKT 的在线工具： https://wktmap.com/

样例代码描述了 GEOS 从 `string` 表示的 WKT 中读取地理空间对象，转为 `Geometry` 类型的智能指针存储，之后就能

```cpp
GeometryFactory::Ptr factory = GeometryFactory::create();
/*
* Reader requires a factory to bind the geometry to
* for shared resources like the PrecisionModel
*/
WKTReader reader(*factory);

/* Input WKT strings */
std::string wkt_a("POLYGON((0 0, 10 0, 10 10, 0 10, 0 0))");
std::string wkt_b("POLYGON((5 5, 15 5, 15 15, 5 15, 5 5))");

/* Convert WKT to Geometry */
std::unique_ptr<Geometry> geom_a(reader.read(wkt_a));
std::unique_ptr<Geometry> geom_b(reader.read(wkt_b));
```

- `WKTReader` 是用于解析 WKT 的类，构造时传入 `GeometryFactory` 对象，即操作 WKT 也要通过工厂类
- `WKTReader` 的成员函数 `read` 和另一个重载都传入一个 `std::string` 类型的 WKT
- `readCoordinateSequence` 从 WKT 中获取坐标序列

```cpp
std::unique_ptr<T> read (const std::string &wkt) const // Parse a WKT string returning a Geometry.
std::unique_ptr<geom::Geometry> read (const std::string &wellKnownText) const
std::unique_ptr<geom::CoordinateSequence> readCoordinates (const std::string &wellKnownText) const
```

类似的，GEOS 还可以操作 WKB： https://libgeos.org/doxygen/classgeos_1_1io_1_1WKBReader.html

## Reference

- https://www.cnblogs.com/denny402/p/4967049.html
- https://zhuanlan.zhihu.com/p/400676925

