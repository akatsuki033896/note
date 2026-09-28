---
tags:
  - Parallel101
---
# RAII 和资源管理

## 资源的概念

资源是任何在使用前需要获取与释放（显式 / 隐式）的东西，分为内存资源（`std::string`, `std::vector`) 和非内存资源，非内存资源有锁(`std::mutex`) / socket / 文件句柄 / 线程句柄(`std::thread`) 等，长时间运行的系统要避免资源泄漏，也要避免过度占用资源。

### 资源管理

资源管理的目标是要处理所有资源类型。

很多语言把资源管理的任务交给垃圾回收机制，垃圾回收机制是全局内存管理模式，但是现在越来越多系统都是分布式的（多核 / 缓存 / 集群），所以在局部范围内管理资源越来越重要了。可以使用垃圾回收机制，但是最好的方法是不要乱扔垃圾，就不需要垃圾回收机制。

## RAII 的概念

RAII - Resource Acquisition Is Initialization 指的是资源获取即初始化，资源释放即销毁，对应了类的构造函数(constructor)和析构函数(destructor)，通过定义构造函数 / 拷贝 / 移动 / 析构函数，用户就能对受控资源的生命周期完全控制。

RAII 将在使用前获取的资源的生命周期，与某个对象的生命周期绑定在一起，在这个对象生命周期结束的时候，按照与获取资源的顺序相反的顺序释放资源，消除了资源泄漏和异常安全。

RAII 的核心思想就是**利用栈上局部变量的自动析构来保证资源一定会被释放**

### RAII 的作用

1. 减少内存泄漏：RAII 避免了 c语言中的 `malloc()` 和 `free()`， c++ 的裸 `new` 和 `delete` 操作，使对象在离开 `{}` 括起来的作用域时自动销毁，把内存管理的过程交给语言实现
2. 异常安全：保证当异常发生时，会调用已创建对象的解构函数。

### RAII 常用操作

```cpp
class X {
public:
  X(Sometype); // 普通的构造函数，申请资源
  X(); // 默认构造函数
  X(const X&); // 拷贝构造函数
  X(X&&); // 移动构造函数
  X& operator=(const X&); // 重载拷贝赋值操作符，清空目标对象并拷贝
  X& operator=(X&&); // 重载移动赋值操作符，清空目标对象并移动
  ~X(); // 析构函数回收资源
};
```

### 实现 RAII 类的步骤

1. 用一个类封装资源
2. 用构造函数**执行资源的初始化，比如申请内存、打开文件、申请锁**
3. 用**析构函数中执行销毁操作，比如释放内存、关闭文件、释放锁**
4.  使用时声明一个该对象的类，一般在你希望的作用域声明即可，比如在函数开始，或者作为类的成员变量

对于文件资源

```cpp
#include <iostream>
#include <fstream>

int main() {
    std::ifstream myfile("example.txt"); // 换自己的文件路径
    if (myfile.is_open()) {
        std::cout << "File is opened." << std::endl;
        // do some work with the file
    }
    else {
        std::cout << "Failed to open the file." << std::endl;
    }
    myfile.close();
    return 0;
}
```

使用 RAII 类封装后

```cpp
class File {
	std::ifstream filename;
public:
	File(const char* _filename) : filename(_filename) {}
	~File() {
		if (filename.is_open()) {
			filename.close();
		}
	}
	std::ifstream& getFilename() {
		return filename;
	}
}
```

# 构造函数

## 构造函数的调用：`{}` 和 `()`

使用 `{}` 意味着非强制类型转换，因此 `int{3.14f}` 会出错，`int(3.14f)` 不会出错，但是 `{}` 看起来没那么像函数调用，可读性比较好

## 自定义构造函数 / 初始化表达式

不同的自定义构造函数和生成对象的语法

```cpp
struct Circle {
	std::string name;
	int size;
	Circle() {
		name = "c1";
		size = 50;
	} // 无参构造
  
  Circle() : name("c1"), size(50) {} // 初始化表达式
  // Circle circle;
  
  Circle(std::string m_name, int m_size) : name(m_name), size(m_size) {} // 多参数构造的初始化表达式
  // Circle circle("c1", 50);
  
  explicit Circle(int m_size) : name("A " + std::to_string(size) + " circle"), size(m_size) {} // 单参数构造的初始化表达式
  // Circle circle(50);
};
```

> 为什么使用初始表达式?
>
> 1. 假如类成员为 const 和引用
>
> 2. 假如类成员没有无参构造函数
>
> 3. 避免重复初始化，更高效

### `explicit` 关键字在构造函数的作用

#面试常考 

详解：[[explicit关键字]]

#### 单参数构造函数

