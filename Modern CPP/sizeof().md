---
tags:
---

经典问题

```c
#include <iostream>
#include <string.h>

struct MyStruct {
    int a;
    double b;
    char c;
}; // 24byte 边界对齐

int arr[10]; // 40byte

struct EmptyStruct {}; // 空结构体（或空类）占1字节，是因为编译器需要保证每个实例在内存中都有独一无二的地址。这1字节用于标记结构体的存在

int main() {
    // char str[] = "Hello, World!";
    // std::cout << sizeof(str) << std::endl; // sizeof
    // std::cout << strlen(str) << std::endl; // strlen

    // char* p = str; // 指向 str 的指针固定8字节(64bit)
    // std::cout << sizeof(p) << std::endl;

    MyStruct myStruct;
    std::cout << sizeof(myStruct) << std::endl;

    std::cout << sizeof(arr) << std::endl;
    std::cout << sizeof(EmptyStruct) << std::endl;
    return 0;
}
```