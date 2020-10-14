---
title: Understanding CMake
date: 2017-12-25 20:00:43
tags:
---

![](/images/20171219190328.png)

If you need to look at many open source projects frequently, especially those written in C/C++, you must learn the basic use of CMake.

No why

<!--more-->

# Installation
On macOS, you can install CMake directly through homebrew, and on other Linux distributions, you can also install it through the corresponding package manager.

# Hello World
Create a directory and put a very simple source code file in it, called main.c

```c
#include <stdio.h>
 
int main(int argc, char *argv[])
{
  printf("hello world\n");
  return 0;
}
```

In the same level directory, add a CmakeLists.txt file with the following content:

```cmake
cmake_minimum_required(VERSION 2.8)
 
project(cmakeTutorial)
 
add_executable(bin main.c)
```

Then, in your code directory:

```shell
mkdir .build
cd .build
cmake ..
make
./bin
```

After a fierce operation, you will find that the code you just wrote ran out

## Hello World interpretation

I won't introduce the C code much, just look at the CMakeKLists.txt file.

"Cmake_minimum_required" means the minimum CMake version that can be used. If your installed CMake is less than 2.8, you cannot "cmake ..".

"Project(cmakeTutorial)" means to name your current project

"Add_executable(bin main.c)" means that the current project goal is to compile into an executable bin file, and the source code implementation is main.c.

At least by setting the compilation target, we achieved a CMake compilation of C code. But obviously only compiling one file is not what we want. It may be faster to call clang directly on the command line from a file.

# Multi-source file compilation
In order to demonstrate multi-file compilation, let's split printf into a single function to try again, create hello.h

```c
#ifndef _CMAKE_TUTORIAL_HELLO
#define _CMAKE_TUTORIAL_HELLO
 
extern void hello();
 
#endif
```

Create hello.c

```c
#include <stdio.h>
 
void hello()
{
  printf("hello world\n");
}
```

Modify main.c

```c
#include "hello.h"
 
int main(int argc, char *argv[])
{
  hello();
  return 0;
}
```

Then, we only need to add another hello.c to add_executable in CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 2.8)
 
project(cmakeTutorial)
 
add_executable(bin main.c hello.c)
```

If you try again in the .build directory, you can still compile and run successfully.

But what if I have N source files, it is too troublesome to manually one by one, you can change it like this:

```cmake
cmake_minimum_required(VERSION 2.8)
 
project(cmakeTutorial)
 
aux_source_directory(./DIRSRCS)
 
add_executable(bin ${DIRSRCS})
```

Use the built-in function aux_source_directory of CMakeLists.txt to scan the source code files of a directory and assign them to a variable. (You always remember that the source file is an implementation file like .c/.cc/.cpp)

But putting all the code (including header files) in one directory is bad, it looks like it was written by a rookie. Let's implement multiple directories next.

# Multi-directory compilation

If I created a "hello" directory and dragged hello.h and hello.c into it, based on the above experience, you might want to write CMakeLists.txt like this:

```cmake
cmake_minimum_required(VERSION 2.8)
 
project(cmakeTutorial)
 
aux_source_directory(./DIRSRCS)
aux_source_directory(./hello HELLO_SRCS)
 
add_executable(bin ${DIRSRCS} ${HELLO_SRCS})
```

However, the turning point has come. However, you usually write for the code to be divided into modules when you divide the directory, so the more files in the subdirectory, the worse it looks (forced bad). Usually it is because one of your submodules wants to be compiled separately for use. So we can compile the sub-module separately into a library for the main project.

# Compile and link static library
Add a CMakeLists.txt in the hello directory

```cmake
cmake_minimum_required(VERSION 2.8)
 
aux_source_directory(. SRCFILES)
 
add_library(hello ${SRCFILES})

```

Then we modify the original CMakeLists.txt:

```cmake
cmake_minimum_required(VERSION 2.8)
 
project(cmakeTutorial)
 
aux_source_directory(./DIRSRCS)
add_subdirectory(hello)
 
add_executable(bin ${DIRSRCS})
 
target_link_libraries(bin hello)

```

that's it.

add_subdirectory will automatically cmake subdirectories when the main cmake is done, and the subdirectories can be written normally.

The hello directory specifies its own source file to be compiled into a static library called hello, and then the CMakeLists.txt in the main directory specifies the link to target_link_libraries.

Moreover, the compiled product you see in the .build directory is obviously libhello.a

If you want to compile a dynamic library, just add SHARED after the library name of add_library, such as "add_library(hello SHARED ${SRCFILES})"


# head File

After linking the submodule as a library, you will obviously be puzzled: we usually don't use the library to find the directory where the header file of the care library is placed, because it is directly configured in the search path, then how to configure cmake. It’s very simple. Take the example above, you only need to add

include_directories(hello)
In this way, you can #include "hello.h" directly in main.c without having to care about the specific directory.

# Compile parameters
I don't want to mention much in this part, I just want to tell everyone that there is such a thing. For example, if you want to set the C++ Version to 11, then you can write:

```cmake
set(CMAKE_CXX_STANDARD 11)
```

In fact, conceptually CMAKE_CXX_STANDARD is a built-in variable of cmake, and set is a function used to set the value of the variable. Similarly, you can also set cxx flags

```cmake
set(CMAKE_CXX_FLAGS'-w')
```

For more other built-in variables or parameters, you can refer to the official CMake documentation. In fact, you can directly Google it when you need it.

 

Understanding CMake is actually a very simple and effective means for reading C/C++ projects, which lays a foundation for understanding LLVM projects later.