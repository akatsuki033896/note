
## 01 视 C++ 为一个语言联邦

C++ 分为 C / Object-Oriented C++ / Template C++ / STL

## 02 尽量以 `const` / `enum` / `inline` 替换 `#define`

- 作用类似函数的宏可以用 `inline` 函数替换，类似常量的宏用 `const`
- 如果实在需要调用宏，记得加 `()` 

```cpp
#define ASPECT_RATIO 1.653 // 预处理器只知道 1.653，不知道 ASPECT_RATIO
const double aspect_ratio = 1.653; // 编译器知道变量名字
```

实际上就是把预处理做的事情给编译器做

## 03 尽量用 `const`

`const` 防止不小心改变值而出错。`const` 在 `*` 左边是被指物是常量，在 `*` 右边是指针本身是常量，`*` 两边都有是指针和被指物都是常量，和 `const` 写在类型左边和右边没关系。

```cpp
char* p = greeting;
const char* p = greeting;
char* const p = greeting;
const char* const p = greeting;
```

`const` 也是 STL 的迭代器的基础，迭代器类似 `T*` 指针，指向不同的东西的行为是不可更改的，但是可以更改指向的东西本身。

`const` 修饰成员函数

```cpp
int func(int a, int b) const
{
	a = sizeof(b);
	return a;
} // 返回值不可更改
mutable int a; // const 函数内允许被更改
```
