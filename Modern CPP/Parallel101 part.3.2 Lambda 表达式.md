---
tags:
  - Parallel101
---
C++11最重要的特性之一，本质是语法糖，最多的应用是回调函数和Qt的 `connect`
## Reference

[bilibili.com/video/BV1ui4y1R78s/?spm_id_from=333.1387.search.video_card.click](https://www.bilibili.com/video/BV1ui4y1R78s/?spm_id_from=333.1387.search.video_card.click&vd_source=9d28f0e4734f1bde4c84c3169e3a429d)

## 省流


| 捕获类型        | 描述                                                                                                  |
| ----------- | --------------------------------------------------------------------------------------------------- |
| 空           | 没有使用任何函数对象参数                                                                                        |
| `=`         | 函数体内可以使用**Lambda所在作用范围内所有可见的局部变量（包括Lambda所在类的this）**，并且是值传递方式（相当于编译器自动为我们按值传递了所有局部变量                |
| `&`         | 函数体内可以使用**Lambda所在作用范围内所有可见的局部变量（包括Lambda所在类的this）**，并且是引用传递方式（相当于编译器自动为我们按引用传递了所有局部变量）             |
| `this`      | 函数体内可以使用Lambda所在类中的成员变量                                                                             |
| `变量名`       | 将变量名按**值**进行传递。按值进行传递时，**函数体内不能修改传递进来的变量的拷贝**，因为默认情况下函数是const的。**要修改传递进来的变量的拷贝**，可以添加 `mutable` 修饰符 |
| `&a`        | 将a按引用进行传递                                                                                           |
| `a, &b`     | 将a按值进行传递，b按引用进行传递                                                                                   |
| `=, &a, &b` | 除a和b按引用进行传递外，其他参数都按值进行传递                                                                            |
| `&, a, b`   | 除a和b按值进行传递外，其他参数都按引用进行传递                                                                            |

## 语法

lambda表达式可以在函数体内创建一个函数，不需要在全局添加函数，基本语法为：

```cpp
[捕获列表](参数列表) mutable(可选) 异常属性 -> 返回类型 {  
	// 函数体 
}
```

- 捕获列表：可以理解为参数的一种类型，**Lambda 表达式内部函数体在默认情况下是不能够使用函数体外部的变量的， 这时候捕获列表可以起到传递外部数据的作用**，捕获列表分为值捕获，引用捕获，隐式捕获，表达式捕获 
- 参数列表：类似函数的参数列表
- 返回类型：在参数列表后 `-> 类型` 添加返回类型，不指定时和 `-> auto` 等价，没有 `return` 时和 `-> void` 等价。
- `mutable` 声明，按值传递函数对象参数时，加上 `mutable` 修饰符后，可以修改按值传递进来的拷贝（**注意是能修改拷贝，而不是值本身**）。

```cpp
template<class Func>
void call_twice(Func func) {
  std::cout << func(0) << '\n';
  std::cout << func(1) << '\n';
}
int main() {
  auto twice = [](int n) -> int {
    return n * 2;
  };
  call_twice(twice);
  return 0;
}
```

## 值捕获

与参数传值类似，**值捕获的前提是变量可以拷贝**
不同之处则在于：被捕获的变量在 Lambda 表达式被创建时拷贝， 而非调用时才拷贝

```cpp
void lambda_value_capture() {  
    int value = 1;  
    auto copy_value = [value] {  
        return value;  
    };  
    value = 100;  
    auto stored_value = copy_value();  
    std::cout << "stored_value = " << stored_value << std::endl;  
    // 这时, stored_value == 1, 而 value == 100.  
    // 因为 copy_value 在创建时就保存了一份 value 的拷贝  
}
```
## 引用捕获

与引用传参类似，引用捕获保存的是引用，值会发生变化。

```cpp
void lambda_reference_capture() {  
    int value = 1;  
    auto copy_value = [&value] {  
        return value;  
    };  
    value = 100;  
    auto stored_value = copy_value();  
    std::cout << "stored_value = " << stored_value << std::endl;  
    // 这时, stored_value == 100, value == 100.  
    // 因为 copy_value 保存的是引用  
}
```

## 隐式捕获

手写捕获列表很复杂，可以直接在捕获列表中写一个 `[&]` 或 `[=]` 向编译器声明采用引用捕获或者值捕获
### `[&]` 捕获引用变量

lambda函数体中可以使用定义他的 `main` 函数中的变量，把 `[]` 改成 `[&]`，与引用传参类似，引用捕获保存的是引用。

```cpp
#include <iostream>

template <class Func> void call_twice(Func func) {
  std::cout << func(0) << '\n';
  std::cout << func(1) << '\n';
}
int main() {
  int fac = 2;
  auto twice = [&](int n) -> int { return n * fac; }; // fac 是 main 函数定义的变量
  call_twice(twice);
  return 0;
}
```

`[&]` 还可以写入 `main()` 中的变量

```cpp
int main() {
  int fac = 2;
  int cnt = 0;
  auto twice = [&](int n) -> int { 
    cnt++;
    return n * fac;
  };
  call_twice(twice);
}
```

#### 传入常引用避免拷贝开销

将模板参数声明为 `const&` 避免不必要的拷贝，传参时只传入内存地址，不用重新创建一个对象

```cpp
template<class Func>
void call_twice(Func const& func) {
   ...
}
```

#### 函数作为返回值

```cpp
auto make_twice(int fac) {
  return [](int n) {
    return n * 2;
  };
}
int main() {
  auto twice = make_twice();
  call_twice(twice);
  return 0;
}
```

然而使用 `return [&](int n) {return n * fac;};` 捕获定义域 `make_twice` 的 `fac` 时出错，因为 `[&]` 捕获的是引用即 `fac` 的地址，但是 `make_twice` 已经返回了，导致 `fac` 的引用变成了一块已经失效的地址，因此要保证lambda对象生命周期不超过他捕获的所有引用的寿命

###  `[=]` 值捕获变量

`[=]` 捕获的是值而不是引用，给每个引用的变量做一份拷贝，性能可能不如 `[&]`，上面的例子改成 `return [=](int n) {return n * fac;};` 即可

### `[]` 捕获变量

Lambda 表达式的本质是一个和函数对象类型相似的类类型（称为闭包类型）的对象（称为闭包对象）， 当 Lambda 表达式的捕获列表为空时，闭包对象还能够转换为函数指针值进行传递。

> 闭包(*closure*)
>
> 函数式编程的特性，指的是函数可以引用定义位置所有的变量


- `[]` 空捕获列表
- `[name1, name2, ...]` 捕获一系列变量
- `[&]` 引用捕获, 从函数体内的使用确定引用捕获列表
- `[=]` 值捕获, 从函数体内的使用确定值捕获列表

### 信号槽

```c++
/* 做信号和槽连接，默认内部变量是加锁的，也就是只读状态，如果在内部进行修改，那么程序会崩溃 */
connect(btn3, &QPushButton::clicked, this, [&](){
	btn3->setText("mutable 叒改名惹！");
});
```



