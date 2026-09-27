# 排序

`set` 自动给其中的元素从小到大排序，对于字符串类型 `std::string` 就按照字典序来排。注意，如果用 `set<char*>` 做字符串集合会按字符串指针的地址去判断相等，而不是所指向字符串的内容。

## 自定义排序函数

`set` 有两个模板参数： `set<T, CompT>`
- `T` 为容器内元素类型
- `CompT` 为比较函子，调用它来决定怎么排序，默认为 `<`

```cpp
struct MyComp {
	bool operator()(string const &a, string const &b) const {
		return a < b;
	}
}; // 默认
int main() {
	set<string, MyComp> s = {"arch", "any", "zero"};
}
```

# 迭代器

- `set` 的 `begin()` 和 `end()` 返回的迭代器对象重载了 `*` 来访问指向的地址
- `set` 重载了 `++` 遍历红黑树
- `set` 没有重载 `+=` 和 `+`，因为 `set` 在内存中不是连续存储，不能随机访问，只能顺序访问，例如需要让迭代器向前移动 3 个位置，要调用 3 次 `++`，这个模拟的复杂度是 $O(n)$，并且会改变迭代器本身

## `std::end`

原型为 `iterator end() const`
指向 `set` 中最后一个元素**后面一个位置的地址**，该地址不可能有任何元素，仅仅用来做标记

## `std::next`：等价于 `+`

语法为 `std::next(迭代器, 次数)`，返回自增后迭代器
自动判断迭代器是否支持 `+`，如果不支持，会改为比较低效的调用 $n$ 次 `++`

```cpp
set<int> b = {1, 2, 3, 4};
set<int>::iterator b_it = b.begin();
b_it = std::next(b_it, 3); // 调用三次 b_it++
```

## `std::advance`：等价于 `+=`

语法为 `std::advance(迭代器, 次数)`，**就地自增**作为引用传入的迭代器，没有返回值
判断是否支持 `+=` 来决定要采用哪一种实现

```cpp
set<int> b = {1, 2, 3, 4};
set<int>::iterator b_it = b.begin();
std::advance(b_it, 3); // 调用三次 b_it++
```

如果是双向迭代器，`std::next` 和 `std::advance` 就支持负数，等价于在 `std::prev` 使用正数

## `std::distance`：等价于 `-`

语法为 `std::distance(it1, it2)`，相当于 `it2 - it1`，求出两个迭代器之间的距离差值

# 插入

## 成员函数 `insert`

原型为 `pair<iterator, bool> insert(T val)`，调用 `insert()` 插入元素，`set` 会自动排序，自动去重。

- 返回值 `iterator`：插入成功时返回指向新插入的元素，插入失败意味着 `set` 里已经有相同元素，则指向存在的这个相同元素
- 返回值 `bool`：返回插入是否成功

## `std::pair` 结构化绑定拆解(C++17)

结构化绑定可以拆解 POD 结构体的成员，包括 `pair` 也可以拆

```cpp
set<int> b = {1, 2, 3, 4};
auto [it, ok] = b.insert(3);
// 等价于
// auto tmp = b.insert(3);
// auto it = tmp.first;
// auto ok = tmp.second;
```

# 查找

## 成员函数 `find`

原型为 `iterator find(int const &val) const`，调用 `find(val)` 查找容器中值为 `val` 的元素，找到则返回指向找到元素的迭代器。如果找不到，则返回 `end()` 迭代器 。

`set.find(x) != set.end()` 是**固定写法**，来判断集合 `set` 中是否存在元素 `x`

```cpp
set<int> b = {1, 2, 3, 4};
if (b.find(2) != b.end()) {
	std::cout << "exist" << '\n';
}
```

## 成员函数 `count`

原型为 `size_t count(int const &val) const`，返回表示集合中相等元素的个数，虽然 `set` 有去重功能，但是标准库要考虑到接口的泛用性， `set` 的 `count` 只返回 `0` 或 `1`，同样用来判断元素是否存在

```cpp
if (set.count(2) != 0) {
	std::cout << "exist" << '\n';
}
```

## 成员函数 `lower_bound` `upper_bound`

原型为 `iterator lower_bound(int const &val) const` 和 `iterator upper_bound(int const &val) const`

`lower_bound(x)` 找第一个大于等于 `x` 的元素，`upper_bound(x)` 找第一个大于 `x` 的元素，找不到时都会返回 `end()`

# 删除

