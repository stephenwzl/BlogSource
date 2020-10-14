---
title: Flutter's compilation mode
date: 2018-07-30 17:26:22
tags:
---
People who have used Flutter to build apps must be confused about what the product compiled by Flutter is. Sometimes it is divided into several files, sometimes it is a dynamic library, which is really confusing.
This article explains in detail the compilation mode of Flutter.

<!-- more-->
# Classification of compilation modes.
A programming language needs to be compiled to be runnable. Generally speaking, there are two types of compilation modes: JIT and AOT.
## JIT:
JIT stands for Just In Time (Just In Time), a typical example is v8, which can compile and run JavaScript on the fly. So you only need to enter the source code string, and v8 can help you compile and run the code. Generally speaking, languages ​​that support JIT can support introspection function (eval), which dynamically executes code at runtime.
The advantage of the JIT model is obvious. It can dynamically issue and execute code regardless of the architecture of the user's machine, providing rich and dynamic content for application users.
But the disadvantages of JIT are also obvious. A large number of strings of code can easily make the JIT compiler spend a lot of time and memory to compile, and the direct feeling that the user has is that the application starts slowly.

## AOT:
The full name of AOT is Ahead Of Time (pre-compilation). A typical example is C/C++, LLVM or GCC compiles and generates C/C++ binary code, and then these binaries can be loaded and executed by the process after they are installed and executed by the user.
The advantages of AOT are also obvious. The pre-compiled binary code will be loaded and executed very fast. (So ​​the top of the programming language speed rankings are AOT compiled languages) Such speed can bring users a very good experience in intensive computing scenarios, such as engine rendering and logic execution of large games.

But the disadvantages of AOT are also obvious. Compilation needs to distinguish the architecture of the user's machine and generate binary codes of different architectures. In addition to the architecture, the binary code itself will also allow users to download larger installation packages. Binary code generally requires execution permissions to be executed, so it cannot be dynamically updated in systems with strict permissions (such as iOS).

# Dart's compilation mode
Flutter uses Dart as a programming language, and naturally its compilation mode cannot be separated from Dart. First of all, we need to understand the compilation mode supported by Dart.

* **Script**: The most common JIT mode. Calling dart vm on the PC command line to execute dart source code files is this mode.
* **Script Snapshot**: JIT mode. The difference from the previous one is that the tokenized dart source code is loaded here, and the lexer step of the previous step is executed in advance.
* **Application Snapshot**: JIT mode, which comes from dart vm directly loading the source code and dumping the data. Dart vm will start faster with this kind of data. However, it is worth mentioning that this mode distinguishes architecture, and the data generated on x64 cannot be used by arm.
* **AOT**: AOT mode, directly compile the dart source code into a .S file, and then generate the code of the corresponding architecture through the assembler.

Summarizing the list just now, you can find:

 Mode/Comparison | Compilation Mode | Distinguish Architecture | Pack Size | Dynamic|
:---: | :---: | :---: | :---:| :---:
Script | JIT | No | Small | Yes
Script Snapshot | JIT |No| Very Small| Yes|
Application Snapshot | JIT |Yes | Larger | Yes (note the architecture)|
AOT | AOT|Yes|Bigger|No|

# Flutter's compilation mode
Flutter completely adopts Dart. It is reasonable to say that the compilation mode is consistent, but this is not the case. Due to the ecological differences between the Android and iOS platforms, Flutter has also derived a very rich compilation mode.

+ **Script**: Same as Dart Script mode. Although Flutter supports it, it hasn't been used yet. After all, it affects startup speed.
+ **Script Snapshot**: Same as Dart Script Snapshot, and it is also supported but not used. Flutter has a lot of view rendering logic, and pure JIT mode affects the execution speed.
+ **Kernel Snapshot**: Dart's bytecode mode, unlike Application Snapshot, bytecode mode does not distinguish between architectures. Kernel Snapshot is also called **Core Snapshot** in the Flutter project. The bytecode mode can be classified as AOT compilation.
+ **Core JIT**: A binary mode of Dart, which packs instruction code and heap data into files, and then loads them when vm and isolate are started, and directly marks the memory as executable. It can be said that this is an AOT mode. Core JIT is also called **AOTBlob**
+ **AOT Assembly**: Dart's AOT mode. Generate assembly source code files directly and compile them by each platform.

