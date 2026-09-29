## 内存开销

`std::optional` 的内存开销**不只是简单的“多加 1 字节”**，实际大小受**布尔标志位**和**内存对齐（Alignment）**的共同影响

- 不仅是 1 字节：为了记录是否有值，`std::optional` 内部会组合存储一个类似 `bool` 的标志位，而由于内存对齐的要求，这个标志位往往会撑大整个结构体。
- 对齐填充（Padding）：如果原本的类型 T 需要较大的对齐字节数（例如 4 字节或 8 字节），编译器为了让数据排布合理，会在布尔标志后补充大量的填充空白

在常见的 64 位系统上，`sizeof(std::optional<T>)` 的典型表现如下

`std::optional<bool>`：原本 1 字节，结果通常是 **2 字节**（1 字节值 + 1 字节标志，刚好对齐）。

`std::optional<int>`：原本 4 字节，结果通常是 **8 字节**（4 字节值 + 1 字节标志 + 3 字节填充对齐）。

`std::optional<double>`：原本 8 字节，结果通常是 **16 字节**（8 字节值 + 1 字节标志 + 7 字节填充对齐）。

## 用法

[C++ std::optional 用法與範例](https://shengyu7697.github.io/std-optional/#google_vignette)

[现代化的 API 设计指南 - ✝️小彭大典✝️](https://parallel101.github.io/cppguidebook/type_rich_api/)

空值语义

形如 `optional<T>` 的类型表示变量和函数返回值的两种可能的状态：

1. 为空
2. 有类型为 `T` 的值

对于函数，如果一个函数可能成功返回 `T` ，也可能失败，那就可以让他返回 `optional<T>`，用 `std::nullopt`来表示失败。

```cpp
std::optional<BookInfo> foo(ISBN isbn) {
    if (找到了) {
        return BookInfo(...);
    } else {
        return std::nullopt;
    }
}
```

### `std::nullopt`

类似 `nullptr` 但用途更加单一，更具说明性，显式地表达`optional<T>` 没有值的状态，防止和空指针混淆

### `has_value()`判断是否有值

```cpp
auto book = foo(isbn);
if (book.has_value()) {  // book.has_vlaue() 为 true，则表示有值
    BookInfo realBook = book.value();
    print("找到了:", realBook);
} else {
    print("找不到这本书");
}
```

`optional`类型可以在 if 条件中自动转换为 bool，判断是否有值，等价于 `.has_value()` ，即`if (book.has_value())` 可以自动转换为 `if(book)`

### `.value` 获取值

使用 `value()` 方法从 `std::optional` 中获取值， `*` 等价于 `.value()` 获取值

```cpp
BookInfo realBook = book.value();
BookInfo realBook = *book;
```

如果是空的会抛出`std::bad_optional_access`异常，用这个方法可以便捷地把“找不到书本”转换为异常抛出给上游调用者，而不用成堆的 if 判断和返回。

```cpp
BookInfo book = foo(isbn).value();
```

通过 `.value_or(默认值)` 指定“找不到书本”时的默认值：

```cpp
BookInfo defaultBook;
BookInfo book = foo(isbn).value_or(defaultBook);
```

### `reset()` 重设值

使用 `reset()` 方法將 `std::optional` 重設為空。

### `->` 访问成员

```cpp
if (auto book = foo(isbn)) {
    print("找到了:", book->name);
    book->readOnline();
}
```