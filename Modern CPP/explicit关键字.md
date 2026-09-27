用于单参数构造函数，防止编译器进行隐式转换

**隐式转换是编译器自动发生的，而显式转换则需人为指定**。  
使用 `explicit` 可以让类对象转换更安全，需通过明确的 `static_cast` 或类型转换符号进行转换。

## 隐式转换

当一个类拥有单参数构造函数时（或单参数且其余参数有默认值），编译器会自动使用该参数类型的值去调用构造函数，生成临时对象。这种转换可能会隐藏逻辑错误。

```cpp
class MyString {
public:
    MyString(int size) {} // 接受 int 的构造函数
};
MyString s = 10; // 隐式转换：将 int 隐式转换为 MyString 对象
```

```cpp
class MyString {
public:
    explicit MyString(int size) {} // 使用 explicit
};
// MyString s = 10; // 错误：无法隐式转换
MyString s(10);     // 正确：直接初始化
MyString s2 = MyString(10); // 正确：显式类型转换
```

## 显式转换

明确指出需要从一种类型转换为另一种类型。即使使用了 `explicit`，程序员也可以通过显式地调用构造函数来进行转换。

`static_cast<Type>(value)` 或构造函数强制转换。
