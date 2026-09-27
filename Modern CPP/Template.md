为什么可以定义整数模板参数？

整数参数在模板参数里编译的时候就会生成，没有运行时的开销

> 为什么需要模板函数？
>
> 使用模板函数避免编写对函数的重载，实际情况例如按照传统面向对象思想，多个具体数据类型继承自一个 `Numeric` 类，有一个虚函数实现乘法运算，需要重载很多次函数，不同类型相乘则重载次数就是排列组合，处理不过来

# 模板函数

## 定义

含参数类型的调用：`twice<int>(21)`，`twice<float>(3.14f)`，etc.

自动推导参数类型的调用：`twice(21)`，`twice(3.14f)`

```cpp
template<class T>
T twice(T t) {
    return t * 2;
}
```

## 模板函数特化的重载

对于一些不符合统一实现的特殊情况，可以自己添加一个特化，特化会和已有的 `twice<T>(T)` 相互重载，调用方式为 `twice(std::string("str"))`

```cpp
template<class T>
T twice(T t) {
    return t * 2;
}
std::string twice(std::string t) {
  return t + t; // 字符串不能乘
}
```

## 默认参数类型

模板类型参数 `T` 没出现在函数参数中的时候，编译器无法自动推导参数类型，必须手动指定

```cpp
template<class T = int>
T two() {
  return 2;
}
```

手动指定调用：`two<float>()`

不指定时默认参数类型为 `int`：`two()`，等价于 `two<int>()`

# 模板参数

## 整数作为参数

模板参数只支持整数类型 `int` 和 `enum`，自定义类型也不可以

```cpp
template<int N>
void show(std::string msg) {
  for (int i = 0; i < N; i++)
    std::cout << msg << '\n';
}
int main() {
  show<1>("one");
  show<4>("four");
}
```

> 为什么支持整数作为模板参数？
>
> `template<int N>` 传入的 `N` 是编译期常量，编译器对每个不同的 `N` 单独生成一份代码对其优化，`func(int N)` 是运行期常量，编译器不能自动优化。例如 `show<0>()` 会被编译器自动优化成空函数，减少开销，提高性能，因此模板元编程对高性能编程很重要，同时过度使用模板会导致生成的二进制文件大小剧增，编译变慢

**模板的内部实现需要被暴露出来，定义和实现都必须放在头文件里**

## 多个模板参数

```cpp
template<int N = 1, class T>
void show(T msg) {
    for (int i = 0; i < N; i++) {
        std::cout << msg << '\n';
    }
}

int main() {
    show("one");
    show<3>(42);
}
```

可以同时指定整数和 `T`，注意默认值要在前

## 参数部分特化

使用参数部分特化实现任意类型容器内元素的总和

```cpp
template <class T>
T sum(std::vector<T> const & arr) {
    T res = 0;
    for (int i = 0; i < arr.size(); i++) {
        res += arr[i];
    }
    return res;
}
int main() {
    std::vector<int> a = {1, 2, 3, 4};
    std::cout << sum(a) << '\n'; // 10
}
```

# 模板的应用

## 编译期优化

声明一个函数求所有数的和，用一个参数 `debug` 控制是否输出调试信息，但 `debug` 是运行时判断，这样即使为 `false` 也会浪费 CPU 时间

```cpp
int sumto(int n, bool debug) {
  int res = 0;
  for (int i = 1; i <= n; i++) {
    res += i;
    if (debug) {
      std::cout << i << "-th": << res << '\n';
    }
  }
  return res;
}
```

将 `debug`改成模板参数也就是编译期常量，编译器生成 `sumto<true>` 和 `sumto<false>`，后者自动优化掉了输出语句

```cpp
template<bool debug>
int sumto(int n) {
  int res = 0;
  for (int i = 1; i <= n; i++) {
    res += i;
    if (debug) {
      std::cout << i << "-th": << res << '\n';
    }
  }
  return res;
}
```

## 编译期分支：`if constexpr`(C++17)

```cpp
template<bool debug>
int sumto(int n) {
  int res = 0;
  for (int i = 1; i <= n; i++) {
    res += i;
    if constexpr (debug) {
      std::cout << i << "-th": << res << '\n';
    }
  }
  return res;
}
```

`if constexpr` 表达式不能用运行时变量，因此可以保证是编译期确定的分支

## 编译期常量的限制

1. 不能通过运行时变量组成的表达式指定

2. 模板尖括号内的参数也不能用运行时变量，可以在变量定义前加上 `constexpr` 解决，但右值必须为编译期常量

```cpp
int main() {
  constexpr bool debug = true;
  std::cout << sumto<debug>(4) << '\n';
}
```

## 编译期常函数

编译期 `constexpr` 的表达式一般无法调用其他函数，但如果能保证被调用的函数都可以在编译期求值，在前面标 `constexpr` 即可解决

```cpp
constexpr bool isnegative(int n) {
  return n < 0;
}
int main() {
  constexpr bool debug = isnegative(-1);
  cout << sumto<debug>(4) << '\n';
}
```

`constexpr` 函数不能调用 `non-constexpr` 函数，`constexpr` 函数必须内联，不能分离声明和定义在另一个文件里。标准库函数例如 `std::min()` 也是 `constexpr` 函数。

## 模板的惰性

模板函数的声明和实现不能分离在两个文件，否则会出现 `undefined reference`，因为编译器对模板的编译是**惰性**的（只有当前 `.cpp` 文件用到了这个模板，该模板的函数才会被定义）

```cpp
sumto.cpp
#include "sumto.h" // 声明模板函数
#include <iostream>
// 定义模板函数 分离了
template<bool debug>
int sumto(int n) {
  int res = 0;
  for (int i = 1; i <= n; i++) {
    res += i;
    if constexpr (debug) {
      std::cout << i << "-th": << res << '\n';
    }
  }
  return res;
}
```

`sumto.cpp` 中没用到 `sumto<>` 函数的任何一个定义，所以 `main.cpp` 只看到函数的声明，可以在看得见 `sumto<>` 定义的 `sumto.cpp` 里增加两个显式编译模板的声明

```cpp
template int sumto<true>(int n);
template int sumto<false>(int n);
```

### 延迟编译

可以使用假模板实现延迟编译，加快编译的速度

```cpp
template<class T = void>
  void func() {
  	"1" = 111; // 字符串不可能被写入
}
int main() {
  return 0;
}
```

该例子编译可以通过，因为模板没被调用，不会被实际编译，不会报错

## 模板函数应用

输出一个容器

```cpp
#include <iostream>
#include <vector>

template <class T>
void print(std::vector<T> const& a) {
    std::cout << "{";
    for (size_t i = 0; i < a.size(); i++) {
        std::cout << a[i] << ' ';
    }
    std::cout << "}\n";
}
int main() {
    std::vector<int> a = {1, 2, 3, 4};
    print(a);
}
```

可配合运算符重载

```cpp
#include <iostream>
#include <ostream>
#include <vector>

template <class T>
std::ostream &operator<<(std::ostream &os, std::vector<T> const& a) {
    os << "{";
    for (size_t i = 0; i < a.size(); i++) {
        std::cout << a[i] << ' ';
    }
    os << "}\n";
    return os;
}
int main() {
    std::vector<int> a = {1, 2, 3, 4};
    std::cout << a;
}
```