加入一个打印信息的函数，尝试单个函数构造并在主函数调用

```cpp
struct Circle {
  ...
  explicit Circle(int m_size) : name("A " + std::to_string(size) + " circle"), size(m_size) {} // 单参数构造的初始化表达式
  // Circle circle(50);
}
void show(Circle c) {
	std::cout << c.name << " " << c.size << std::endl;
}
int main() {
  show(80); // 编译出错
  show(Circle(80)); // 编译通过
}
```

单个参数构造一个对象时，可能写成 `show(80)`，但是编译器帮你隐式转换导致编译通过了。为了避免隐式转换，在单个参数构造函数前声明 `explicit`，例如 `std::vector` 的构造函数 `vector(size_tn)` 也声明了 `explicit`

#### 多参数构造函数

```cpp
struct Circle {
  ...
	explicit Circle(std::string m_name, int m_size) : name(m_name), size(m_size) {} // 多参数构造的初始化表达式
}
Circle func() {
  return Circle{"cirlce", 80}; // 通过
  return Circle("circle", 80); // 通过
  return {"circle", 80}; // 不通过
}
```

多个参数构造一个对象时，`explicit` 可以禁止从一个 `{}` 初始化，在这个例子中禁止通过返回的 `{"circle", 80}` 初始化一个 `Cirlcle` 的对象

## 编译器生成的构造函数

```cpp
struct Circle {
	std::string name;
	int size;
}
```

当一个类没有定义任何构造函数，且所有成员都有无参构造函数时，编译器会自动生成一个无参构造函数，他会调用每个成员的无参构造函数来进行初始化。但是出于兼容老标准的考虑，有些成员类型(POD, *plain-old-data*)不可以被初始化为 `0`，而是内存的随机值：

1. `int`,`float` 等基础类型
2. `void*`, `Object*` 等指针类型
3. 完全由基础类型和指针类型组成的类

### 使用 `{}` 指定成员的值

通过 `{}` 手动指定初始化成员的值解决 POD 陷阱，这样在编译器生成构造函数中就能直接执行，但是在自定义的构造函数中也会执行

```cpp
struct Circle {
  std::string name;
  int size{0};
}
```

#### 零初始化的其他语法

使用 `{}` 不用写 `0`也可以表示零初始化例如 `int x{}`，`void *p{}` ，没有 `{}` 则会变成内存随机值

### 初始化列表

一个类没有定义任何构造函数时，编译器会生成一个参数个数和成员一样的构造函数，把 `{}` 和 `={}` 里的内容按顺序赋值给成员

还可以只初始化一部分参数，其他为默认值（前提是有定义，否则是内存中的随机值）

#### 简化函数多返回值的处理

```cpp
struct HitRes {
  bool hit;
  Vec3 pos;
  Vec normal;
  float depth;
}
HitRes intersect(Ray r) {
  ...
  return {ture, r.origin, r.direction, 233.0f};
}
int main() {
  Ray r;
  auto hit = intersect(r); // 通过intsercet初始化列表构造
}
```

 比 `std::tuple` 好用，每个属性都有名字不容易搞错

#### 处理函数的复杂类型参数

```cpp
void func(std::tuple<int, float, std::string> args, std::vector<int> arr) {
  ...
}
int main() {
  func({1, 1.0, "a"}, {1, 2, 3});
  func(std::tuple<int, float, std::string>(1, 1.0, "a"), std::vector<int>({1, 2, 3}));
  func(std::tuple(1, 1.0, "a"), std::vector({1, 2, 3})); // c++17
}
```

相比 `std::tuple` 不用重新写一遍类型名

### `=default` 有构造函数时使用默认构造函数

一旦用户定义里构造函数，编译器就不再生成默认的无参构造函数，还想使用默认构造函数就要添加 `=default`

```cpp
struct Circle {
	std::string name;
	int size;
  Circle() = default;
	Circle(std::string m_name, int m_size) : name(m_name), size(m_size) {} // 多参数构造的初始化表达式
};
```

# 拷贝构造和拷贝赋值

### 拷贝构造函数

编译器会默认生成的特殊构造函数 `X(const X&)`，在一个类的对象尚未初始化的时候把另一个对象拷贝进来初始化，`X x2 = x1;` 和 `X x2(x1)` 都能调用

用户也可以自定义拷贝构造函数

```cpp
struct Circle {
	std::string name;
	int size;
  Circle(Circle& const m_circle) : name(m_circle.name), size(m_circle.size) {} // 自定义拷贝函数
};
```

### 拷贝赋值函数

编译器默认重载运算符 `=`，语法为 `X& operator=(const X&);`， 在一个类的对象已经初始化的时候将当前对象销毁，再把另一个对象拷贝进来，出于性能的考虑最好使用拷贝构造以避免一次无参构造。同样可以使用 `=default` 和 `=delete`

