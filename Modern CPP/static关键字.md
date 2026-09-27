---
tags:
---

# 修饰全局变量

`static` 修饰全局变量使该变量的作用域限制在所在的文件中，其他文件无法访问

`static` 修饰的全局变量在程序启动时被初始化（执行 `main` 函数之前），生命周期和程序一样长。

```cpp
// a.cpp 文件
static int a = 10;  // static 修饰全局变量
int main() {
    a++;  // 合法，可以在当前文件中访问 a
    return 0;
}

// b.cpp 文件
extern int a;  // 声明 a
void foo() {
    a++;  // 非法，会报链接错误，其他文件无法访问 a
}
```

# 修饰局部变量

`static` 修饰局部变量可以使得**变量在函数调用后仍留在内存里**，不会被销毁，下次调用该函数的时候继续使用。

`count` 用 `static` 修饰，`foo` 退出的时候 `count=1`，下次调用的时候 `static int count = 0` 不再执行，直接 `count++`
`count` 仍然是局部变量，作用域仅限于函数内部，只有 `foo` 可以访问

```cpp
void foo() {
    static int count = 0;  // static 修饰局部变量
    count++;
    cout << count << endl;
}

int main() {
    foo();  // 输出 1
    foo();  // 输出 2
    foo();  // 输出 3
    return 0;
}
```

# 修饰函数

类似修饰全局变量，`static` 修饰函数可以将函数的作用域限定在当前文件中，使得其他文件无法访问该函数。同时，由于 `static` 修饰的函数只能在当前文件中被调用，因此可以避免命名冲突和代码重复定义。

```cpp
// a.cpp 文件
static void foo() {  // static 修饰函数
    cout << "Hello, world!" << endl;
}

int main() {
    foo();  // 合法，可以在当前文件中调用 foo 函数
    return 0;
}

// b.cpp 文件
extern void foo(); // 声明 foo
void bar() {
    foo();  // 非法，会报链接错误，找不到 foo 函数，其他文件无法调用 foo 函数
}
```

# 修饰类成员变量和函数

`static` 修饰类成员变量和函数可以使得它们在所有类对象中共享，且不需要创建对象就可以直接访问

```cpp
class MyClass {
public:
    static int count;  // static 修饰类成员变量
    static void foo() {  // static 修饰类成员函数
        cout << count << endl;
    }
};
// 访问：

MyClass::count;
MyClass::foo();
```

