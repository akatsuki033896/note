---
tags:
  - Parallel101
---
# 智能指针(C++11)

**自动管理动态内存、防止内存泄漏的指针**，运用了 RAII 的资源管理方式，能够防止手动使用 `new/delete` 导致的内存泄漏问题。

没有智能指针的时候只能手动 `new` 和 `delete`，如果忘记释放指针就会内存泄漏，空悬指针会被利用于篡改系统内存来盗取数据

## `unique_ptr`：封装指针为容器

`unique_ptr` 容器在C++11引入，析构的时候会自动 `delete`，避免用户犯错

使用 `make_unique<type>()` 创建一个 `unique_ptr`，类似 `new`，参数可以为构造函数需要的参数

```cpp
struct C {
  C();
  ~C();
};
int main() {
  std::unique_ptr<C> p = std::make_unique<C>(); // 类似 new
  return 0; // 自动释放 ptr
}
```

我们知道一个指针释放后必须要把它设成 `NULL` 防止空悬指针，即 `delete p; p = nullptr`，而 `unique_ptr` 把这个操作封装了，只需要 `p = nullptr` 或 `p.reset()` 即可，就没有空悬指针了

### 禁止拷贝智能指针

根据三五法则，智能指针的实现把拷贝构造函数设成 `=delete` 了，防止指针容器的重复释放，因此不可以使用拷贝构造即 `std::unique_ptr<C> c1 = std::make_unique<C>()`，要用就直接用 `make_unique`

如果需要拷贝，根据是否需要控制对象生命周期有两种解决方案

#### 不需要控制对象生命周期：使用 `get()` 获取原始指针

```cpp
void func(C *p) {
  p -> do_something(); // 执行成员函数
}
int main() {
	std::unique_ptr<C> c1 = std::make_unique<C>();
	func(p.get());
}
```

#### 需要控制对象生命周期：移动

```cpp
std::vector<std::unique_ptr<C>> objlist;
void func(std::unique_ptr<C> p) {
  objlist.push_back(std::move(p));
}
int main() {
	std::unique_ptr<C> p = std::make_unique<C>(); // 转移指针控制权
	func(std::move(p)); 
}
```

移动构造函数转移指针后，`p` 就会变成空指针，如果还想访问的话需要提前用 `get()` 获取原始指针

```cpp
int main() {
	std::unique_ptr<C> p = std::make_unique<C>(); // 转移指针控制权
  C* raw_p = p.get(); // 获取原始指针
	func(std::move(p)); 
  raw_p->do_something(); // 执行成员函数
}
```

但是要保证获取到的原始指针 `raw_p` 存在时间不超过 `p` 的生命周期即更早地销毁，否则出现空悬指针，下面的例子是危险的

```cpp
int main() {
	std::unique_ptr<C> p = std::make_unique<C>(); // 转移指针控制权
  C* raw_p = p.get(); // 获取原始指针
	func(std::move(p)); 
  raw_p->do_something(); // 执行成员函数
  objlist.clear(); // 清空了，释放资源，raw_p悬空
  raw_p->do_something();
}
```

## `shared_ptr`

`unique_ptr` 通过禁止拷贝解决重复释放，使用比较困难，而 `shared_ptr` 牺牲了效率，使用引用计数解决重复释放的问题，换来可以拷贝的自由度。

`shared_ptr` 维护的引用计数在初始化时为 `0`，拷贝一次 `+1`，析构一次 `-1`，当引用计数为 `0` 时自动销毁指向的对象，因此只要只要还存在一个指针指向该对象，就不会被解构。

这个引用计数用 `use_count()` 获取，`shared_ptr` 用 `make_shared()` 构造

```cpp
std::shared_ptr<C> ptr = std::make_shared<C>();
std::cout << ptr.use_count() << '\n'; // 1
```

### 引用计数的实现

引用计数本身是使用指针实现的，也就是将计数变量存储在堆上，所以共享指针的 `shared_ptr` 就存储一个指向堆内存的指针

### `shared_ptr` 的问题

不能完全代替 `unique_ptr` 因为：

1. `shared_ptr` 维护的引用计数器是原子操作，需要额外一块内存，访问实际对象需要二级指针，`deleter` 使用类型擦除
2. 全部使用 `shared_ptr` 会出现循环引用导致内存泄漏（两个指针相互引用对方，结果谁都因为引用计数不为0导致释放不掉）

之后讨论循环引用的问题

## `weak_ptr`：弱引用

因为弱引用的拷贝与解构不影响其引用计数器，因此有时候我们想维护一个 `shared_ptr` 的弱引用。

类似的，`C*` 是 `unique_ptr<C>` 的弱引用，但是 `weak_ptr` 提供失效检测，更安全。之后有需要时通过 `lock()` 产生一个新的 `shared_ptr` 作为强引用，但不 `lock` 的时候不影响计数。如果计数器归零则 `expired()` 返回 `false` 且 `lock()` 返回 `nullptr`

```cpp
std::shared_ptr<C> p = std::make_shared<C>();
std::weak_ptr<C> weak_p = p; // 创建弱引用
func(std::move(p)); // 移动，p为nullptr
std::cout << weak_p.exipired() << '\n'; // false 并未失效
weak_p.lock()->do_something(); // 正常执行
```

## 智能指针作为类的成员变量

> `shared_ptr` 对象生命周期取决于引用中最长寿的一个，`unique_ptr` 对象生命周期取决于引用的唯一对象

根据实际所有权判断用哪种指针

1. `unique_ptr`：对象仅属于自己
2. 原始指针：对象不属于自己，但他释放时自己必然被释放
3. `shared_ptr`：该对象由多个对象共享 / 该对象仅仅属于自己，但需要使用 `weak_ptr`
4. `weak_ptr`：该对象不属于自己，且释放后自己仍可能不被释放
5. `shared_ptr` 和 `weak_ptr` 一起更安全

### 循环引用的解决方案

把逻辑上不具有所有权的指针改成 `weak_ptr`，在这个例子中规定了子窗口属于父窗口

```cpp
struct C {
	std::shared_ptr<C> m_child;
	std::weak_ptr<C> m_parent;
};
int main() {
  auto parent = std::make_shared<C>();
  auto child = std::make_shared<C>();
  // 相互引用
  parent->m_child = child;
  child->m_parent = parent;
  
  parent = nullptr;
  child = nullptr;
}
```

或者改成原始指针，使用 `get()` 获取，因为这个例子中父窗口的释放必然导致子窗口的释放

```cpp
struct C {
  std::shared_ptr<C> m_child;
  C* m_parent;
};
int main() {
  ...
  child->m_parent = parent.get()
}
```

继续改进，把子窗口指针改成 `unique_ptr`，因为这个对象仅仅属于自己

```cpp
struct C {
  std::unique_ptr<C> m_child;
  C* m_parent;
};
int main() {
  ...
  parent->m_child = std::move(child);
  child->m_parent = parent.get();
}
```