### `=delete` 不让编译器自动生成拷贝函数

```cpp
struct Circle {
	std::string name;
	int size;
  Circle() = default;
  Circle(Circle& const) = delete; // 禁止拷贝构造
  Circle &operator=(Circle& const) = delete; // 禁止拷贝赋值
};
```

# 三五法则

## 拷贝构造函数

> 如果一个类定义了解构函数，那么您必须同时定义或删除拷贝构造函数和拷贝赋值函数，否则出错

在使用 `=` 的时候默认会拷贝

```cpp
struct Vector {
  size_t m_size;
  int *m_data;
	Vector(size_t n) {
    m_size = n;
    m_data = (int*)malloc(n * sizeof(int));
  }
  ~Vector() {
    free(m_data);
  }
}
int main() {
  Vector v1(32);
  Vector v2 = v1; // 发生拷贝
  return 0;
}
```

对于类型为指针的类成员，拷贝的时候指针是按地址值**浅拷贝**，而不是**深拷贝**其指向的数据，意味着退出 `main()` 作用域的时候 `v1.m_data` 会被释放两次，`v1` 被析构但 `v2` 还在使用的时候会内存泄漏

### 允许用户拷贝

同时定义拷贝构造函数和拷贝赋值函数来把数据深拷贝过来

```cpp
Vector(Vector const& other) {
  m_size = other.m_size;
  m_data = (int*)malloc(m_size * sizeof(int));
  memcpy(m_data, other.m_data, m_size * sizeof(int));
}
```

### 禁止用户拷贝

直接禁止用户拷贝，并使用 `=delete` 让编译器不要生成默认的会导致指针浅拷贝的拷贝构造函数，就能在编译期提前发现错误

```cpp
struct Vector {
  size_t m_size;
  int *m_data;
	Vector(size_t n) {
    m_size = n;
    m_data = (int*)malloc(n * sizeof(int));
  }
  Vector(Vector const&) = delete; // 不要让编译器生成
  ~Vector() {
    free(m_data);
  }
}
```

> RAII 保证任何单个操作前后对象都处于正确的状态，避免程序读取到空悬指针，避免内存泄漏，为面向对象的“封装：不变性”服务

## 拷贝赋值函数

> 如果一个类定义了拷贝构造函数，那么您必须同时定义或删除拷贝赋值函数，否则出错，删除可导致低效



> 拷贝赋值函数 = 析构函数 + 拷贝构造函数

区分两种拷贝可以提高性能，内存的销毁重新分配可以通过 `realloc` 从而就地利用当前现有的 `m_data` 避免重新分配，因此建议自定义拷贝赋值

```cpp
Vector &operator=(Vector const& other) {
  m_size = other.m_size;
  m_data = (int*)realloc(m_data, m_size * sizeof(int));
  memcpy(m_data, other.m_data, m_size * sizeof(int));
  return *this;
}
```

## 移动构造函数

移动构造函数允许对象从一个作用域直接移到另一个作用域，更高效，也适用于不能拷贝的情况

### `std::move` 实现移动 / 移动语义

> C++11 新增了移动函数，适合需要把一个对象移动到另一个对象上，但不涉及实际数据的的拷贝的情况

```cpp
void func() {
	std::vector<int> v1(10);
	std::vector<int> v2(100);
	
  v1 = std::move(v2); // 移动赋值 O(1)
	v1 = v2; // 拷贝赋值 O(n)
}
```

使用 `std::move` 把 `v2` 移到 `v1`，原来的 `v2` 会被清空即元素个数变成 `0`

### `std::swap` 实现交换

`std::swap(v1, v2)` 交换两者的值，可以在高性能计算中实现双缓存(*ping-pong buffer*)，使两个缓存单元交替读写

### 移动构造函数的缺省实现

不重视降低复杂度(O(n) >>> O(1))的情况下

> 移动构造 = 拷贝构造 + 他解构 + 他默认构造
>
> 移动赋值 = 拷贝赋值 + 他解构 + 他默认构造

用户不自定义移动构造和移动赋值时，编译器会自动构造

自定义移动构造，不重视提高性能(O(1) >>> O(0.1))时

> 移动赋值 = 析构 + 移动构造

### 删除拷贝赋值函数

如果已经实现移动赋值，可以删除拷贝赋值函数，这样调用 `v2 = v1` 时，因为拷贝赋值被删除了，编译器尝试 `v2 = List(v1)`，从而先调用拷贝构造函数，因为 `List(v1)` 相当于就地构造对象，成为了移动语义，进一步调用移动赋值函数

## 符合三五法则的类型



