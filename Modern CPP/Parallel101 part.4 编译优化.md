---
tags:
  - Parallel101
---
# 汇编语言

## gcc 编译器开启优化选项

[Compiler Explorer](https://godbolt.org/) 是一个便于查看编译器优化和汇编代码的C++在线运行平台。

```cpp
int func(int a, int b, int c, int d, int e, int f) {
    return a;
}
```

传入六个参数，返回参数 `a`，汇编为

```assembly
func(int, int, int, int, int, int):
        push    rbp
        mov     rbp, rsp
        mov     DWORD PTR [rbp-4], edi
        mov     DWORD PTR [rbp-8], esi
        mov     DWORD PTR [rbp-12], edx
        mov     DWORD PTR [rbp-16], ecx
        mov     DWORD PTR [rbp-20], r8d
        mov     DWORD PTR [rbp-24], r9d
        mov     eax, DWORD PTR [rbp-4]
        pop     rbp
        ret
```

1. 使用 `gcc -O` 或 `gcc -O1` 在不影响编译速度的前提下，尽量采用一些优化算法降低代码大小和可执行代码的运行速度，并开启一些化选项

2. `gcc -O2` 牺牲部分编译速度，除了执行 `-O1` 所执行的所有优化之外，还会采用几乎所有的目标配置支持的优化算法，用以提高目标代码的运行速度

3. `gcc -O3` 除了执行 `-O2` 所有的优化选项之外，一般都是采取很多向量化算法，提高代码的并行执行程度，利用现代CPU中的流水线 / Cache等，提高执行代码的大小，当然会降低目标代码的执行时间。在CMAKE中 `set(CMAKE_BUILD_TYPE Release)` 开启 `-O3`

4. `gcc -Os` 在-O2的基础之上，尽量的降低目标代码的大小，这对于存储容量很小的设备来说非常重要。为了降低目标代码大小，会禁用部分选项，一般就是压缩内存中的对齐空白(alignment padding)

使用这些优化得到的汇编都是：

```assembly
func(int, int, int, int, int, int):
        mov     eax, edi
        ret
```

`eax` 存放函数返回的结果

## 对数组大小和索引使用 `size_t` 类型

`size_t` 在 64 位系统上相当于 `uint64_t`，在 32 位系统上相当于 `uint32_t`，从而不需要用 `movslq` 从 32 位符号扩展到 64 位，更高效。而且也能处理数组大小超过 `INT_MAX `的情况，推荐始终用 `size_t` 表示数组大小和索引

```cpp
#include <cstdint>
int func(int *a, std::size_t b) {
    return a[b];
}
```

```assembly
func(int*, unsigned long):
				# movsx   rsi, esi 使用 int b 的情况
        mov     eax, DWORD PTR [rdi+rsi*4]
        ret
```

## SIMD：单个指令处理多个数据

SIMD（single-instruction multiple-data）是单个指令处理多个数据的技术，可以大大增加计算密集型程序的吞吐量

例如在一定条件下，编译器能够把一个处理标量 float 的代码，转换成一个利用 SIMD 指令的处理矢量 float 的代码，从而增强程序的吞吐能力，SIMD 把 4 个 float 打包到一个 `xmm` 寄存器（$128$ bit的寄存器，可存储 $4$ 个 `float` 或 $2$ 个 `double`）里同时运算，很像数学中矢量的逐元素加法。因此 SIMD 又被称为矢量，而原始的一次只能处理 1 个 float 的方式，则称为标量。

```cpp
float func(float a, float b) {
    return a + b;
}
```

```assembly
func(float, float):
        addss   xmm0, xmm1
        ret
```

在这个例子中 `xmm0` 存储 `a`，`xmm1` 存储 `b`

> 如果你看到编译器生成的汇编里，有大量 ss 结尾的指令则说明矢量化失败；如果看到大多数都是 ps 结尾则说明矢量化成功

# 编译器的代数化简

## 常量折叠

编译期直接算好了，甚至 $\sum_{i=1}^{100}=5050$ 都可以直接算，然而使用 `std::vector` 等容器时不能优化，因此**避免代码复杂化，避免使用在堆上分配内存，会造成 `new` `delete` 的容器，编译器就能自己帮你优化**

```cpp
int func() {
    int a = 30;
    int b = 12;
    return a + b;
}
```

```assembly
func():
        mov     eax, 42
        ret
```

存储在堆上的容器（妨碍优化，存储在栈上无法动态扩充大小）：`vector`,` map`,` set`,` string`,` function`, `any`, `unique_ptr`,` shared_ptr`, `weak_ptr`

存储在栈上的容器（促进优化）：`array`, `bitset`, `glm::vec`, `string_view`, `pair`, `tuple`,` optional`,` variant`

### `constexpr`：强迫编译期求值

如果发现编译器放弃了自动优化，可以用 `constexpr` 函数迫使编译器进行常量折叠，但 `constexpr` 函数中无法使用非 `constexpr` 的容器，这个关键字明确的告诉编译器应该去验证一个表达式在编译期是常量表达式

# 编译器的内联化

内联：当编译器看得到被调用函数的实现的时候，会直接把函数实现贴到调用他的函数里

只有定义在同一个文件的函数可以被内联。为了效率我们可以尽量把常用函数定义在头文件里，然后声明为 static。这样调用他们的时候编译器看得到他们的函数体，从而有机会内联。

```cpp
int other(int a) {
    return a;
}
int func() {
    return other(233);
}
```

```assembly
other(int):
        mov     eax, edi
        ret
func():
        mov     eax, 233
        ret
```

> 现代编译器的高强度优化下，加不加 `inline` 无所谓，编译器不是傻子，只要他看得见 other 的函数体定义，就会自动内联
>
> 内联与否只取决于是否在同文件，且函数体够小，需要性能的定义在头文件声明为 `static` 即可，`static` 纯粹是为了避免多个 .cpp 引用同一个头文件造成冲突，并不是必须 `static `才内联。如果你不确定某修改是否能提升性能，那你最好实际测一下，不要脑内模拟，`inline` 在现代 C++ 中有其他含义，但和内联没有关系

# 指针

主要两种语法

## 指针别名

这个调用下 `b` 和 `c` 指向了同一个变量，如果优化了之后 `b = b` 则 `b` 没有改变，所以编译器放弃优化

```cpp
void func(int *a, int *b, int *c) {
  *c = *a;
  *c = *b;
}
int main() {
  int a, b;
  func(&a, &b, &b);
}
```

### `__restrict` 向编译器保证这些指针之间不会发生重叠

`__restrict ` 只需要加在所有具有写入访问的指针（这里是 c）上，就可以优化成功，而我们可以用 `const` 禁止写入访问，因此所有非 `const` 的指针都声明 `__restric`

```cpp
void func(int const *a, int const *b, int *__restrict c) {
  *c = *a;
  *c = *b;
}
```

`__restrict` 对 `std::vector` 没用

### `volatile` 禁止优化

加了 `volatile `的对象，编译器会放弃优化对他的读写操作。做性能实验的时候非常有用，语法为 `volatile int *a `，注意与 `int *__restrict a` 区分

# 矢量化

## 让编译器自动检测当前硬件支持的指令集

`-march=native` 让编译器自动判断当前硬件支持的指令

## 合并写入

```cpp
void func(int *a) {
    a[0] = 111;
    a[1] = 222;
    a[2] = 333;
    a[3] = 444;
}
```

```assembly
func(int*):
        movdqa  xmm0, XMMWORD PTR .LC0[rip]
        movups  XMMWORD PTR [rdi], xmm0
        ret
.LC0:
        .long   111
        .long   222
        .long   333
        .long   444
```

`xmm0` 由 SSE 引入，是个 128 位寄存器，可以一次存储 4 个 `int`，或 4 个 `float`

## SIMD 加速

```cpp
void func(int *a) {
    for (int i = 0; i < 1024; i++) {
        a[i] = i;
    }
}
```

```assembly
func(int*):
        mov     edx, 4
        movdqa  xmm0, XMMWORD PTR .LC0[rip]
        lea     rax, [rdi+4096]
        movd    xmm1, edx
        pshufd  xmm1, xmm1, 0
.L2:
        movups  XMMWORD PTR [rdi], xmm0
        add     rdi, 16
        paddd   xmm0, xmm1
        cmp     rax, rdi
        jne     .L2
        ret
.LC0:
        .long   0
        .long   1
        .long   2
        .long   3
```

`movdqa`：加载四个 `int`, `paddd` 是四个 `int` 的加法，一次写入 4 个 `int`，一次计算 4 个 `int` 的加法，从而更加高效，但这样数组的大小必须为 4 的整数倍否则就会写入越界的地址。如果不是 $4$ 的倍数可以使用边界特判法，假如写入 $1023$ 个元素可以先对前 $1020$ 个元素用 SIMD 指令填入，每次处理 $4$ 个，剩下 $3$ 个元素用传统的标量方式填入，每次处理 $1$ 个，**对边界特殊处理，而对大部分数据能够矢量化**

如果能保证元素数总是 $4$ 的倍数，可以写 `n = n / 4 * 4`，编译器会发现 `n % 4 = 0`，从而不会生成边界特判的分支

# 循环

循环中的矢量化如果出现指针别名，会生成两份代码，一份是SIMD的，一份是传统标量的，检测两个指针的差是否超过 $1024$ 来判断是否重叠，如果没有重叠则跳到SIMD版本运行，否则运行标量版本。

1. 加上 `__restrict` 关键字可以只生成SIMD版本
2. 对gcc编译器可以用 `#pragma GCC ivdep` 表示忽视下方 for 循环内可能的指针别名现象

3. 循环里的 `if` 分支很难优化，最好移到外面来，就可以自由地使用 SIMD 指令
4. 循环中的不变量最好移到循环体外，提前计算，避免重复计算

## 循环展开

对于 GCC 编译器，可以用 `#pragma GCC unroll 4` 表示把循环体展开为4个

对小的循环体进行 unroll 可能是划算的，但最好不要 unroll 大的循环体，否则会造成指令缓存的压力反而变慢！

```cpp
void func(float *a) {
#pragma GCC unroll 4
    for (int i = 0; i < 1024; i++) {
        a[i] = 1;
    }
}
```

```assembly
func(float*):
        movss   xmm0, DWORD PTR .LC1[rip]
        lea     rax, [rdi+4096]
        shufps  xmm0, xmm0, 0
.L2:
        movups  XMMWORD PTR [rdi], xmm0
        add     rdi, 64
        movups  XMMWORD PTR [rdi-48], xmm0
        movups  XMMWORD PTR [rdi-32], xmm0
        movups  XMMWORD PTR [rdi-16], xmm0
        cmp     rax, rdi
        jne     .L2
        ret
.LC1:
        .long   1065353216
```

`movups` 重复了 $4$ 次

# 结构体

我们知道结构体存储边界对齐，对齐了才能矢量化。计算机喜欢 2 的整数幂，2, 4, 8, 16, 32, 64, 128... 结构体大小若不是 **2** 的整数幂，往往会导致 SIMD 优化失败，可以追加一个没用的变量让结构体大小变成 $2$ 的整数幂

```cpp
struct MyVec {
  float a;
  float b;
  float c;
  char padding[4];
}
```

## `alignas`：对齐引入

在 struct 后加上 `alignas(要对齐到的字节数)` 即可实现同样效果，就不需要手动写 `padding `变量了

```cpp
struct alignas(16) MyVec {
  float a;
  float b;
  float c;
}
```

不要对所有结构体打上 `alignas`，有可能不仅不变快，反而还变慢！SIMD 和缓存行对齐只是性能优化的一个点，又不是全部。还要考虑结构体变大会导致内存带宽的占用，对缓存的占用等一系列连锁反应，总之，要根据实际情况选择优化方案。

## 结构体的内存布局

### AOS（Array of Struct）

单个对象的属性紧挨着存，例如 `xyzxyzxyzxyz`，符合一般面向对象编程 (OOP)的习惯，但常常不利于性能

```cpp
struct alignas(16) MyVec {
  float x;
  float y;
  float z;
};
MyVec a[1024];
```

### SOA：分离存储多个属性

不符合面向对象编程 (OOP) 的习惯，例如 `xxxxyyyyzzzz` 但常常有利于性能。又称之为面向数据编程 (DOP)

```cpp
struct alignas(16) MyVec {
  float x[1024];
  float y[1024];
  float z[1024];
};
MyVec a;
```

### AOSOA

4 个对象一组打包成 SOA，再用一个 `n / 4` 大小的数组存储为AOS

```cpp
struct alignas(16) MyVec {
  float x[4];
  float y[4];
  float z[4];
};
MyVec a[1024 / 4];
```

`std::vector` 也可以实现 AOS 和 SOA

# 数学运算

1. 将除法优化成乘法（乘法更快）例如把 `a / 2` 优化成 `a * 0.5f`

2. 编译器不会分离公共除数，因为除数可能为零，可以提前算好倒数，还可以开启编译器 `-ffast-math` 选项（如果你能保证程序中永远不会出现 `NaN` 和无穷大）
   ```cpp
   void func(float *a, float b) {
     float inv_b = 1 / b;
     for (int i = 0; i < 1024; i++) {
       a[i] *= inv_b;
     }
   }
   ```

3. 数学函数前加 `std::`，请勿用全局的数学函数，他们是C 语言的遗产。始终用 `std::sin`等

# 优化手法省流

1. 函数尽量写在同一个文件内
2. 避免在 `for` 循环内调用外部函数
3. 非 `const` 指针加上 `__restrict `修饰
4. 试着用 SOA 取代AOS
5. 对齐到 16 或 64 字节
6. 简单的代码，不要复杂化
7. 试试看 `#pragma omp simd`
8. 循环中不变的常量挪到外面来
9. 对小循环体用 `#pragma unroll`
10. `-ffast-math` 和 `-march=native`