It can be seen that Flutter complicates Dart's compilation mode and adds a lot of concepts. It is more difficult to describe it clearly, so we focus on interpreting from the various stages of Flutter application development.

## Compilation mode during development
In the development phase, we need Flutter's Hot Reload and Hot Restart functions to facilitate rapid UI shaping. At the same time, the framework layer also needs relatively high performance for view rendering. Therefore, in development mode, Flutter uses Kernel Snapshot mode to compile.
In the packaged product, you will find several things:
+ **isolate\_snapshot\_data**: used to accelerate isolate startup, business-independent code, fixed, only related to flutter engine version
+ **platform.dill**: The kernel code related to dart vm is only related to the dart version and engine compiled version. Fixed, business-independent code.
+ **vm\_snapshot_data**: A product used to accelerate the startup of dart vm, business-independent code, only related to the flutter engine version
+ **kernel\_blob.bin**: business code product

Project/Platform | Android | iOS
:---: | :---: | :---:
Code environment | debug | debug
Compilation mode | Kernel Snapshot | Kernel Snapshot
Packaging tool | dart vm (2.0) | dart vm (2.0)
Flutter command | flutter build bundle | flutter build bundle
Package product | flutter_assets/* | flutter_assets/*|

## Compilation mode in production phase
In the production phase, the application needs very fast speed, so the Android and iOS targets are unsurprisingly selected for AOT packaging. However, due to different platform characteristics, the packaging model is also worlds apart.


Project/Platform | Android | iOS|Android(--build-shared-library)
:---: | :---: | :---: | :---:
Code environment | release | release | release
Compilation Mode | Core JIT | AOT Assembly | AOT Assembly
Packaging tool | gen_snapshot | gen_snapshot | gen_snapshot
Flutter command | flutter build aot | flutter build aot --ios | flutter build aot --build-shared-library
Package product | flutter_assets/* | App.framework | app.so

First of all, it is easy for us to realize the reason for the practice on the iOS platform: App Store review regulations do not allow dynamic distribution of executable binary code.
So on iOS, in addition to JavaScript, the runtime implementation of other languages ​​has chosen AOT. (For example, the implementation of OpenJDK in iOS is AOT)

On Android, Flutter's approach is interesting: it supports two different ways.
There are 4 packaged products of **Core JIT**: isolate\_snapshot\_data, vm\_snapshot\_data, isolate\_snapshot\_instr, vm\_snapshot\_instr. There are only 2 products we don’t know: isolate\_snapshot\ _instr and vm\_snapshot\_instr, in fact, they both represent the instructions and other data carried after vm and isolate are started. After loading, you can directly execute the block of memory.
Android's **AOT Assembly** packaging method is easy to think of the need to support multiple architectures, which undoubtedly increases the code package, and the code needs to be called from JNI, which is far less convenient than Core JIT's Java API. Therefore, Core JIT packaging is used by default on Android instead of AOT Assembly.

# Flutter Engine support for compilation mode
In my last article: [A brief explanation of the principle of Flutter](/2018/05/14/flutter-principle/) mentioned that the engine carries the dart runtime, and there is no doubt that the engine needs to be compatible with the packaged code Number is fine.
In the engine's compilation mode, Flutter is selected as follows:

Project/Platform | iOS | Android
:---: | :---: | :---:
Script | Not Supported | Not Supported
ScriptSnapshot | Theoretical Support | Theoretical Support
Kernel Snapshot | Support, runmode = dynamic | Support, runmode = dynamic
CoreJIT | Not Supported | Supported
AOT Assembly | Support | Support |

So we can see that Flutter's compilation mode is completely designed according to the engine's support.

# In conclusion  
Seeing this, we can draw a conclusion: Flutter is a high-performance, **cross-platform**, **dynamic** application development program.
On iOS and Android platforms, dynamization can be completely packaged with Kernel Snapshot, and the product is consistent and universal. However, currently it has been castrated by the packaging tool and can only generate debug products.
And if dynamization is not required, binary library files with higher execution performance can also be packaged for use. This feature is currently supported

With the support of theory, we can start to transform.