---
tags:
  - STL
---
# `std::variant`：存储不同类型的值

有时候需要一个类型要么存储 `int` 要么存储 `float`，可以用 `std::variant<int, float>`，`variant` 符合 RAII，是安全的 `union`，只存储一种类型（比较：`std::tuple` 是每个类型都存储）

```cpp
#include <variant>
int main() {
  std::variant<int, float> v = 3; // = 赋值即可
  v = 3.14f;
}
```

## `std::get` 获取容器数据

使用 `std::get<int>` 获取 `int` 类型数据的值，如果 `variant` 里不是这个类型，抛出异常 `std::bad_variant_access`，还能用 `std::get<0>` 获取 `variant` 列表第0个类型

## `std::holds_alternative`(C++17) 判断当前类型

判断当前类型是否在 `variant` 里面存在，不能查不在 `variant` 的类型。

```cpp
void print(std::variant<int, float> const &v) {
  if (std::holds_alternative<int>(v)) {
    std::cout << std::get<int>(v) << std::endl;
  }
  else if (std::holds_alternative<float>(v)) {
    std::cout << std::get<float>(v) << std::endl;
  }
}
```

## `.index()` 判断当前第几个类型

```cpp
if (v.index() == 0) {
  std::cout << std::get<0>(v) << std::endl;
  }
}
```

## `std::visit` 批量匹配

如果每个`if-else` 分支长得差不多，可以考虑用 `std::visit`，自动用相应类型调用你的lambda，lmbda往往是个重载函数

```cpp
void print((std::variant<int, float> const &v) {
  std::visit([&] (auto const &t) {
    std::cout << t << '\n';
  }, v);
}
```

带 `auto` 的lambda有多次编译的特性，实现编译多个分支的效果

### 静态多态和动态多态

`std::visit` `std::variant` 是静态多态，虚函数 / 抽象类是动态多态

静态多态优点：性能开销小，存储大小固定

缺点：类型固定不能运行时扩充

### 对多个参数支持

`visit` 支持多个 `variant` 作为参数，相应的lambda的参数数量要与之匹配，`visit` 会自动罗列出所有的排列组合，因此如果 `variant` 有 $n$ 个类型，lambda就要被编译 $n^2$ 次，但标准库可以保证运行的时候是 $O(1)$ 的

```cpp
auto add(std::variant<int, float> const &v1, std::variant<int, float> const &v2) {
  std::variant<int, float> ret;
  std::visit([&] (auto const &t1, auto const &t2) {
    ret = t1 + t2;
  }, v1, v2);
  return ret;
}
int main() {
  std::variant<int, float> v = 3;
  print(add(v, 3.14f));
  return 0;
}
```

### 可以有返回值

`visit` 里的lambda可以有返回值但要同样类型，进一步优化

```cpp
auto add(std::variant<int, float> const &v1, std::variant<int, float> const &v2) {
  return std::visit([&] (auto const &t1, auto const &t2) -> std::variant<int, float> {
   	return t1 + t2;
  }, v1, v2);
}
```

