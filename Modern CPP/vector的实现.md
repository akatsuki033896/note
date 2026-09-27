---
tags:
  - STL
---
这是一个最基本的实现，仅仅考虑这个容器存储 `int` 类型数据。除了构造和析构函数，还有返回容器的大小的 `size()`，重载 `[]` 运算符使得像数组一样访问。

```cpp
#include <iostream>
#include <cstdio>
#include <vector>

struct Vector {
    int *m_data;
    size_t m_size;

    Vector(size_t n) {
        m_data = new int[n];
        m_size = n;
    }
    size_t size() {
        return m_size;
    }
    int operator[](size_t i) {
        return m_data[i];
    }
    ~Vector() {
        delete[] m_data;
    }
};
```

注意到 `size()` 和重载 `[]` 都是只读的操作，在声明中用 `const` 修饰

```cpp
size_t size() const {
        return m_size;
    }
int operator[](size_t i) const {
    return m_data[i];
}
```

尝试在主函数对容器元素赋值例如  `a[1] = 1`，报错：Expression is not assignable (clang typecheck_expression_not_modifiable_lvalue)，说明容器元素 `vec[i]` 还不是左值。给 `[]` 添加一个重载，返回 `int&` 左值引用类型，这样就可以修改引用

```cpp
int& operator[](size_t i) {
    return m_data[i];
}
```

对返回右值的重载，如果不是 `int` 而是另一个容器的类型，拷贝开销很大，改为返回引用

```cpp
int const &operator[](size_t i) const{
    return m_data[i];
} // 只读, 返回引用减少开销
```

这个构造函数是隐式构造函数，加上 `explicit` 防止隐式类型转换，也就是不许 `Vector vec = 3` 这么构造必须加括号。标准库同样不许这么做。补上 `{}` 这样初始化的时候不会是内存的随机值。

```cpp
explicit Vector(size_t n) {
    m_data = new int[n] {};
    m_size = n;
}
```

在不指定长度的时候，`vector` 一开始长度应该是空的，然后根据使用情况使用 `resize()` 再给他一个长度。使用默认构造函数实现，增加成员函数 `resize()`，实现在 `resize()` 的时候把旧数据拷贝过去的功能，拷贝完后要把旧的数据释放回内存

```cpp
Vector() {
    m_data = nullptr;
    m_size = 0;
}
void resize(size_t n) {
    auto *old_data = m_data;
    auto old_size = m_size;
    m_data = new int[n] {};
    m_size = n;
    if (old_data) {
        memcpy(m_data, old_data, old_size * sizeof(int));
        delete[] old_data;
    }
}
```

考虑 `resize()` 后变小的情况，要把多的地方砍掉，`memcpy` 第三个参数改为 `std::min(old_size, m_size) * sizeof(int)`

`clear()` 等价于 `resize()` 到大小为 `0`，就类似析构函数，增加 `resize(0)` 的特殊情况

```cpp
void resize(size_t n) {
    auto *old_data = m_data;
    auto old_size = m_size;
    if (n == 0) {
        m_data = nullptr;
        m_size = 0;
    }
    else {
        m_data = new int[n] {};
        m_size = n;
    }
    if (old_data) {
        size_t copy_size = std::min(old_size, m_size);
        if (copy_size) {
            memcpy(m_data, old_data, copy_size * sizeof(int));
        }
        delete[] old_data;
} // 释放旧的数据

void clear() {
    resize(0);
}
```

考虑拷贝一个对象的情况，一旦定义了析构函数，必须同时定义拷贝函数和移动构造函数（只定义拷贝也行），否则编译器会默认生成拷贝函数，这个拷贝函数对指针成员是浅拷贝，只拷贝了指针，不拷贝指针指向的对象。

```cpp
Vector (Vector const& that) {
    m_data = that.m_data;
    m_size = that.m_size;
} // 浅拷贝 编译器生成的

Vector (Vector const& that) {
    m_size = that.m_size;
    m_data = new int [m_size];
    memcpy(m_data, that.m_data, m_size * sizeof(int));
} // 深拷贝
```

之后给所有函数加上判空条件。补上 `push_back()`，`at()`（判空的访问元素），`front` `back` 和他们的引用类型访问版本

```cpp
int& front() {
    return operator[](0);
}

int const& front() const {
    return operator[](0);
}

int& back() {
    return operator[](size() - 1);
}

void push_back(int val) {
    resize(size() + 1);
    back() = val;
}

int& at(size_t i) {
  if (i >= m_size) [[unlikely]] {
      throw std::out_of_range("vector::at");
  }
  return m_data[i];
}
...
```

根据标准，拷贝是 $O(n)$ 复杂度，还要移动构造函数（$O(1)$ ）和移动赋值函数，记得清空被移动的数据，并且注意移动不会申请新的内存，只是通过指针移动转移了数据，也就不会抛出异常 `out_of_memory` 的，同样 `front()` `back()` 析构函数也不会，给这些不会抛出异常的函数加上 `noexcept` 修饰

