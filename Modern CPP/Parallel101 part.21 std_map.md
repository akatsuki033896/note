# STL：map

# Reference

[STL 精讲：std::map 和他的朋友们 - ✝️小彭大典✝️](https://142857.red/book/stl_map/#map)

[【C++ STL】全网最完整的map教程_哔哩哔哩_bilibili](https://www.bilibili.com/video/BV1qw4m1k7WD/?spm_id_from=333.1391.0.0&vd_source=9d28f0e4734f1bde4c84c3169e3a429d)

## 逻辑结构

![](https://142857.red/book/img/stl/logicmap.png)

一个键只能对应一个值，键不得重复，值可以重复

map 的具体实现 / 物理结构可以是红黑树、AVL 树、线性哈希表、链表哈希表、跳表，但逻辑结构都是键-值映射

## 物理结构

![](https://142857.red/book/img/stl/physmap.png)

map 和 set 一样，都是基于红黑树的二叉排序树，实现 $O(\log N)$ 复杂度的高效查找，在持续的插入和删除操作下，始终维持元素的有序。

### 红黑树

AVL树通过左子树和右子树的差不超过1维持平衡，避免了插入时退化成链表，map基于红黑树，没那么严格，左子树和右子树的差不超过2倍即可，因此牺牲了一部分查找性能，换取了更好的插入和删除性能。因此如果需要大量查找，考虑用查找平均复杂度低至 $O(1)$ 的哈希表 `unordered_map`

红黑树的性质：

- 不得出现相邻的红色节点（相邻指两个节点是父子关系），黑色可以相邻
- 从根节点到所有底层叶子的距离（以黑色节点数量计），必须相等

![](https://142857.red/book/img/stl/Red-black_tree_example.svg.png)

## 基本操作

```cpp
map<string, int> config; // 创建

config["timeout"] = 985; // 插入
config["delay"] = 211;

config.at("timeout"); // 查询
config.size(); // 元素数量
config.count("timeout"); // 返回容器中键和参数相等的元素个数，类型为 size_t
```

### 初始化

```cpp
map<string, int> config = {
    {"timeout", 985},
    {"delay", 211},
}; // C++11
config.at("timeout"));  // 985
```

作为函数参数

```cpp
void myfunc(map<string, int> const &config);  // 函数声明

myfunc(map<string, int>{               // 直接创建一个 map 传入
    {"timeout", 985},
    {"delay", 211},
});
```

从 `vector` 中导入，也可以自己写循环一个个插入

```cpp
vector<pair<string, int>> kvs = {
    {"timeout", 985},
    {"delay", 211},
};
map<string, int> config(kvs.begin(), kvs.end());
```

从c语言数组导入，`std::begin` 和 `std::end` 为 C++17 新增函数，专门用于照顾没法有成员函数 `.begin()` 的 C 语言数组

```cpp
pair<string, int> kvs[] = {  // C 语言原始数组
    {"timeout", 985},
    {"delay", 211},
};
map<string, int> config(kvs, kvs + 2);                    // C++98
map<string, int> config(std::begin(kvs), std::end(kvs));  // C++17
```

## 为什么不要用 `[]` 查找要用 `.at()`

map 具有 `[]`运算符重载，`[]` 里写要查询的键就可以返回对应值，也可以用 `=` 往里面赋值，但 `[]` 去**读取元素**是很不安全的，当查询的键值不存在时，`[]` 会默默创建并返回 0，而使用 `at()`会马上报错

```bash
terminate called after throwing an instance of 'std::out_of_range'
  what():  map::at
Aborted (core dumped)
```

`[]` 运算符实际上是在调用 `operator[]` 函数，这个成员函数后面没有 `const` 修饰，因此当 `map` 修饰为 `const` 时编译会不通过

```cpp
const map<string, int> config = {  // 此处如果是带 const & 修饰的函数参数也是同理
    {"timeout", 985},
    {"delay", 211},
};
print(config["timeout"]);          // 编译出错
```

为什么 `operator[]` 是非 `const` 修饰的？通常来说，一个成员函数不是 `const`，意味着他会**就地修改 this 对象**。`operator[]` 发现所查询的键值不存在时,会自动创建那个不存在的键值为 0，但当我们写入一个本不存在的键值的时候，恰恰需要 `[]` 的“自动创建”这一特性，这是 `at()` 所不具有的

```cpp
map<string, int> config = {
    {"timeout", 985},
    {"delay", 211},
};
print(config);
print(config["tmeout"]);  // 有副作用！"tmeout": 0
```

`[]`返回的必须是个具体的类型，由于 `[]` 不能报错，值的类型又千变万化，`map<K, V>` 的 `[]` 只能返回“V 类型默认构造函数创建的值”：对于 int 而言是 0，对于 string 而言是 `""`

- 读取元素时，统一用 `at()`
- 写入元素时，统一用 `[]`

## 合理利用 `[]`

`[]` 的效果：当所查询的键值不存在时，会调用默认构造函数创建一个元素

- 对于 int, float 等数值类型而言，默认值是 0。
- 对于指针（包括智能指针）而言，默认值是 nullptr。
- 对于 string 而言，默认值是空字符串 “”。
- 对于 vector 而言，默认值是空数组 {}。
- 对于自定义类而言，会调用你写的默认构造函数，如果没有，则每个成员都取默认值

### 出现次数统计

一开始不存在键值时，默认为0

```cpp
vector<string> input = {"hello", "world", "hello"};
map<string, int> counter;
for (auto const &key: input) {
    counter[key]++;
}
// {"hello": 2, "world": 1}
```

### 归类

```cpp
vector<string> input = {"happy", "world", "hello", "weak", "strong"};
map<char, vector<string>> categories;
for (auto const &str: input) {
    char key = str[0];
    categories[key].push_back(str);
}
// {'h': {"happy", "hello"}, 'w': {"world", "weak"}, 's': {"strong"}}
```

## 为什么不能在 `map` 里存 `const char*`

1. `const char*` 的 `==` 判断的是指针的相等，两个 `const char*` 只要地址不同，即使实际的字符串相同，也不会被视为同一个元素，导致 map 里会出现重复的键，以及按键查找可能找不到等，如果使用 `std::string`  那么内容和地址都是相等的
2. 保存的是弱引用，如果你把局部的 `char []` 或 `string.c_str()` 返回的 `const char *` 存入 `map`，等这些局部释放了，`map` 中的 `const char *` 就是一个空悬指针了，会造成 segfault

## `map` 构造反向查找表

例如查找特定元素在 vector 中的位置，每查一次就需要遍历一次也就是 $O(N)$

构建查找表后仅构建时遍历，查一次是$O(logN)$

```cpp
vector<string> arr = {"hello", "world", "nice", "day", "fucker"};
map<string, size_t> arrinv;
for (size_t i = 0; i < arr.size(); i++) {                // O(N) 一次性受苦
    arrinv[arr[i]] = i;
}
print("fucker在数组中的下标是：", arrinv.at("fucker"));  // O(log N) 高效
print("nice在数组中的下标是：", arrinv.at("nice"));      // O(log N) 高效
```

### **`map` 构建另一个 `map` 的反向查找表**

```cpp
map<string, string> tab = {
    {"hello", "world"},
    {"fuck", "rust"},
};
map<string, string> tabinv;
for (auto const &[k, v]: tab) {
    tabinv[v] = k;
}
print(tabinv);
```

要求 tab 中不能存在重复的值，键和值必须是一一对应关系，才能用这种方式构建双向查找表

## 遍历

```cpp
for (auto it = m.begin(); it != m.end(); ++it) {
    print("Key:", it->first);
    print("Value:", it->second);
} // c++11

for (auto it = m.begin(); it != m.end(); ++it) {
    auto [k, v] = *it;
    print("Key:", k);
    print("Value:", v);
} // c++17结构化绑定

for (auto [k, v]: m) {
    print("Key:", k);
    print("Value:", v);
} // 结构化绑定+范围for循环
```

### 在遍历时修改值

注意 `*it` 解引用得到的是 `pair<const K, V>` 类型的键值对，需要 `(*it).second` 才能获取单独的值 `v` ,可以简写成 `it->second`

```cpp
map<string, int> m = {
    {"fuck", 985},
    {"rust", 211},
};
for (auto it = m.begin(); it != m.end(); ++it) {
    it->second = it->second + 1;
} // {"fuck": 985, "rust": 212}
```

注意范围for循环和结构化绑定都只是语法糖

```cpp
for (auto [k, v]: m) {
    v = v + 1;
} // {"fuck": 985, "rust": 211}

for (auto &[k, v]: m) {  // 捕获一个引用，写入这个引用会立即作用在原值上
    v = v + 1;
} // {"fuck": 985, "rust": 212}
```

实际上等于

```cpp
for (auto it = m.begin(); it != m.end(); ++it) {
    auto &tmp = *it;
    auto &k = tmp.first;
    auto &v = tmp.second;
    v = v + 1;
}
```

这样保存下来的 `v` 是个引用，是对原值的引用，不仅避免拷贝的开销节省了性能，而且对 v 的修改会实时反映到原 map 中去，因此要修改值的时候需要使用 `auto&` ，即使不需要修改 map 中的值时，也建议用 `auto const &` 避免拷贝的开销

## `find()` 优化查找

find 的高效在于可以把两次查询合并成一次，count和at都是基于find实现的

```cpp
if (m.count("key")) {    // 第一次查询，只包含"是否找到"的信息
    print(m.at("key"));  // 第二次查询，只包含"找到了什么"的信息
} // 2logN

auto it = m.find("key"); // 一次性查询
if (it != m.end()) {     // 查询的结果，既包含"是否找到"的信息
    print(it->second);   // 也包含"找到了什么"的信息
} // logN
```

## `erase()` 删除元素

指定键值 key，erase 会删除这个键值对应的元素，返回一个整数类型为 size_t，表示删除了多少个元素（只能是 0 或 1）

当已知指向要删除元素的迭代器时（例如先通过 find 找到），直接指定那个迭代器比指定键参数更高效

```cpp
size_t erase(K const &key);  // 指定键版 O(logN)
// 实际上需要先调用 find(key) 找到元素位置，然后才能删除，而且还有找不到的可能性
iterator erase(iterator it);   // 已知位置版 O(1)+
```

```cpp
map<string, string> msg = {
    {"hello", "world"},
    {"fuck", "rust"},
};
msg.erase("fuck");
```

### 一边遍历一边删除

先获取到迭代的下一个位置，再把当前位置删掉。如果直接删掉的话当前地址已经没有任何东西，再往下迭代的地址是一片什么都没有的地址，会导致崩溃，也就是迭代器失效。

```cpp
map<string, string> msg = {
    {"hello", "world"},
    {"fucker", "rust"},
    {"fucking", "java"},
    {"good", "job"},
};
for (auto it = m.begin(); it != m.end(); ) {  // 没有 ++it
    auto const &[k, v] = *it;
    if (k.starts_with("fuck")) {
        it = msg.erase(it);
    } else {
        ++it;
    }
} // 可以

for (auto it = m.begin(); it != m.end(); ++it) {
    auto const &[k, v] = *it;
    if (k.starts_with("fuck")) {
        msg.erase(it);
        // 或者 msg.erase(k);
    }
}// 不行
```

实在不行就把键全部先存在vector里遍历

## 插入

pair 是一个 STL 中常见的模板类型，`pair<K, V>` 有两个成员变量：

- first：K 类型，表示要插入元素的键
- second：V 类型，表示要插入元素的值

```cpp
// 原型
pair<iterator, bool> insert(pair<const K, V> const &kv);
pair<iterator, bool> insert(pair<const K, V> &&kv);

// 最简单的插入 cpp11初始化列表
map<K, V> m;
m.insert({"key", "val"});
```

当键 K 不存在时，insert 和 [] 都会创建键值对。当键 K 已经存在时，insert 不会覆盖，默默离开，而 [] 会覆盖旧的值。

返回值是一个 pair 类型，其具有两个成员：

- first：iterator 类型，是个迭代器
- second：bool 类型，表示插入成功与否，如果发生键冲突则为 false

其中 first 这个迭代器指向的是：

- 如果插入成功（second 为 true），指向刚刚成功插入的元素位置
- 如果插入失败（second 为 false），说明已经有相同的键 K 存在，发生了键冲突，指向已经存在的那个元素