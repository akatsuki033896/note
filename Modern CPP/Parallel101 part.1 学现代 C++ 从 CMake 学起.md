---
title: "CMake part.1: 基本CMake操作"
publishDate: 2026-03-02
updatedDate: 2026-09-13
description: CMake 入门
tags:
  - CMake
  - Parallel101
---
## What is CMake

CMake **是一个跨平台的构建系统生成器(Build System Generator)**，它使用 `CMakeLists.txt` 文件来描述项目的构建过程。CMake 可以生成适用于不同平台和编译器的构建文件，如 Makefile、Visual Studio 工程文件等，简化了跨平台开发的复杂性。

<Aside type='caution'>构建系统生成器不等于构建工具！构建工具是更底层的具体程序，而构建系统生成器是管理这些工具的框架。构建工具例如 Ninja, 微软的MSBuild, Qt 的 qmake, 对应构建文件为 `build.ninja`, `.vcxproj`, `.pro`</Aside>

## Feature

- 只需要写一份 `CMakeLists.txt`，他就能够在调用时生成当前系统所支持的构建系统
- CMake 可以自动检测源文件和头文件之间的依赖关系，导出到 `Makefile` 里
- CMake 具有相对高级的语法，内置的函数能够处理 `configure`，`install` 等常见需求
- CMake 可以自动检测当前的编译器，需要添加哪些 `flag`：比如 OpenMP，只需要在 `CMakeLists.txt` 中指明 `target_link_libraries(a.out OpenMP::OpenMP_CXX)`即可

## Using CMake

### `add_executable`: 添加可执行文件

```cmake
add_executable(a.out main.cpp hello.cpp) # 添加可执行文件 a.out, 由 main.cpp 和 hello.cpp 编译而来
```

### 构建命令

```sh
CMake -B build # 生成构建目录 build
CMake --build build # 执行构建
```

## 库(library)

<Aside type='tip'>有时候我们会有多个可执行文件，他们之间用到的某些功能是相同的，我们想把这些共用的功能做成一个库，方便大家一起共享。库中的函数可以被可执行文件调用，也可以被其他库文件调用。</Aside>

库文件又分为**静态库文件**和**动态库文件**， 静态库相当于直接把代码插入到生成的可执行文件中，会导致体积变大，但是只需要一个文件即可运行。动态库只在生成的可执行文件中生成“插桩”函数，当可执行文件被加载时会读取指定目录中的 `.dll` 文件，加载到内存中空闲的位置，并且替换相应的“插桩”指向的地址为加载后的地址，这个过程称为重定向。这样以后函数被调用就会跳转到动态加载的地址去。

在一个项目中，通常有 `Include` 文件夹存放 `.h` 头文件，`lib` 文件夹存放库文件，静态库在 Windows 下的扩展名是 `.lib`，动态库的扩展名是 `.dll`，在 Linux 下的扩展名是 `.a` 和 `.so`，在 MacOS 下的扩展名是 `.a` 和 `.dylib`。

如果不是纯头文件的库的话，头文件和库文件必须要同时存在。

### `add_library`：添加库目标

CMake 除了 `add_executable` 可以生成可执行文件外，还可以通过 `add_library` 生成库文件

`add_library` 的语法与 `add_executable` 大致相同，除了他需要指定是动态库还是静态库:

- **生成静态库** libtest.a: `add_library(test STATIC source1.cpp source2.cpp)`
- **生成动态库** libtest.so: `add_library(test SHARED source1.cpp source2.cpp)`

动态库有很多坑，特别是 Windows 环境下，初学者自己创建库时，建议使用静态库。但是他人提供的库，大多是作为动态库的

创建库以后，要在某个可执行文件中使用该库，只需要:

- 为 myexec 链接刚刚制作的库 libtest.a: `target_link_libraries(myexec PUBLIC test)`
- 其中 PUBLIC 的含义稍后会说明(CMake 中有很多这样的大写修饰符)