## 成员函数 `erase`

原型为 `size_t erase(int const &val)`: `erase(x)` 可以删除集合中值为 `x` 的元素，返回一个整数，表示被他删除元素的个数，`0` 就说明集合中没有该元素，删除失败，`1` 就说明集合中存在该元素，删除成功

原型为 `iterator erase(iterator pos)`: `erase(it)` 删除集合位于 `it` 处的元素，常见用法有：
- `set.erase(set.find(x))` 会删除集合中值为
`x` 的元素
- `set.erase(set.begin())` 会删除集合中最小的元素
- `set.erase(std::prev(set.end()))` 会删除集合中最大的元素(自动排序）

原型为 `iterator erase(iterator first, iterator last)`：输入两个迭代器，`erase(beg, end)` 可以删除集合中从 `beg` 到 `end` 之间的元素，包含 `beg`，不包含 `end` 即**前开后闭**区间（标准库都是这样），注意 `beg` 必须在 `end` 前面

## 删除指定区间元素

使用 `set.erase(set.find(x), set.find(y)` 的前提是这两个元素都存在，否则若`x` 不存在，`find(x)` 返回 `end()`，违背 `beg` 必须在 `end` 前面的原则，标准库会崩溃。

因此，使用 `a.erase(a.lower_bound(x), a.upper_bound(y))` 就不会返回 `end()` 了，会删除**闭区间** `[x, y]` 的元素

# 修改

无法修改，只能删掉再插入

# 成员函数 `clear`：清空元素

类似 `vector` 的 `clear`

#  成员函数 `size`：查询元素个数

原型为 `size_t size() const noexcept`查询其中元素个数

# 遍历

和 `vector` 一样

```cpp
for (auto it = b.begin(); it != b.end(); it++) {
	int value = *it;
	std::cout << value << '\n';
}
```

## 指针到迭代器

## 基于范围的 `for` 循环 (C++17)

语法糖，语法为 `for (类型 变量名 : 可迭代对象)`

~~越来越像 python 了~~

```cpp
for (int value : b) {
	std::cout << value << '\n';
}
```

有时写完整版会有更大的自由度，可以把迭代器换成别的

# 和其他容器的转换

把 `set` 整个拷贝到 `vector` 里

```cpp
set<int> b = {1, 2, 3, 4};
vector<int> arr(b.begin(), b.end());
```

`vector` 的构造函数允许接受 2 个前向迭代器，`set` 是双向迭代器满足要求，可以把 `set` 一个区间拷贝到 `vector` 里，就可以过滤出所有该区间的元素

```cpp
set<int> b = {1, 2, 3, 4};
vector<int> arr(b.lower_bound(2), b.upperbound(3)); // {2, 3}
```

也可以反过来

## 排序

把 `vector` 转成 `set` 会让元素自动排序和去重，把 vector 转成 set 再转回
vector，这样就实现对 `vector` 去重

```cpp
vector<int> arr = {9, 8, 5, 2, 1, 1};
set<int> b(arr.begin(), arr.end());
arr.assign(b.begin(), b.end());
```

# `std::multiset`：不去重的 `set`

允许重复的元素，但仍保留自动排序，能高效地查询的特点

## 查找等值区间

`multiset` 里相等的元素都是紧挨着排列的，可以用 `upper_bound` 和 `lower_bound` 函数获取所有相等值的区间 `[lower_bound, upper_bound)`

### `equal_range`：求出两个边界

原型为 `pair<iterator, iterator> equal_range(int const &val) const`

对于 `lower_bound` 和 `upper_bound` 的参数相同的情况，可以用 `equal_range`
一次性求出两个边界，获得等值区间，更高效，当指定的值找不到时，`equal_range` 返回两个 `end()` 迭代器，代表空区间

```cpp
auto r = b.equal_range(2)
```

## 删除等值区间

`erase` 只有一个参数的版本，会把所有等于 `2` 的元素删除

## 求等值元素个数

`count(x)` 返回 `multiset` 中等于 `x` 的元素个数

`find(x)` 会返回第一个等于 `x` 的元素的迭代器。找不到也是返回 `end()`

# `std::unordered_set` (C++11)

`unordered_set` 不会排序，里面的元素都是完全随机的顺序，和插入的顺序也不一样。`set` 基于红黑树实现，相当于二分查找树，`unordered_set` 基于散列哈希表实现，正是哈希函数导致了随机的顺序。
