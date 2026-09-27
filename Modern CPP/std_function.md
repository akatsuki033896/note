现代 C++ 最重要的特性之一，Lambda 提供类似匿名函数的特性。匿名函数的使用场景是需要一个函数，但是又不想费力去命名一个函数的情况。

函数可以作为另一个函数的参数，且这个作为参数的函数也可以有参数例如 `void func_2(void func_1(int))`，函数类型还可以作为模板参数 `template <class Func>`，但是要在全局添加函数
# `std::function`：避免用模板参数 (C++11)

`std::function` 是一种通用、多态的函数封装， 它的实例可以对任何可以调用的目标实体进行存储、复制和调用操作， 它也是对 C++ 中现有的可调用实体的一种类型安全的包裹（相对来说，函数指针的调用不是类型安全的）， 换句话说，就是函数的容器。当我们有了函数的容器之后便能够更加方便的将函数、函数指针作为对象进行处理。

采用了**类型擦除技术**，无需写明仿函数类的具体类型，能容纳任何仿函数或函数指针。只需在模板参数中写明函数的参数和返回值类型即可，所有具有同样参数和返回值类型的仿函数或函数指针都可以传入。

```cpp
#include <functional>
#include <iostream>

int foo(int para) {
    return para;
}

int main() {
    // std::function 包装了一个返回值为 int, 参数为 int 的函数
    std::function<int(int)> func = foo;
    
    int important = 10;
    std::function<int(int)> func2 = [&](int value) -> int {
        return 1+value+important;
    };
    std::cout << func(10) << std::endl;
    std::cout << func2(10) << std::endl;
}
```
## 使用案例

`<class Func>` 可以让编译器对每个不同的lambda生成一次，有助于优化，但有时候我们希望通过头文件分离声明和实现，这时不能用 `template class` 作为参数，为了灵活性可以用 `std::function`，尖括号内写 `<返回类型(参数列表)>` 

```cpp
#include <functional>
#include <iostream>

void call_twice(std::function<int(int)> const &func) {
    std::cout << func(0) << '\n';
    std::cout << func(1) << '\n';
}

std::function<int(int)> make_twice(int fac) {
    return [=] (int n) {
        return n * fac;
    };
}

int main() {
    auto twice = make_twice(2);
    call_twice(twice);
    return 0;
}
```

## 无捕获的lambda传为函数指针

lambda不捕获局部变量，为 `[]`，那么只需要用函数指针类型 `int(int)` 即可，最大的好处是用于一些只接收函数指针的C语言API比如 `pthread`

## 类型擦除

只要提供一个 `operator()` 就可以实现调用各种类型的函数，`std::array` 也是一个类型擦除的容器

# Media Link

`std::function` 的原理：
https://www.bilibili.com/video/BV1yH4y1d74e/?spm_id_from=333.337.search-card.all.click&vd_source=9d28f0e4734f1bde4c84c3169e3a429d

