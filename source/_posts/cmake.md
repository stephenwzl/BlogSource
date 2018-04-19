---
title: 了解 CMake
date: 2017-12-25 20:00:43
tags:
---

![](http://cdn.stephenw.cc/wp-content/uploads/2017/12/20171219190328.png)

如果你需要经常看众多开源的项目，尤其是 C/C++编写的项目，你一定要学会 CMake的基本使用。

没有为什么

<!--more-->

# 安装
在 macOS上，你可以直接通过 homebrew安装 CMake，在其他 Linux发行版上，你也可以通过对应的包管理器安装。

# Hello World
创建一个目录，里面放一个非常简单的源代码文件，叫做 main.c

```c
#include <stdio.h>
 
int main(int argc, char *argv[]) 
{
  printf("hello world\n");
  return 0;
}
```

在同级目录中，添加一个 CmakeLists.txt文件，内容如下：

```cmake
cmake_minimum_required(VERSION 2.8)
 
project(cmakeTutorial)
 
add_executable(bin main.c)
```

然后，在你的代码目录里面：

```shell
mkdir .build
cd .build
cmake ..
make
./bin
```

一通猛如虎的操作后你会发现刚才写的代码跑出来了

## Hello World解读

C代码我就不多做介绍了，直接看 CMakeKLists.txt文件。

“cmake_minimum_required” 表示最小可使用的 CMake版本，如果你安装的 CMake小于 2.8是没法 “cmake ..”的。

“project(cmakeTutorial)” 表示给你当前的项目命名

“add_executable(bin main.c)” 表示当前项目目标是编译到可执行的 bin文件中去，源码实现是 main.c。

至少通过设定编译目标，我们实现了一次 C代码的 CMake编译。 但很明显只编译一个文件不是我们想要的，一个文件直接命令行调用 clang可能还更快点。

# 多源文件编译
为了演示一下多文件编译，我们再把 printf 给拆分成一个单独的函数试一下，创建 hello.h

```c
#ifndef _CMAKE_TUTORIAL_HELLO
#define _CMAKE_TUTORIAL_HELLO
 
extern void hello();
 
#endif
```

创建 hello.c

```c
#include <stdio.h>
 
void hello()
{
  printf("hello world\n");
}
```

修改 main.c

```c
#include "hello.h"
 
int main(int argc, char *argv[]) 
{
  hello();
  return 0;
}
```

然后，我们只需要把 CMakeLists.txt中的 add_executable里面再添加一个 hello.c就好了

```cmake
cmake_minimum_required(VERSION 2.8)
 
project(cmakeTutorial)
 
add_executable(bin main.c hello.c)
```

你再在 .build目录执行一遍猛如虎的操作试试，还是能编译运行成功的。

但是假如我有 N个源文件咋办，手动一个一个太麻烦了，可以这样改:

```cmake
cmake_minimum_required(VERSION 2.8)
 
project(cmakeTutorial)
 
aux_source_directory(./ DIRSRCS)
 
add_executable(bin ${DIRSRCS})
```

用了 CMakeLists.txt的内置函数 aux_source_directory，扫描一个目录的源代码文件，赋值给一个变量。（你始终记住源码文件是 .c/.cc/.cpp 这种实现文件)

但是把所有代码（包括头文件）放在一个目录下面很糟糕，看起来像是菜鸟写的。我们接下来实现一下多个目录。

# 多目录编译

假如我创建了一个 “hello”目录，并且把 hello.h , hello.c拖了进去，根据上面的经验，你可能会想 CMakeLists.txt这么写：

```cmake
cmake_minimum_required(VERSION 2.8)
 
project(cmakeTutorial)
 
aux_source_directory(./ DIRSRCS)
aux_source_directory(./hello HELLO_SRCS)
 
add_executable(bin ${DIRSRCS} ${HELLO_SRCS})
```

但是，转折来了。但是，你通常分目录的时候是为了代码分模块写的，那么子目录里面文件越多, 这看起来就越糟糕（强行糟糕） 通常情况是因为你的一个子模块希望能单独编译出去用。所以我们可以把子模块单独编译成库，给主项目用。

# 编译和链接静态库
在 hello目录里面添加一个 CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 2.8)
 
aux_source_directory(. SRCFILES)
 
add_library(hello ${SRCFILES})

```

然后我们把原来的那个 CMakeLists.txt修改一下：

```cmake
cmake_minimum_required(VERSION 2.8)
 
project(cmakeTutorial)
 
aux_source_directory(./ DIRSRCS)
add_subdirectory(hello)
 
add_executable(bin ${DIRSRCS})
 
target_link_libraries(bin hello)

```

这样就可以了。

add_subdirectory会在主 cmake时自动 cmake子目录，而子目录就正常写就可以了。

hello目录指定了自己的源文件编译成一个静态库叫 hello，然后主目录的 CMakeLists.txt里面指定了最后需要进行 target_link_libraries的链接。

而且，你在 .build目录看到的编译产物很明显也能见到 libhello.a

如果你想编译出一个动态库，只需要在 add_library的库名称后面加上 SHARED即可，如 “add_library(hello SHARED ${SRCFILES})”


# 头文件

在把子模块作为库链接后，你很明显会疑惑：我们平时用库都不会去 care库的头文件放在哪个目录，因为直接配在了 search path里面，那么 cmake怎么配呢。很简单，以上面为例, 你只需要添加一行

include_directories(hello)
这样你在 main.c 里面直接就可以 #include “hello.h”了，不用 care具体目录。

# 编译参数
这一部分并不想提多少，只是想告诉大家有这么个东西存在。比如你想设置 C++ Version为 11，那么你可以这样写：

```cmake
set(CMAKE_CXX_STANDARD 11)
```

其实概念上 CMAKE_CXX_STANDARD 是 cmake的一个内置变量， set就是用于设置变量值的函数。同样地你也可以设置 cxx flags

```cmake
set(CMAKE_CXX_FLAGS '-w')
```

更多的其他内置变量或者参数你可以参考 CMake官方文档，其实有需要的时候就可以直接 Google一下。

 

了解 CMake其实是用于阅读 C/C++项目的一种非常简单有效的手段，这为后面了解 LLVM项目打下一个基础。