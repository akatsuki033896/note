---
title: .clangd配置文件简介
source: https://vanilla-beauty.github.io/2024/10/08/%E5%AE%9E%E7%94%A8%E5%B7%A5%E5%85%B7%E7%AC%94%E8%AE%B0/clangd%E9%85%8D%E7%BD%AE%E6%96%87%E4%BB%B6%E7%AE%80%E4%BB%8B/
author:
  - "[[Vanilla-Beauty]]"
published: 2024-10-08
created: 2026-09-30
description:
tags:
  - clippings
---
## 1 背景介绍

`clangd` 是一个基于 LLVM 的 Clang 编译器前端提供的代码补全服务工具，主要用于 C 和 C++ 的代码补全、诊断、重构等功能。它支持多种编辑器和 IDE，并且通过 Language Server Protocol (LSP) 提供服务，使得它可以很容易地集成到不同的开发环境中。

个实际开发过程中不需要了解 `clangd` 的细节, 只需要通过 `.clangd` 配置文件指导 `clangd` 行为。

## 2 配置规则介绍

`.clangd` 是 Clangd 的配置文件，用于在项目中自定义 Clangd 的行为。这个文件通常位于项目的根目录下，Clangd 会在当前文件所在的所有父目录中搜索 `.clangd` 文件。

## 2.1 配置文件层次结构

`.clangd` 文件使用 YAML 格式，可以包含以下几个主要部分：

- `CompileFlags`: 编译标志部分
- `Diagnostics`: 诊断设置部分
- `InlayHints`: 代码提示设置部分
- `Completion`: 代码补全设置部分
- `SemanticTokens`: 语义标记设置部分

## 2.2 案例配置文件说明

