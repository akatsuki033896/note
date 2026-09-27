# C 语言的 `char` 类型

C语言只规定了 `unsigned char` 是无符号8位整数，`signed char` 是带符号8位整数

`char` 是8位整数，可以是有符号也可以是无符号，由编译器决定。gcc规定 `char` 在 x86 架构等价于 `signed char`，而在 arm 架构上则等价于 `unsigned char`，而C++ 标准保证  `char`，`signed char`，`unsigned char` 是三个完全不同的类型，`std::is_same_v` 分别判断他们总会得到 false，无论 x86 还是 arm

## C 语言的字符串

字符串是字符组成的数组

```c
char c = 'h';
// 等价
char s[] = "str";
char s[] = {'s', 't', 'r', '0'};
```

末尾的是ASCII码的空字符，表示数组结尾，可以利用这个特性在原本非`0`的字符处写入`0`来提前结束字符串。

# `std::string`

```c++
std::string s = "str";
string(“hello”) + string(“world”) == string(“helloworld”) 
```

1. 可以从 `const char*` 隐式构造
2. 运算符重载
3. 符合 `vector<char>` 的接口，例如 `begin/end/size/resize`，很多成员函数
4. 通过 `c_str()` 重新转换回 `const char *`
5. 离开作用域时自动释放内存 

## 构造

- C++字符串是`string` 类，包含 `char *ptr; size_t len` 两个成员，`len`用来确定结尾的位置，不需要 `\0` 结尾

- C的字符串是单独一个 `char *ptr`，自动以 `‘\0’` 结尾

## `c_str()` ：获取C类型的字符串

- `s.c_str()` 返回以 `\0` 结尾的字符串首地址指针，总长度为 `s.size() + 1`，`s.data()` 只保证返回长度为 `s.size()` 的连续内存的首地址指针，不保证 0 结尾
- 把 C++ 的 `string` 作为参数传入像 printf 这种 C 语言函数时，需要用 `s.c_str()`
- 如果只是在 C++ 函数之间传参数，直接用 `string` 或 `string const &` 即可

```cpp
void legacy_c(const char *name);                 // 这个函数是古老的 C 语言遗产
void modern_cpp(std::string name);             // 这个函数是现代 C++，便民！
void performance_geek(std::string const &name);    // 有点追求性能的极客
void performance_nerd(std::string_view name);       // 超级追求性能的极客
```

### C 和 C++ 风格字符串的转换

- `const char *` 可以隐式转换为 `string`（为了方便）
- `string` 不可以隐式转换为 `const char *`（安全起见）
- **如果确实需要从 `string` 转换为 `const char *`，请调用 `.c_str()` 这个成员函数**

## 字符串连接

C 语言规定双引号包裹的字符串是 `const char *` ，他们没有 `+` 运算符，C++为了兼容重载了`+`，需要把两个字符串加在一起，就必须至少有一方是 `string`，用 `string(“hello”)` 这种形式包裹住每个字符串常量，这样就方便用 `+`  

```cpp
“hello” + “world”                      // 错误
string(“hello”) + “world”              // 正确
“hello” + string(“world”)              // 正确
string(“hello”) + string(“world”)      // 正确（推荐）
```

### C++14：自定义字面量后缀

标准库和 `std::literals` 定义了 `s`后缀，更方便

```cpp
using namespace std;
// using namespace literals;
string s3 = "hello"s + "world"s; // 等价于string(“hello”) + string(“world”)
```

## 字符串与数字

### `std::to_string` 数字转字符串