```cpp
Vector (Vector&& that) noexcept {
    m_data = that.m_data;
    m_size = that.m_size;
    that.m_data = nullptr;
    that.m_size = 0;
    // 清空
}

Vector &operator=(Vector&& that) noexcept {
    clear();
    m_data = that.m_data;
    m_size = that.m_size;
    that.m_data = nullptr;
    that.m_size = 0;
    return *this;
}
...
```

补一个构造函数，把所有元素初始化成指定值的重载

```cpp
explicit Vector(size_t n, int val) {
    m_data = new int[n]; // 初始化后不会有内存的随机值
    m_size = n;
    m_cap = n;
    for (size_t i = 0; i < m_size; i++) {
        m_data[i] = val;
    }
}
```

实现 `erase()` 删除指定索引的元素，只需把后面的一个元素覆盖到前面的，再把最后一个删掉。因为之后不仅仅 `int`  要考虑其他复杂类型，用移动语义开销更小。还要实现它的另一个重载，也就是删除一个索引范围的元素。这个重载在标准库里会返回一个迭代器，但是现在先不实现它。

```cpp
void erase(size_t i) {
    for (size_t j = i + 1; j < m_size; j++) {
        m_data[j - 1] = std::move(m_data[j]);
    }
    resize(m_size - 1);
}
void erase(size_t ibeg, size_t iend) { // ibeg <= iend
    size_t diff = iend - ibeg;
    for (size_t j = iend; j < m_size; j++) {
        m_data[j - diff] = std::move(m_data[j]);
    }
    resize(m_size - diff);
}
```

一开始实现的 `push_back()` 每次调用都要调用 `resize()`，`resize()` 复杂度是 $O(N)$，连续 `push_back()` 的复杂度就是 $O(N^2)$，实际上标准库因为把 `size` 和 `capacity` 解耦了。`capacity` 的增长过程是公比为 $2$ 的等比数列，单次是 $O(1)+$，全过程总体是 $O(N)$，因此标准库 `push_back` 复杂度只有 $O(n)$

实现 `capacity`，类似 `size`，添加成员 `m_cap` 和接口，再修改构造函数

```cpp
struct Vector {
  ...
    size_t m_cap;
  ...
    size_t capacity() const noexcept {
        return m_cap;
    }
}
```

实际上刚才实现的 `resize()` 是对 `capacity` 的操作。但是这不是一个公开的API，将它重命名为 `_recap`，里面对 `m_size` 操作也改成对 `m_cap` 操作，但是拷贝的部分还是拷贝的 `m_size`

真正的 `resize()` 只对 `m_size` 直接更改，同时扩充 `m_cap`，实现 `reserve()` 来扩充 `m_cap`

```cpp
void resize(size_t n) {
    reserve(n);
    m_size = n;
}

void reserve(size_t n) {
    if (n <= m_cap) [[likely]] {
        return;
    }
    n = std::max(n, m_cap * 2);
    printf("Grow from %zd to %zd\n", m_cap, n);
    auto old_data = m_data;
    if (n == 0) {
        m_data = nullptr;
        m_cap = 0;
    }
    else {
        m_data = new int[n] {};
        m_cap = n;
    }
    if (old_data) {
        if (m_size) {
            memcpy(m_data, old_data, m_size * sizeof(int));
        }
        delete [] old_data;
    }
}
```

与 `reserve` 相对还有一个 `shrink_to_fit` 函数，调用之后 `capacity` 就会等于 `size`，已分配的内存 `clear` 是释放不掉的。

```cpp
void shrink_to_fit(size_t n) noexcept {
    m_cap = m_size;
    auto old_data = m_data;
    if (m_data == 0) {
        m_data = nullptr;
    }
    else {
        m_data = new int[m_size];
    }
    if (old_data) {
        if (m_size != 0) {
            memcpy(m_data, old_data, m_size * sizeof(int));
        }
        delete [] old_data;
    }
}
```

有些函数例如 `erase()` 实际上是接受迭代器作为参数，为了实现迭代器 `begin` 和 `end`，先假设它返回 `int` 类型指针

```cpp
// 迭代器
int* begin() {
    return m_data;
}

int* end() {
    return m_data + m_size;
}

int* const begin() const {
    return m_data;
}

int* const end() const{
    return m_data + m_size;
}
```

接下来修改 `erase()` 为接受迭代器作为参数，迭代器必须是 `const` 的

```cpp
void erase(int const *it) {
    size_t i = it - m_data;
    for (size_t j = i + 1; j < m_size; j++) {
        m_data[j - 1] = std::move(m_data[j]);
    }
    resize(m_size - 1);
}

void erase(int const *first, int const *last) { // ibeg <= iend
    size_t diff = last - first;
    for (size_t j = last - m_data; j < m_size; j++) {
        m_data[j - diff] = std::move(m_data[j]);
    }
    resize(m_size - diff);
}
```

