---
title: Flutter的编译模式
date: 2018-07-30 17:26:22
tags:
---
使用 Flutter构建过 App的人一定有一个困惑，就是 Flutter编译出的产物到底是什么玩意，有时候分为几个文件，有时候是一个动态库，真的叫人摸不着头脑。  
本文详细解释一下 Flutter的编译模式。  

<!-- more-->
# 编译模式的分类. 
编程语言要达到可运行的目的需要经过编译，一般地来说，编译模式分为两类：JIT 和 AOT。  
## JIT：
JIT全称 Just In Time(即时编译），典型的例子就是 v8，它可以即时编译并运行 JavaScript。所以你只需要输入源代码字符串，v8就可以帮你编译并运行代码。通常来说，支持 JIT的语言一般能够支持自省函数（eval），在运行时动态地执行代码。  
JIT模式的优势是显而易见的，可以动态下发和执行代码，而不用管用户的机器是什么架构，为应用的用户提供丰富而动态地内容。  
但 JIT的劣势也是显而易见的，大量字符串的代码很容易让 JIT编译器花费很多时间和内存进行编译，给用户带来的直接感受就是应用启动慢。  

## AOT：
AOT全称 Ahead Of Time（事前编译），典型的例子就是 C/C++，LLVM或 GCC通过编译并生成 C/C++的二进制代码，然后这些二进制通过用户安装并取得执行权限后才可以通过进程加载执行。  
AOT的优势也是显而易见的，事先编译好的二进制代码，加载和执行的速度都会非常快。（所以编程语言速度排行榜上前列都是 AOT编译类语言）这样的速度可以在密集计算场景下给用户带来非常好的体验，比如大型游戏的引擎渲染和逻辑执行。  

但是 AOT的劣势也是显而易见的，编译需要区分用户机器的架构，生成不同架构的二进制代码，除了架构，二进制代码本身也会让用户下载的安装包比较大。二进制代码一般需要取得执行权限才可以执行，所以无法在权限比较严格的系统中进行动态更新（如 iOS）。

# Dart的编译模式  
Flutter使用 Dart作为编程语言，自然其编译模式也脱离不了 Dart的干系。首先我们需要了解一下 Dart所支持的编译模式。    

* **Script**：最普通的 JIT模式，在 PC命令行调用 dart vm执行 dart源代码文件即是这种模式。
* **Script Snapshot**：JIT模式，和上一个不同的是，这里载入的是已经 token化的 dart源代码，提前执行了上一步的 lexer步骤。  
* **Application Snapshot**：JIT模式，这种模式来源于 dart vm直接载入源码后 dump出数据。dart vm通过这种数据启动会更快。不过值得一提的是这种模式是区分架构的，在 x64上生成的数据不可以给 arm使用。
* **AOT**：AOT模式，直接将 dart源码编译出 .S文件，然后通过汇编器生成对应架构的代码。  

总结一下刚才的列表，可以发现：  

 模式/比较项| 编译模式 | 区分架构| 打包大小| 动态化|
:---: | :---: | :---: | :---:| :---: 
Script | JIT | 否| 小| 是
Script Snapshot | JIT |否| 很小| 是|
Application Snapshot |  JIT |是| 比较大| 是（注意架构）|
AOT |  AOT|是|比较大|否|

# Flutter的编译模式  
Flutter 完全采用了 Dart，按道理来说编译模式一致才是，但是事实并不是这样。由于 Android和 iOS平台的生态差异，Flutter也衍生除了非常丰富的编译模式。  

+ **Script**：同 Dart Script模式一致，虽然 Flutter支持，但暂未看到使用，毕竟影响启动速度。  
+ **Script Snapshot**：同 Dart Script Snapshot一致，同样支持但未使用，Flutter有大量的视图渲染逻辑，纯 JIT模式影响执行速度。  
+ **Kernel Snapshot**：Dart的 bytecode 模式，与 Application Snapshot不同，bytecode模式是不区分架构的。 Kernel Snapshot在 Flutter项目内也叫 **Core Snapshot**。bytecode模式可以归类为 AOT编译。
+ **Core JIT**：Dart的一种二进制模式，将指令代码和 heap数据打包成文件，然后在 vm和 isolate启动时载入，直接标记内存可执行，可以说这是一种 AOT模式。Core JIT也被叫做 **AOTBlob**  
+ **AOT Assembly**: 即 Dart的 AOT模式。直接生成汇编源代码文件，由各平台自行汇编。  

可以看出来，Flutter将 Dart的编译模式复杂化了，多了不少概念，要一下叙述清楚是比较困难的，所以我们着重从 Flutter应用开发的各个阶段来解读。  

## 开发阶段的编译模式  
在开发阶段，我们需要 Flutter的 Hot Reload和 Hot Restart功能，方便 UI快速成型。同时，框架层也需要比较高的性能来进行视图渲染展现。因此开发模式下，Flutter使用了 Kernel Snapshot模式编译。   
在打包产物中，你将发现几样东西：  
+ **isolate\_snapshot\_data**：用于加速 isolate启动，业务无关代码，固定，仅和 flutter engine版本有关  
+ **platform.dill**：和 dart vm相关的 kernel代码，仅和 dart版本以及 engine编译版本有关。固定，业务无关代码。  
+ **vm\_snapshot_data**: 用于加速 dart vm启动的产物，业务无关代码，仅和 flutter engine版本有关  
+ **kernel\_blob.bin**：业务代码产物

项目/平台 | Android | iOS 
:---: | :---: | :---:  
代码环境 | debug | debug
编译模式 | Kernel Snapshot | Kernel Snapshot
打包工具 | dart vm (2.0) | dart vm (2.0)
Flutter命令 | flutter build bundle | flutter build bundle  
打包产物 | flutter_assets/* | flutter_assets/*|

## 生产阶段的编译模式  
在生产阶段，应用需要的是非常快的速度，所以 Android和 iOS target毫无意外地都选择了 AOT打包。不过由于平台特性不同，打包模式也是天壤之别。    


项目/平台 | Android | iOS|Android(--build-shared-library)    
:---: | :---: | :---: | :---:
代码环境 | release | release | release
编译模式 | Core JIT | AOT Assembly | AOT Assembly
打包工具 | gen_snapshot | gen_snapshot | gen_snapshot
Flutter命令 | flutter build aot | flutter build aot --ios | flutter build aot --build-shared-library
打包产物 | flutter_assets/* | App.framework | app.so  

首先我们很容易认识到 iOS平台上做法的原因：App Store审核条例不允许动态下发可执行二进制代码。  
所以在 iOS上，除了 JavaScript，其他语言运行时的实现都选择了 AOT。（比如 OpenJDK在 iOS实现就是 AOT）  

在 Android上，Flutter的做法有点意思：支持了两种不同的路子。  
**Core JIT**的打包产物有 4个：isolate\_snapshot\_data, vm\_snapshot\_data, isolate\_snapshot\_instr, vm\_snapshot\_instr.  我们不认识的产物只有 2个：isolate\_snapshot\_instr和 vm\_snapshot\_instr，其实它俩代表着 vm和 isolate启动后所承载的指令等数据，在载入后，直接将该块内存执行即可。  
Android的 **AOT Assembly**打包方式很容易让人想到需要支持多架构，无疑增大了代码包，并且该处代码需要从 JNI调用，远不如 Core JIT的 Java API方便。所以 Android上默认使用 Core JIT打包，而不是 AOT Assembly。  

# Flutter Engine对编译模式的支持  
在我的上篇文章：[Flutter原理简解](/2018/05/14/flutter-principle/)中提到，engine承载了 dart运行时，毫无疑问 engine需要和打包出来的代码对的上号才行。  
在 engine的编译模式中，Flutter是这样选择的：  

项目/平台 | iOS | Android 
:---: | :---: | :---:
Script | 不支持 | 不支持
ScriptSnapshot |理论支持 | 理论支持
Kernel Snapshot | 支持，runmode = dynamic | 支持，runmode = dynamic  
CoreJIT | 不支持 | 支持
AOT Assembly | 支持 | 支持 |

所以我们可以看到，Flutter的编译模式是完全根据 Engine的支持度来设计的。  

# 结论  
看到这里，我们完全可以得出一个结论：Flutter是一种高性能的、**可跨平台的**、**动态化**应用开发方案。  
在 iOS和 Android平台上，动态化完全可由 Kernel Snapshot打包实现，并且产物是一致通用的。不过目前通过打包工具进行了阉割，只能生成 debug产物。  
并且如果不需要动态化，同样可以打包出拥有更高执行性能的二进制库文件使用。这个特性目前就已经支持  

有了理论的支持，我们就可以着手做改造的事了。