```cmake
cmake_minimum_required(VERSION 3.9) # 指定最低版本
project (answer) # 指定项目名称
add_executable(answer main.cpp) # 添加可执行文件
add_library(libanswer STATIC answer.cpp) # 添加answer库目标 static指定为静态库
target_link_libraries(libanswer) # 为answer可执行目标链接上libanswer
```

然后执行 `CMake -B build` 生成构建目录，执行 `CMake --build build` 执行构建

```sh
CMake -B build
-- The C compiler identification is AppleClang 16.0.0.16000026
-- The CXX compiler identification is AppleClang 16.0.0.16000026
-- Detecting C compiler ABI info
-- Detecting C compiler ABI info - done
-- Check for working C compiler: /Library/Developer/CommandLineTools/usr/bin/cc - skipped
-- Detecting C compile features
-- Detecting C compile features - done
-- Detecting CXX compiler ABI info
-- Detecting CXX compiler ABI info - done
-- Check for working CXX compiler: /Library/Developer/CommandLineTools/usr/bin/c++ - skipped
-- Detecting CXX compile features
-- Detecting CXX compile features - done
-- Configuring done (0.9s)
-- Generating done (0.0s)
-- Build files have been written to: /Users/akatsuki/Documents/Project/build

CMake --build build

[ 25%] Building CXX object CMakeFiles/libanswer.dir/answer.cpp.o
[ 50%] Linking CXX static library liblibanswer.a
[ 50%] Built target libanswer
[ 75%] Linking CXX executable answer
[100%] Built target answer
```

改成动态库：

```cmake
add_library(libanswer SHARED answer.cpp) # 添加answer库目标
```

## 声明和头文件

<Aside type='tip'>函数要先声明再引用的原因: 1. 知道函数的参数和返回值类型才能支持重载，隐式类型转换等特性 2. 让编译器知道函数名是函数名，不是变量名和类名</Aside>

### 头文件：批量插入代码

原本目的在于批量插入代码，现在编译前替换的步骤变成了编译器的一部分，称为**预处理阶段**，`#define`定义宏也是预处理阶段。

#### `"header.h"` 和 `<header.h>` 的区别

- `<header.h>` 表示不要在当前目录下搜索，只在系统目录里搜索
- `"header.h"`表示优先搜索当前目录

`cstdio.h` 是提供了一系列函数声明的头文件，实际的实现在 libc.so 这个动态库里

#### 递归地使用头文件

函数声明需要用到某些类时，需要用到声明了该类的头文件，递归地`#include`即可

#### `#pragma once`：防止头文件被重复导入

如果多个头文件都引用了 `MyClass.h`,那么 `MyClass` 会被重复定义两遍，编译器报错`redifinition`，解决方案：在头文件前加上`#pragma once`,这样当预处理器第二次读到同一个文件时就会自动跳过。通常头文件都不想被重复导入，因此建议在每个头文件前加上这句话。

## CMake 中的子模块

### `target_include_directories`：添加头文件搜索目录

我们知道，传统情况下如果想要在一个项目使用一个编译好的库，通常需要在 include 文件夹包含头文件，还要在 lib 文件夹包含库文件。这是因为编译期要通过头文件知道函数和类的声明，运行期要通过静态库或动态库让链接器和知道函数和类的机器代码实现。

如果使用 VS，这些在 IDE 里点几下就可以配置好路径，那么在 CMake 中要怎么编写？

```cmake
cmake_minimum_required(VERSION 3.9) # 指定最低版本
project (answer) # 指定项目名称

add_subdirectory(answer) # 添加子文件夹 然后运行子目录的CMakeLists.txt
add_executable(answer_app main.cpp) # 添加可执行文件
target_link_libraries(answer_app libanswer)

# CMake -B build 生成构建目录
# CMake --build build 执行构建
```

```cmake
/answer/CMakeLists.txt
add_library(libanswer STATIC answer.cpp) # 添加answer库目标 static指定为静态库

target_include_directories(libanswer PUBLIC ${CMAKE_CURRENT_SOURCE_DIR}/include)
#CMAKE_CURRENT_SOURCE_DIR: 当前CMakeLists所在目录
```

子目录的 CMakeLists.txt 里路径名都是相对路径，很方便