`std::to_string` 是标准库定义的全局函数，他具有9个针对不同类型的[重载](https://en.cppreference.com/cpp/string/basic_string/to_string)

```cpp
int n = 42;
auto s = to_string(n) + "yuan"s; // 42yuan
```

同理还有 `std::to_wstring` 转为宽字符串 `wstring`

### `std::sto*` 字符串转数字

`std::stoi/stof/stod` 也是标准库定义的一系列全局函数，也有不同[重载](https://en.cppreference.com/w/cpp/string/basic_string/stol)，实现字符串转数字

```cpp
auto s = "42"s;
int n = stoi(s); // 42
```

#### `std::stoi`

```cpp
int       stoi ( const std::string& str,
                 std::size_t* pos = nullptr, int base = 10 );
```

- 可以处理数字后面有多余字符的情况，例如 `stoi(“42yuan”)` 和 `stoi(“42”) `等价，都会返回 42
- 如果字符串的开头不是数字，则会抛出 `std::invalid_argument` 异常，可以用 `catch` 捕获。但是可以开头有空格。有正负号会被当成数字的一部分例如负数。
- 第二个参数默认为 `nullptr`，若不为空则会往他指向的变量写入一个整数，表示数字部分结束的那个字符所在的位置，例如`stoi(“42yuan”, &pos)` 会返回 42，并把 `pos` 设为 2（从0开始计算）
- 第三个参数表示进制，默认10进制，使用时如果不想指定 `pos` 可以令他为 `nullptr`，因为默认十进制所以 `stoi("7cfe")` 会不识别字母，得到7。16进制时字母可以任意大小写。

## 字符串流

`cout` 可以代替 `printf()` 输出字符串，不需要指定 `%`，还可以通过 `cout << hex` 输出16进制，如果不想要输出到控制台而是存进字符串可以使用 `stringstream`。

### `stringstream`

模仿 `cout`，取代 `to_string`

```cpp
#include <sstream>
stringstream ss;
ss << hex << 42; // 类似 cout
string s = ss.str();
```

使用 `.str()` 取出字符串

同理模仿 `cin` 取代 `sto*()`

```cpp
string s = "42yuan"s;
stringstream ss(s);
int num;
ss >> num; // 42
```

## 字符串常用操作

### `at()`：获取指定位置字符

`s.at(i)` 和 `s[i]` 都能获取第 `i` 个字符，遇到诡异 bug 时，试试把 `[]` 都改成 `at`

| `s.at(i)`                                                    | `s[i]`                                                       |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| 检测到 `i ≥ s.size()` 时，会抛出 `std::out_of_range` **异常**终止程序 | 不会抛出异常，给字符串的首地址指针和 `i` 做个加法运算，得到新的指针并解引用，**未定义行为** |
| 性能低，越界检测要额外开销                                   | 性能高                                                       |

### 获取字符串长度

`s.size()` 等价于 `s.length()`，`size()` 类似 `vector`

### `substr()` 切下子字符串

```cpp
string substr(size_t pos = 0, size_t len = -1) const;
```

- 截取从第 `pos` 个字符开始，长度为 `len` 的子字符串，原字符串不会改变
- 如果原字符串剩余部分长度不足 `len`，则返回长度小于 `len` 的子字符串而不会出错，`len` 默认为 -1（即 `string::npos`）指的是此时会截取从 `pos` 开始直到原字符串末尾的子字符串例如 `hello.substr(1) = "ello"`
- 如果 `pos` 超出了原字符串的范围，则抛出 `std::out_of_range` 异常

### `string::npos` 静态常量

等价于 `(size_t)-1`

### `find()` 寻找子字符串

```cpp
size_t find(char c, size_t pos = 0) const noexcept;
size_t find(string_view svt, size_t pos = 0) const noexcept;
size_t find(string const &str, size_t pos = 0) const noexcept;
size_t find(const char *s, size_t pos = 0) const;
size_t find(const char *s, size_t pos, size_t count) const;
```

最后两个重载没有 `noexcept` 意为不会抛出异常，返回 `-1`

- `find('c')` 返回字符串中第一个 `'c'` 的位置，找不到返回 `-1`

- `find(‘c’, pos)` 从 `pos` 位置开始找，返回字符串中第一个 `'c'` 的位置，找不到返回 `-1`，`pos` 也是从0开始计数

- `find(“str”, pos, len)` 和 `find(“str”.substr(0, len), pos)` 等价，用于要查询的字符串已经确定长度，或者要查询的字符串是个切片 `string_view` 的情况。若不指定这个长度，则默认是 C 语言的 0 结尾字符串，`find` 还要去求 `len = strlen(“str”)`，相对低效。

### `rfind()` 反向查找

- 从尾部开始查找，返回最后一次出现的地方，`helloworld”.rfind(‘l’)` 会返回 8。

- `rfind` 和 `find` 的最坏复杂度都为 O(n)，最好复杂度都为 O(1)

### `find_first_of` 寻找集合内任意字符

```cpp
size_t find_first_of(string const &s, size_t pos = 0) const noexcept;
size_t find_first_of(const char *s, size_t pos = 0) const noexcept;
size_t find_first_of(const char *s, size_t pos, size_t n) const noexcept;
```

`“str”.find_first_of(“chset”, pos)` 会从第 `pos` 个字符开始，在 `“str”` 中找到第一个出现的 ‘c’ 或 ‘h’ 或 ‘s’ 或 ‘e’ 或 ‘t’ 字符，并返回他所在的位置。如果都找不到，则会返回 -1，这个 “chset” 是个字符的集合，顺序无所谓，重复没有用

```cpp
string s = "helloworld"s;
int n = s.find_first_of("onl");
```

还有 `find_first_not_of` 寻找不在集合内的字符，

### `replace()` 替换一段子字符串

```cpp
string &replace(size_t pos, size_t len, string const &str);
string &replace(size_t pos, size_t len, const char *s);
string &replace(size_t pos, size_t len, const char *s, size_t slen);
```

- `replace(pos, len, “str”)` 会把从 `pos` 开始的 `len` 个字符替换为 `“str”`
- 如果 `len` 过大，超出了字符串的长度，则会自动截断，相当于从 `pos` 开始到末尾都被替换
- 如果 `pos` 过大，超出了字符串的长度，则抛出 `out_of_range` 异常
- 因为 -1 转换为 `size_t` 后根据补码的规则，他实际上变成 `0xffffffffffffffff`，所以可以给 `len` 指定 `-1`来迫使 replace 从某个数 开始一直到字符串末尾都替换掉，例如 `“helloworld”.replace(4, -1, “pful”)` 会得到 `“helpful”`

性能：

- string 的本质和 vector 一样，是内存中连续的数组，替换的过程中需要预留出空格进行平移，最坏是O(n)复杂度，如果原来的子字符串和新的子字符串一样长度，就直接覆盖上去，最好是O(1)复杂度

- `replace` 会就地修改原字符串，返回的是指向原对象的引用，并不是一份新的拷贝

### `append()` 追加一段字符串

```cpp
string &append(string const &str);                    // str 是 C++ 字符串类 string 的对象
string &append(const char *s);                         // s 是长度为 strlen(s) 的 0 结尾字符串
string &append(string const &str, size_t len);   // 只保留后 str.size() - len 个字符
string &append(const char *s, size_t len);        // 只保留前 len 个字符
```

- `s.append(“world”)` 和 `s += “world”` 等价（前面两个）
- 可以指定第二个参数，限定字符串长度，用于要追加的字符串已经确定长度，或者是个切片的情况

- append 的扩容方式和 `vector` 的 `push_back` 一样，每次超过 capacity 就预留两倍空间，所以重复调用 append 的复杂度其实是 **amortized O(n)** 的

C++17支持 `string_view` 更方便

- `s.append(“world”, 3)` 改成 `s += string_view(“world”).substr(0, 3)`
- `s.append(“world”s, 3)` 改成 `s += string_view(“world”).substr(3)`

### `insert()` 插入一段字符串

通常只用前两个就行，就地修改字符串，返回指向自身引用

```cpp
string &insert(size_t pos, string const &str);                    // str 是 C++ 字符串类 string 的对象
string &insert(size_t pos, const char *s);                         // s 是长度为 strlen(s) 的 0 结尾字符串
string &insert(size_t pos, string const &str, size_t len);   // 只保留后 str.size() - len 个字符
string &insert(size_t pos, const char *s, size_t len);        // 只保留前 len 个字符
```

`s.insert(pos, str)` 会把子字符串 `str` 插入到原字符串中第 `pos` 个字符和第 `pos+1` 个字符之间

### 字符串比较

- `==、!=、>、<、>=、<=` 这些运算符使用字典序比较：按 ASCII 码比较，比到最后一位都相等时，长的字符串大于短的字符串，相同长度就一样。
- C语言有 `strcmp(a, b)`，返回 -1 代表 a < b，返回 1 代表 a > b，返回 0 代表 a == b
- 类似的 `string` 有 `compare()`，也能这么返回，`a == b` 和 `!a.compare(b)` 等价
- C++20支持 `<=>` 万能比较运算符代替 `compare()`

### C++20：`starts_with` 和 `ends_with`

不会抛出异常，只会返回真假

- `s.starts_with(str)` 等价于 `s.substr(0, str.size()) == str`
- `s.ends_with(str)` 等价于 `s.substr(str.size()) == str`

- `“hello”.starts_with(‘h’)` 等价于 `“hello”.size() != 0 && “hello”[0] == ‘h’`

# 字符串胖指针

## 胖指针

描述一个动态长度的数组（此处为字符串），需要首地址指针和数组长度两个参数，把 **ptr** 和 **len** 这两个逻辑上相关的参数绑在一起，避免程序员犯错。为了表示动态长度的数组，C++ 中的 vector 和 string 其实都是胖指针。

```cpp
struct FatPtr {
   char *ptr;
   size_t len;
};

struct vector {
  char *ptr;
  size_t len;
  size_t capacity;
};
```

对于动态数组，`[ptr, len]` 其实就是表示实际有效范围（存储了字符的）的胖指针， `[ptr, capacity]` 就是表示实际已分配内存（操作系统认为的）的胖指针。

## 字符串胖指针

 `string` 是掌握着字符串生命周期（lifespan）的胖指针，这种掌管了所指向对象生命周期的指针称为强引用（strong reference）

<img src="/Users/akatsuki/Desktop/Screenshot 2026-08-31 at 20.47.53.png" alt="Screenshot 2026-08-31 at 20.47.53" style="zoom:50%;" />

强引用指的是：

- 容器被拷贝时，其指向的字符串也会被拷贝（深拷贝）
- 容器被销毁时，其指向的字符串也会被销毁（内存释放）

如果把一个强引用的 `string` 到处拷贝来拷贝去，则其指向的字符串也会被多次拷贝，比较低效。人们常用 `string const &` 来避免不必要拷贝，但仍比较麻烦。因此 C++17 引入了弱引用胖指针 `string_view`，这种弱引用（weak reference）不影响原对象的生命周期，原对象的销毁仍然由强引用控制。

弱引用指的是：

- 当 `string_view` 被拷贝时，其指向的字符串仍然是同一个（浅拷贝）
- 当 `string_view` 被销毁时，其指向的字符串仍存在（弱引用不影响生命周期）

![Screenshot 2026-08-31 at 20.57.24](/Users/akatsuki/Desktop/Screenshot 2026-08-31 at 20.57.24.png)

图中 `s1` 为 `string`，`sv1` 为 `string_view`，`s2` 是对 `s1` 的深拷贝（调用拷贝构造函数）所以 `s1` 被修改时，`s2` 仍保持旧的值不变。`sv1` 和 `sv2` 都是指向 `s1` 的弱引用，所以 `s1` 被改写时，`sv1` 和 `sv2` 看到的字符串也改写了

## 弱引用的安全问题

- 强引用和弱引用都可以用来访问对象。
- 每个存活的对象，强引用有且只有一个。
- 但弱引用可以同时存在多个，也可以没有。
- 强引用销毁时，所有弱引用都会失效。如果强引用销毁以后，仍存在其他指向该对象的弱引用，就会变成野指针，访问他会导致程序崩溃



建议创建 `string_view` 以后，不要改写原字符串

```cpp
string s1 = "hello";
string_view sv1 = s1;
s1[0] = 'm';
cout << sv1; // 可以
s1 = "helloworld";
cout << sv1; // 失效
```

被引用的 `string` 本体修改的时候，因为深拷贝调用了拷贝函数，`ptr` 和 `len` 改变了，原先生成的 `string_view` 还指向原先的指针，但是原先的指针已经变成野指针了，会失效

## 常见容器和相应的弱引用

| 强引用             | 弱引用            |
| --------------- | -------------- |
| `string`        | `string_view`  |
| `wstring`       | `wstring_view` |
| `vector<T>`     | `span<T>`      |
| `unique_ptr<T>` | `T*`           |
| `shared_ptr<T>` | `weak_ptr<T>`  |

## `string_view` 高效切片

`string` 的 `substr()` 返回一个全新的 `string` 对象，然后把需要保留的部分拷贝进去，也就是如果切下来的子字符串长度是 `n` 那么复杂度是O(n)，而 `string_view` 切片后的胖指针 `[ptr, len]`，就让新字符串和原字符串共享一片内存，实现了零拷贝零分配，不论子字符串多大，真正改变的只有两个变量，`substr` 函数复杂度为 O(1)

- `sv.remove_prefix(n)` 等价于 `sv = sv.substr(n)`
- `sv.remove_suffix(n)` 等价于 `sv = sv.substr(0, n)`
- 就地修改 `string_view` 对象本身，而不是修改他指向的字符串，原 `string` 还是不会变的
- `substr(pos, len)` 遇到 `pos > sv.size()` 的情况会抛出 `out_of_range` 异常。而 `remove_prefix/suffix` 就不会，如果他的 `n > sv.size()`，则属于未定义行为，可能崩溃。`remove_prefix/suffix` 更高效，`substr()` 更安全

其他很多接口都和 `string` 一样，有的都是只读的函数，没有就地修改的函数如 `append`

## 类型转换规则

隐式构造和显式构造的区别是是否允许编译器自动进行隐式类型转换。

| 转换                                     | 类型  | 时间复杂度  |
| -------------------------------------- | --- | ------ |
| `const char* -> string_view`           | 隐式  | $O(n)$ |
| `string -> string_view`                | 隐式  | $O(1)$ |
| `const char* -> string`                | 隐式  | $O(n)$ |
| `string_view -> string`                | 显式  | $O(n)$ |
| `string.c_str -> const char*`          |     | $O(1)$ |
| `string s = "str";`                    | 隐式  |        |
| `string s("str");`<br>`auto s("str");` | 显式  |        |
| `const char* cs = s.c_str()`           |     |        |

# 字符串编码

ASCII码建立了英文字母和标点符号到 `0x00-0x7F` 的映射。根据不同国家语言不同，产生了各种语言编码，例如中文使用 GBK。为了将不同国家的编码统一起来，产生了可以给世界上所有字符编码的 Unicode编码，映射到 `0x000000-0x10FFFF`，但是这超过了 `char` 类型的表示范围，因此产生了宽字符串（Linux上大小4byte）类型 `wchar_t` 和各种适配函数例如 `wcslen()`。

中文Windows系统默认编码是GBK。读写双方编码格式不同会导致乱码。

推出了完全兼容 ASCII 的 UTF-8 格式。
- UTF-32 是固定为 4 字节的编码（实际 Unicode 只有 3 字节）。
- UTF-16 是在 2 和 4 字节之间变长的编码。
- UTF-8 则是在 1、2、3、4 字节之间变长的编码。
- 其中 1 字节就是 ASCII 部分 0x00~0x7F，剩余的部分根据他们在 Unicode 中的顺序，依次变长。作为变长编码的代价，UTF-8 需要在二进制中浪费很多额外的空间来表示

https://zhuanlan.zhihu.com/p/427488961

- Windows 的 MSVC 在 Debug 模式下会默认把未初始化的栈内存填满 0xCC（x86 的 INT3 单步中断指令），未初始化的堆内存填满 0xCD。
- 而 `0xCCCC` 在 GBK 编码中就是“烫”，所以如果不小心打印了栈上未初始化的字符串数组，就会看到“烫烫烫”。
- 而 `0xCDCD` 在 GBK 编码中就是“屯”，所以如果不小心打印了堆上未初始化的字符串数组，就会看到“屯屯屯”。
- 现在普遍采用了 UTF-8 格式，虽然 Windows 还在用 UTF-16 和 GBK。

|                    |           |         |        |             |
| ------------------ | --------- | ------- | ------ | ----------- |
| 字符类型               | 字符串类型     | 字符串常量语法 | 大小（字节） | 编码格式        |
| `char`             | string    | "字符"    | 1      | 随系统默认编码格式而变 |
| `wchar_t`（Linux）   | wstring   | L"字符"   | 4      | UTF-32      |
| `wchar_t`（Windows） | wstring   | L"字符"   | 2      | UTF-16      |
| `char8_t`（C++20）   | u8string  | u8"字符"  | 1      | UTF-8       |
| `char16_t`（C++11）  | u16string | u"字符"   | 2      | UTF-16      |
| `char32_t`（C++11）  | u32string | U"字符"   | 4      | UTF-32      |

其中后面三个是不随系统而改变的（C++ 标准委员会定义），前面三个是随系统而改变的（Linux 和 Windows 自己定义）。

## C++14 自定义常量后缀的联动

| 表达式        | 类型             |
| ---------- | -------------- |
| `"字符"s`    | `string`       |
| `L"str"s`  | `wstring`      |
| `"str"sv`  | `string_view`  |
| `L"str"sv` | `wstring_view` |
| `u8"str"s` | `u8string`     |
