---
tags:
---

>[!quote]
>*一个成员函数是否以 const 修饰，不在于这个成员函数到底是否会修改自己的成员，而在于 “不变性”*

Reference：
- https://csguide.cn/cpp/basics/const.html
- Effective C++ 条款03

# 概述：在代码中表达成员不会被改变

不修改当前类的数据成员就要加 `const`，明确语义。但也不仅于此。
`const` 的对象不能调用没有用 `const` 修饰的函数，与该函数的实现是否修改数据无关。

```cpp
class Date {
	using Month = int;
public:
	Month m;
	Month month() const {
		return m;
	} // const 和非 const 对象都可以调用
};
```

因此，STL 的源码的函数通常也提供 `const` 和非 `const` 版本，因为函数返回的对象也通过 `const` 被区分成可以修改的 / 不可修改的。例如 `std::array` 重载的运算符 `[]`

`const` 修饰的函数成员想要修改私有数据可以使用 `mutable`

---

# 修饰变量

`const` 修饰变量时，该变量将被视为只读变量，即不能被修改。对于确定不会被修改的变量，应该加上 `const`，这样可以保证变量的值不会被无意中修改，也可以使编译器优化更加智能（当然只是对编译器的建议）

```cpp
const int a = 10;
a = 20; // 编译错误，a 是只读变量，不能被修改
```

实际上可以通过指针在运行时去间接修改这个变量的值即 `const_cast`，但很不安全

# 修饰函数参数

当 `const` 修饰函数参数时，表示**函数内部不会修改该参数的值**。这样做可以使代码更加安全，避免在函数内部无意中修改传入的参数值。尤其是引用作为参数时，如果确定不会修改引用，那么一定要使用 `const` 引用

```cpp
void func(const int a) {
    // 编译错误，不能修改 a 的值
    a = 10;
}
```

# `const T func()` 修饰返回值

当 `const` 修饰函数返回值时，表示**函数的返回值为只读，不能被修改**。这样做可以使函数返回的值更加安全，避免被误修改。

```cpp
const int func() {
    int a = 10;
    return a;
}

int main() {
    const int b = func(); // b 的值为 10，不能被修改
    b = 20; // 编译错误，b 是只读变量，不能被修改
    return 0;
}
```

# 修饰指针 / 引用

1. 声明指针本身为只读变量
2. 指向只读变量的指针

## `const T*` 声明指针本身为只读变量

**指针指向的变量是常量**

`const` 修饰的是指针所指向的变量，而不是指针本身。因此，指针本身可以被修改比如让他指向别的变量，但是不能通过指针修改所指向的变量的值。

```cpp
const int* p;  // 声明一个指向只读变量的指针
int a = 10;
const int b = 20;
p = &a;  // 合法，指针可以指向普通变量
p = &b;  // 合法，指针可以指向只读变量
*p = 30;  // 非法，无法通过指针修改只读变量的值
```

还有一种写法是 `int const * p`，意义完全一样，总之 `const` 出现在 `*` 左边就是被指的 `p` 是常量，否则是变量

## `T* const` 只读指针

**指针本身是常量**

`const` 修饰指针本身，使得指针本身只读，因此，指针本身不能被修改（即指针一旦初始化就不能指向其它变量），但是可以通过指针修改所指向的变量。

```cpp
int a = 10;
int b = 20;
int* const p = &a;  // 声明一个只读指针，指向 a
*p = 30;  // 合法，可以通过指针修改 a 的值
p = &b;  // 非法，无法修改指针本身
```

`T* const` 最常见的应用是 STL 的迭代器，意思是不可以让迭代器指向别的东西，但是迭代器指向的东西的值可以更改

```cpp
std::vector<int> vec;
const std::vector<int>::iterator iter = vec.begin();
*iter = 10; // 合法
++iter; // 非法
```

如果希望迭代器所指向的东西不能改变（也就是希望迭代器模拟出 `const T*`，要用 `const_iterator`

```cpp
std::vector<int> vec;
const std::vector<int>::const_iterator citer = vec.begin();
*citer = 10; // 非法
++citer; // 合法
```

## `const T* const` 只读指针指向只读变量

const 关键字同时修饰了指针本身和指针所指向的变量，使得指针本身和所指向的变量都成为只读变量。

```cpp
const int a = 10;
const int* const p = &a;  // 声明一个只读指针，指向只读变量 a
*p = 20;  // 非法，无法通过指针修改只读变量的值
p = nullptr;  // 非法，无法修改只读指针的值
```

## 常量引用

常量引用是指引用一个只读变量的引用，因此不能通过常量引用修改变量的值。

```cpp
const int a = 10;
const int& b = a;  // 声明一个常量引用，引用常量 a
b = 20;  // 非法，无法通过常量引用修改常量 a 的值
```

# `T func() const` 修饰成员函数

**仅适用类成员函数**
`const` 修饰成员函数时，表示该函数不会修改对象的状态（就是不会修改成员变量）

这样 `const` 的对象就可以调用这些成员方法了，因为 `const` 对象不允许调用非 `const` 的成员方法。既然对象是 `const` 的，要保证对象成员变量不能被修改，那么成员函数肯定要是 `const` 的才能保证不会被修改。

```cpp
class A {
public:
    int func() const {
        // 编译错误，不能修改成员变量的值
        m_value = 10;
        return m_value;
    }
private:
    int m_value;
};
```

`const` 的成员函数不能调用非 `const` 的成员函数，原因在于 `const` 的成员函数保证了不修改对象状态，但是如果调用了非 `const` 成员函数，那么这个保证可能会被破坏。