这样还可以用`<.h>`来引用这个头文件，因为`target_include_directories`被视为与系统路径等价。

另一个 `b.out` 也需要用这个库时我们只需要定义头文件搜索路径，引用他的可执行文件 CMake 会自动添加这个路径：

这里用了`.`表示当前路径，因为子目录里的路径是相对路径，类似还有`..`表示上一层目录

此外，如果不希望让引用这个库的可执行文件自动添加这个路径，把 PUBLIC 改成 PRIVATE 即可。这就是他们的用途：决定一个属性要不要在被 link 的时候传播。

### 目标的一些其他选项

除了头文件搜索目录以外，还有这些选项，PUBLIC 和 PRIVATE 对他们同理：

```cmake
target_include_directories(myapp PUBLIC /usr/include/eigen3)
# 添加头文件搜索目录

target_link_libraries(myapp PUBLIC hellolib)
# 添加要链接的库

target_add_definitions(myapp PUBLIC MY_MACRO=1)
# 添加一个宏定义

target_add_definitions(myapp PUBLIC -DMY_MACRO=1)
# 与 MY_MACRO=1 等价

target_compile_options(myapp PUBLIC -fopenmp)
# 添加编译器命令行选项

target_sources(myapp PUBLIC hello.cpp other.cpp)
# 添加要编译的源文件
```

下列指令可以把选项加到所有接下来的目标去，但是不推荐使用：

```cmake
target_sources(/opt/cuda/include)
# 添加头文件搜索目录

link_directories(/opt/cuda)
# 添加库文件的搜索路径

add_definitions(MY_MACRO=1)
# 添加一个宏定义到每个目标

add_compile_options(-fopenmp)
# 添加编译器命令行选项
```

### 第三方库

#### 作为纯头文件引入

有时候我们不满足于 C++ 标准库的功能，难免会用到一些第三方库。最友好的一类库莫过于纯头文件库了，这里是一些好用的 header-only 库：

1. nothings/stb - 大名鼎鼎的 stb_image 系列，涵盖图像，声音，字体等，只需单头文件！
2. Neargye/magic_enum - 枚举类型的反射，如枚举转字符串等（实现方式很巧妙）
3. g-truc/glm - 模仿 GLSL 语法的数学矢量/矩阵库（附带一些常用函数，随机数生成等）
4. Tencent/rapidjson - 单纯的 JSON 库，甚至没依赖 STL(可定制性高，工程美学经典)
5. ericniebler/range-v3 - C++20 ranges 库就是受到他启发（完全是头文件组成）
6. fmtlib/fmt - 格式化库，提供 std::format 的替代品(需要 -DFMT_HEADER_ONLY)
7. gabime/spdlog - 能适配控制台，安卓等多后端的日志库（和 fmt 冲突！）

只需要把他们的 include 目录或头文件下载下来，然后`include_directories(spdlog/include)` 即可。缺点是函数直接实现在头文件里，没有提前编译，从而需要重复编译同样内容，编译时间长。

#### 作为子模块引入

通过`add_subdirectory`引入

```cmake
...
add_subdirectory(fmt)
add_executable(a.out main.cpp)
target_link_libraries(a.out PUBLIC fmt)
```
#### `find_package`: CMake 引用系统中预安装的第三方库

例如在 macos 上使用包管理器安装好的库可以通过`find_package`寻找

```cmake
find_package(fmt REQUIRED)
target_link_libraries(myexec PUBLIC fmt::fmt)
```

<Aside type='tip'>现代 CMake 认为一个包 (package) 可以提供多个库，又称组件 (components)，比如 TBB这个包，就包含了 tbb, tbbmalloc, tbbmalloc_proxy 这三个组件。</Aside>

因此为避免冲突，每个包都享有一个独立的名字空间，以 `::` 分割

你可以指定要用哪几个组件：

```cmake
find_package(TBB REQUIRED COMPONENTS tbb tbbmalloc REQUIRED)
target_link_libraries(myexec PUBLIC TBB::tbb TBB::tbbmalloc)
```