更详细的配置工作参考 [https://clangd.llvm.org/config](https://clangd.llvm.org/config)

```yaml
# 最顶层包含几个主要部分：If, CompileFlags, Diagnostics, InlayHints, Completion, 和 SemanticTokens

If:                               # 设置配置文件的使用条件
  PathMatch: .*\.h                # 所有头文件都使用
  PathExclude: include/llvm-c/.*  # 排除该路径下的所有文件

CompileFlags:            # 编译标志部分
  Add:
    - "-Wall"           # 启用所有警告
    - "-Wextra"         # 启用额外警告
    - "-std=c++20"      # 使用C++20标准
    - "-Iinclude"       # 包含头文件, 相对路径
    - "-isystemF:/include" # 包含头文件, 绝对路径

  Remove: ["-Werror"]   # 移除将所有警告作为错误的选项
  Compiler: clang++     # 指定编译器

Index:
  Background: true            # 在后台索引项目代码
  BuildSystem: "CMake"        # 指定构建系统类型
  IndexFile: ".clangd-index"  # 索引文件路径
  ImplicitDynamic: true       # 动态索引所有头文件
  ResourceDir: "/path/to/resource"  # 资源目录路径

Diagnostics:             # 诊断设置部分
  ClangTidy:             # Clang-Tidy 配置
    Add: ["cppcoreguidelines-*", "modernize-*", "performance-*"] 
    # 添加Clang-Tidy检查项, 其含义参见https://clang.llvm.org/extra/clang-tidy/
    Remove: ["clang-analyzer-*"] # 移除Clang分析器检查项
  ClangTidyChecks: true  # 启用 Clang-Tidy 检查
  Options:               # Clang-Tidy 的选项设置
    UseChecks: ["*"]     # 使用所有检查项
    HeaderFilterRegex: ".*" # 正则表达式，用于过滤头文件
    WarningsAsErrors: "cppcoreguidelines-*" # 将特定警告视为错误

InlayHints:              # 代码提示设置部分
  Enabled: true          # 启用代码提示
  ParameterNames: true   # 显示参数名称提示
  DeducedTypes: true     # 显示推断类型提示
  Designators: true      # 显示设计器提示

Completion:              # 代码补全设置部分
  AllScopes: true        # 启用所有作用域的代码补全
  Brackets: "[]"         # 补全时使用的括号类型

SemanticTokens:          # 语义标记设置部分
  Enabled: true          # 启用语义标记
  DisabledKinds: ["Macro"] # 禁用宏语义标记
  DisabledModifiers: ["Static"] # 禁用静态修饰符语义标记
```

## 3 生成 compile\_commands.json 文件

`compile_commands.json` 是一个包含编译命令的 JSON 文件，用于描述项目中每个源文件的编译方式。它的主要作用是帮助 `Clangd` 和其他工具了解项目的编译设置，从而提供更准确的代码补全、跳转和诊断功能。具体作用如下：

## 3.1 Cmake生成 compile\_commands.json

### 3.1.1 方案一: CMakeLists.txt设置

```cmake
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)
```

### 3.1.2 方案二: 命令行参数设置

```bash
mkdir build
cd build
cmake -DCMAKE_EXPORT_COMPILE_COMMANDS=ON ..
```

## 3.2 Makefile生成 compile\_commands.json

Makefile本身不能直接生成 `compile_commands.json` 文件1。不过，可以使用一些工具来生成这个文件，例如 **Bear** 和 **CompileDB** 。

### 3.2.1 Bear

Bear是一个生成JSON编译数据库的工具，非常适合用于Makefile项目3。使用方法如下：

1. 安装Bear：  
	**bash** Copy
	```bash
	sudo apt install bear
	```
2. 运行Makefile并生成 `compile_commands.json` ：  
	**bash** Copy
	```bash
	bear make # 有的版本需要使用 bear -- make
	```

### 3.2.2 CompileDB

CompileDB是另一个生成 `compile_commands.json` 的工具，适用于GNU Make项目5。使用方法如下：

1. 安装CompileDB：  
	**bash** Copy
	```
	pip install compiledb
	```
2. 运行Makefile并生成 `compile_commands.json` ：  
	**bash** Copy
	```bash
	compiledb make
	```

## 4 实际案例

以一个使用 `C++20` + `vcpkg` 包管理的 `CMake` 项目为例，详细代码参考: [https://github.com/Vanilla-Beauty/ClangdLearn](https://github.com/Vanilla-Beauty/ClangdLearn)

> 关于vcpkg的使用可以参考我上一篇文章: [vcpkg包管理工具介绍](https://vanilla-beauty.github.io/2024/10/04/%E5%AE%9E%E7%94%A8%E5%B7%A5%E5%85%B7%E7%AC%94%E8%AE%B0/vcpkg%E5%8C%85%E7%AE%A1%E7%90%86%E5%B7%A5%E5%85%B7%E4%BB%8B%E7%BB%8D/)

**配置 `CMakeLixts.txt`**

```cmake
cmake_minimum_required(VERSION 3.10)
project(ClangdLearn)

find_package(fmt CONFIG REQUIRED)

set(CMAKE_CXX_STANDARD 20)

set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

add_executable(ClangdLearn main.cpp)
target_link_libraries(ClangdLearn PRIVATE fmt::fmt)
```

**`main.cpp`, 使用C++20的协程**

```cpp
#include "fmt/core.h" // 引入fmt库
#include <algorithm>
#include <coroutine>
#include <iostream>
#include <optional>
#include <ranges>
#include <vector>

// 使用C++20的range库
void print_even_numbers(const std::vector<int> &numbers) {
  auto even_numbers =
      numbers | std::views::filter( { return n % 2 == 0; });
  for (const auto &num : even_numbers) {
    fmt::print("{} ", num);
  }
  fmt::print("\n");
}

// 使用C++20的协程
struct Generator {
  struct promise_type {
    int current_value;
    auto yield_value(int value) -> std::suspend_always {
      current_value = value;
      return {};
    }
    auto initial_suspend() -> std::suspend_always { return {}; }
    auto final_suspend() noexcept -> std::suspend_always { return {}; }
    auto get_return_object() -> Generator { return Generator{this}; }
    void return_void() {}
    void unhandled_exception() { std::terminate(); }
  };

  using handle_type = std::coroutine_handle<promise_type>;
  handle_type coro;

  Generator(promise_type *p) : coro(handle_type::from_promise(*p)) {}
  ~Generator() {
    if (coro)
      coro.destroy();
  }

  auto move_next() -> bool {
    coro.resume();
    return !coro.done();
  }
  [[nodiscard]] auto current_value() const -> int {
    return coro.promise().current_value;
  }
};

auto generate_numbers(int max) -> Generator {
  for (int i = 1; i <= max; ++i) {
    co_yield i;
  }
}

auto main() -> int {
  std::vector<int> numbers = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};
  fmt::print("Even numbers: ");
  print_even_numbers(numbers);

  fmt::print("Generated numbers: ");
  auto gen = generate_numbers(10);
  while (gen.move_next()) {
    fmt::print("{} ", gen.current_value());
  }
  fmt::print("\n");

  return 0;
}
```

**配置 `.clangd` 文件**

```yaml
CompileFlags:
  Add:
    - "-Wall" # 启用所有警告
    - "-Wextra" # 启用额外警告
    - "-std=c++20" # 使用C++20标准
    - "-Iinclude" # 包含头文件, 相对路径, 这里没有使用, 只是一个示意
    - "-isystemF:/software/vcpkg/installed/x64-windows/include" # 包含头文件, 绝对路径, vcpkg的安装包头文件路径

Diagnostics:
  ClangTidy:
    Add: ["cppcoreguidelines-*", "modernize-*", "performance-*"]
    Remove: ["clang-analyzer-*"]

InlayHints:
  Enabled: true
  ParameterNames: true
  DeducedTypes: true
  Designators: true

Completion:
  AllScopes: true
```

**编译运行**

```powershell
cmake --preset vcpkg
cmake --build build # 可以成功运行
```

---