修改 `resize`，若 `resize` 的大小比 `m_size` 大，要将多出来的地方初始化为 `val` ，默认初始化 `val=0`，同时优化 `push_back` 这样拷贝次数减少一次。

```cpp
void resize(size_t n, int val = 0) {
    reserve(n);
    if (n > m_size) {
        for (size_t i = m_size; i < n; i++) {
            m_data[i] = val;
        }
    }
    m_size = n;
}
void push_back(int val) {
    // resize(size() + 1);
    // back() = val;
    reserve(m_size + 1);
    m_data[m_size] = val;
    m_size++;
}
```

实现从两个迭代器中构造 `vector`，在标准库里的声明为：

```cpp
template< class InputIt >
vector( InputIt first, InputIt last, const Allocator& alloc = Allocator() );
```

`InputIt` (输入迭代器)在 C++ 中是一种迭代器概念，代表了可以**单向、只读访问**序列元素的迭代器，通常用于算法如 [std::distance](https://www.google.com/search?q=std%3A%3Adistance&oq=c%2B%2B+InputIt&gs_lcrp=EgZjaHJvbWUyBggAEEUYOTILCAEQABgKGBMYgAQyBwgCEAAY7wUyBwgDEAAY7wUyBwgEEAAY7wXSAQk0MTI2ajBqMTWoAgiwAgHxBQzLU3GvVUlp&sourceid=chrome&ie=UTF-8&mstk=AUtExfC_tZRx1FMeHA7NRvJ5-BH1OoY_F0Bv3clAy9FpSxoXYgIRUqVSh649IthItX9Oanvq8KbgTl0VcRJTGWRpJ_MmAzKoseu4lX8-tO7WM4C7kTe2yzPl2c2xUR5lwypjCI0zX3vS5EWJ7RwXPNTOTHGaiFnKXAf8kUjlbgHsVRAmGpI&csui=3&ved=2ahUKEwjh1aXT8LySAxUrr1YBHcaQPbMQgK4QegQIARAB)、[std::for_each](https://www.google.com/search?q=std%3A%3Afor_each&oq=c%2B%2B+InputIt&gs_lcrp=EgZjaHJvbWUyBggAEEUYOTILCAEQABgKGBMYgAQyBwgCEAAY7wUyBwgDEAAY7wUyBwgEEAAY7wXSAQk0MTI2ajBqMTWoAgiwAgHxBQzLU3GvVUlp&sourceid=chrome&ie=UTF-8&mstk=AUtExfC_tZRx1FMeHA7NRvJ5-BH1OoY_F0Bv3clAy9FpSxoXYgIRUqVSh649IthItX9Oanvq8KbgTl0VcRJTGWRpJ_MmAzKoseu4lX8-tO7WM4C7kTe2yzPl2c2xUR5lwypjCI0zX3vS5EWJ7RwXPNTOTHGaiFnKXAf8kUjlbgHsVRAmGpI&csui=3&ved=2ahUKEwjh1aXT8LySAxUrr1YBHcaQPbMQgK4QegQIARAC) 等，它允许向前移动，但不能直接反向，是一种基础但强大的类型，是 STL 中用于读取数据流的基础迭代器。 标准库里 `vector` 的 `begin()` 和 `end()` 也符合 `InputIt`

现在来实现用它来构造 `vector`，`std::randon_access_iterator` 限定了只能是一个迭代器。同理应该有一个重载运算符 `=` 的版本，但是重载运算符只能接受一个参数，所以这个表现为 `assign()` 成员函数，`assign()` 还要一个接受一个大小作为参数的重载。

```cpp
// 用迭代器构造, 迭代器可以是任意类型
template <std::random_access_iterator InputIt>
Vector(InputIt first, InputIt last) {
    size_t n = last - first;
  	m_data = new int[n];
    m_size = n;
    m_cap = n;
    for (size_t i = 0; i < n; i++) {
        m_data[i] = *first;
        ++first;
    }
}

void assign(size_t n, int val) {
    reserve(n);
    m_size = n; // 分配新的大小
    for (size_t i = 0; i < n; i++) {
        m_data[i] = val;
    }
}

template <std::random_access_iterator InputIt>
void assign(InputIt first, InputIt last) {
    size_t n = last - first;
    reserve(n);
    m_size = n; // 分配新的大小
    for (size_t i = 0; i < n; i++) {
        m_data[i] = *first;
        ++first;
    }
}
```

接下来实现 `insert()` 在指定的位置插入一个值，倒着遍历类似于 `memmove()`

```cpp
void insert(int const *it, size_t n, int val) {
    size_t j = it - m_data; // 插入位置
    if (n == 0) [[unlikely]] {
        return;
    }
    reserve(m_size + n);
    m_size += n; // 分配新的大小
    // 移动元素
    for (size_t i = n; i > 0; i--) {
        m_data[j + n + i - 1] = std::move(m_data[j + i - 1]);
    }
    for (size_t i = j; i < j + n; i++) {
        m_data[i] = val;
    }
}

template <std::random_access_iterator InputIt>
void insert(int const *it, InputIt first, InputIt last) {
    size_t j = it - m_data; // 插入位置
    size_t n = last - first; // 插入 n 个元素
    if (n == 0) [[unlikely]] {
        return;
    }
    reserve(m_size + n);
    m_size += n; // 分配新的大小
    // 移动元素
    for (size_t i = n; i > 0; i--) {
        m_data[j + n + i - 1] = std::move(m_data[j + i - 1]);
    }
    for (size_t i = j; i < j + n; i++) {
        m_data[i] = *first;
        ++first;
    }
}
```

实现初始化列表构造，C++11新增特性 在构造函数里调用另一个构造函数但必须冒号，再补上 `assign()` `insert()` 接受初始化列表为参数的版本

```cpp
// 初始化列表构造
Vector(std::initializer_list<int> ilist) : Vector(ilist.begin(), ilist.end()) {}
...
void assign(std::initializer_list<int> ilist) {
    assign(ilist.begin(), ilist.end());
}

void insert(int const *it, std::initializer_list<int> ilist) {
    insert(it, ilist.begin(), ilist.end());
}
```

考虑之后要模板化，对于拷贝构造，`memcpy` 之所以可以实现是因为 `int` 是 POD 类型，可以频繁拷贝，模板化之后可能接受复杂类型，要改成遍历更改

```cpp
if (m_size) {
    m_data = new int [m_size];
    for (size_t i = 0; i < m_size; i++) {
        m_data[i] = std::as_const(that.m_data[i]);
    }
    // memcpy(m_data, that.m_data, m_size * sizeof(int));
}
```

如果接受的是 `std::string` 他会有默认构造函数，开销很大，添加成员 `using allocator = std::allocator<int>;`，这样分配出来内存保证可以和 `int` 的4byte对齐，以此类推改掉所有 `memcpy` 为批量构造。

```cpp
Vector (Vector const& that) {
    m_size = that.m_size;
    m_cap = that.m_cap;
    if (m_size) {
        // m_data = new int [m_size];
        m_data = allocator{}.allocate(m_size); // 等于 malloc free
        for (size_t i = 0; i < m_size; i++) {
            // m_data[i] = std::as_const(that.m_data[i]);
            std::construct_at(&m_data[i], std::as_const(that.m_data[i]));
        }
        // memcpy(m_data, that.m_data, m_size * sizeof(int));
    }
    else {
        m_data = nullptr;
    }
}
```

使用分配器改进内存安全，`new` 全部改成 `allocator{}.allocate()`，`delete[]` 换成 `allocator{}.deallocate()`，注意 `deallocate()` 的第二个参数是实际已经分配的内存大小，改变前要额外记录旧的 `cap`。好的代码永远不能出现 `new` 和 `delete`

忘记写 `swap()`了。`swap()` 调用的是标准库的 `std::swap`，不是赋值。

```cpp
void swap(Vector& other) noexcept {
    std::swap(m_data, other.m_data);
    std::swap(m_size, other.m_size);
    std::swap(m_cap, other.m_cap);
}
```

接下来模板化。根据标准库，模板参数是 `template<class T, class Alloc = std::allocator<T>>`，还要添加成员别名。

```cpp
struct Vector {
    using value_type = T;
    using pointer = T*;
    using const_pointer = T* const;
    using reference = T&;
    using const_reference = T const&;
    using iterator = T*;
    using const_iterator = T* const;
    using reverse_iterator = std::reverse_iterator<T*>;
    using const_reverse_iterator = std::reverse_iterator<T* const>;
    using allocator = Alloc;
  ...
};
```

优化 `push_back()`，增加移动的重载，并改为就地赋值，效率更高

```cpp
void push_back(T const &val) {
    reserve(m_size + 1);
    // m_data[m_size] = val;
    std::construct_at(&m_data[m_size], val);
    m_size++;
} // 拷贝

void push_back(T &&val) {
    reserve(m_size + 1);
    std::construct_at(&m_data[m_size], std::move(val));
    m_size++;
} // 对于不许拷贝的类型 e.g. std::thread
```

实现 `emplace_back()`

```cpp
template <class ...Args>
T &emplace_back(Args &&...args) {
    reserve(m_size + 1);
    T *p = &m_data[m_size];
    std::construct_at(&m_data[m_size], std::forward<Args>(args)...);
    m_size++;
    return *p;
}
```

 
