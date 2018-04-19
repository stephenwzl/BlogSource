---
title: LLVM的 Modules
date: 2017-08-23 10:26:44
tags:
---

Modules 其实是 随Xcode5一起发布的 build system，非常早也非常老了，但仍然没有受到足够的重视。

直到 Swift 的到来。

本文带你一起探究：

1. Modules 是什么
2. 使用和生产 Modules
3. Modules 和 Swift
4. 在 LLVM9.0前让 Swift 使用 objc静态库

<!--more-->

# Modules 是什么
网路博客上其实关于 modules 的解释有点少（或者说不够详细），我们先了解一下 modules 的产生背景：

大多数的软件（App）都用了很多很多的库来构建，所以我们在写代码的时候肯定就会引入这些库的头文件来使用它们提供的便捷的方法，比如这样：

```c
#include <SomeLib.h>
```

（假定你已经知道了 #include 和 #import 的异同，我就不多啰嗦这部分了）

这个头文件方法的具体实现实际是 SomeLib 这个库处理的，所以我们需要向链接器提供一个参数 “-lSomeLib”，链接器就能知道具体的实现在哪儿。但这其中有一点坑，通常软件开发者并不会注意到，Modules就是为了来解决这些问题的，请继续往下阅读。

问题1：我们都知道 “#include”关键字引入的头文件实际上就是把目标头文件的内容简单粗暴地拷贝进来，假设你的头文件包含进了 M 个其他库的头文件，而这 M 个头文件里面平均又引入了 N 个头文件，那么造成的结果就是编译器在预处理阶段要做 M*N 次头文件的处理工作。这可能会在你的头文件里面引入数量非常庞大的代码。假如换成 C++，那情况就更糟了，因为它还包含了一些模板代码，数量比 C 还要多好几倍。

问题2：你在使用 “#include” 这些关键字时包含的头文件文本会比较全量。所以你在两个不同的库里面假如定义了相同名称的宏定义，那这结果就非常微妙了，很可能最终运行时结果就出乎意料了，这使得预处理变得非常“脆弱”。

问题3：假如为了解决问题2，开发者约定不同库要有不同的 prefix，然后就会造成一个定义名称非常地冗长，或者又有一些开发者会用下划线作为开头等等这些命名，使得整个库的命名变得非常丑陋，这对非“C 语言程序员”来说是一个障碍。

问题4：什么时候用什么工具、库来开发软件，仅仅从头文件上面看其实你并不能看得懂，因为它并不是“语义化”的，比如哪些命名空间属于特定的库，这些命名空间又该以如何的顺序包含，或者你又只想引入这个库的一部分定义，只通过 “#include”之类的预处理真的是太难搞清楚了。

以上四个问题其实就是简单地翻译了一下 Clang 文档里面关于 Modules 要解决的问题。

所以说 **Modules** 是一种语义化的引入定义的方法。它是用来替代普通的 “#include”系列的指令。

```c
@import WebKit.WebKitLegacy; //in Objective-C
import WebKit.WebKitLegacy   //in Swift
```

可以看到 Objective-C 和 Swift 都非常好地支持了 Modules import，你可以非常清晰地引入 API 声明。

当你使用 Modules 引入时，预处理器并不会像 “#include”那样使用 M*N 量级的重复拷贝粘贴。而是巧妙地通过一个列表来存放已经编译处理过的 Modules 列表，而声明的引入会首先在这个表内查找，如果没有找到会去编译添加进来。所以 Modules 的引入只会被处理一次，可以解决前面提到的引用泛滥问题。

# 使用和生产 Modules

自 Xcode5以来，build settings 都默认开启了 “-fmodules”，一般来讲你的代码里面都可以使用 Modules 来引入其他库。其实 modules 是一种头文件编译后的 map，所以 Modules 始终都能保证你所引入的定义是存在的、有意义的。（其实 Modules 是一种从 precompile headers 演变过来的技术）

modules 和 headers 通过一个 map 来进行一种关系映射，这个 map 文件就叫做 modulemap. 这个文件从语义上描述了你的函数库物理结构。

我们举个例子，用 std 这个 module 来描述 C 的标准库。那么 C 标准库里面的那些头文件：stdio.h, stdlib.h, math.h 都可以映射到 std 这个 module 里面，他们就组成了几个 子模块（submodule）: std.io, std.lib, std.math。通过这样一个映射关系，C 的标准库就可以构建出一个独立的 module。所以通常地，一个库就只有一个 module.modulemap 文件用于描述它的所有头文件映射。

那么实际在编译过程中 Modules 到底代表着什么呢？我们前面说过其实 Modules 是一种预编译技术，当一个模块被导入时，编译器在处理它时会生成一个新的子进程（非 fork），这个子进程拥有干净的 context来编译这个 module（这样就不会产生命名空间冲突等干扰），然后 module 的编译结果会被持久化到这个模块的二进制缓存中，那么下次引用编译的时候就会非常快。 modules 由头文件映射而成，所以当这些头文件改动时，module 还会自动重新编译刷新缓存，不需要我们主动干预。

要构建一个 module，首先从编译参数上就要进行一系列的设置，不过这些参数有点多，也有点难以人为去组合，幸运的是至少现版本的 Xcode 8是可以无缝支持的（预先设置好了），所以不用太 care。接下来我们看一下构建 modules 的关键— modulemap 文件的语法。

```c
module std [system] [extern_c] {
  module assert {
    textual header "assert.h"
    header "bits/assert-decls.h"
    export *
  }
 
  module complex {
    header "complex.h"
    export *
  }
 
  module ctype {
    header "ctype.h"
    export *
  }
 
  module errno {
    header "errno.h"
    header "sys/errno.h"
    export *
  }
 
  module fenv {
    header "fenv.h"
    export *
  }
 
  // ...more headers follow...
}
```

这种东西你可能在用 Swift Package Manager 时碰到过。modulemap 文件的语法其实用了 C99的词法处理的简化版，但仍然具有相同的标识符、单词、字符串字面量和单行/多行注释等规则。它的保留字如下：

```c
config_macros export     private
conflict      framework  requires
exclude       header     textual
explicit      link       umbrella
extern        module     use
```

然后，我们再看一下 modulemap 文件的具体组成，用简单的语法推导范式来描述如下，就是说由若干个模块声明组成。

```c
modulemap:
    module-declaration*
```

模块声明就比较详细了，语法推导式也复杂一点点：

```c
module-declaration:
  explicitopt frameworkopt module module-id attributesopt '{' module-member* '}'
  extern module module-id string-literal
```

（带有 opt 后缀的成员表示都是可选的）

从语法推导来看，module 可以是声明一个模块，也可以是前置声明一个外部模块。不过我暂时没见过第二种的使用情况，所以本文也只讨论第一种。

explicit 关键字只适用于子模块，表示只有导入语法指定导入这个子模块时才能导入，而不会因为导入整个模块而一并导入。

framework 关键字表示一个模块的组织形式是 Darwin 的 framework ，这时候 modulemap 文件是直接包含在framework 里面的。framework 我们常用，所以我就不啰嗦具体的文件结构了。

关于 attribute 选项有好几个关键字可以用：

system 关键字表示的是这个模块是一个系统库，然后具体的头文件映射会在系统库路径寻找。

extern_c 关键字表示这个模块包含 C 代码，并且想在 C++中作为模块使用，实际上和我们使用 extern “C”的效果是一样的。

接下来，继续介绍 Modules 的 module-member 组成：

```c
module-member:
  requires-declaration
  header-declaration
  umbrella-dir-declaration
  submodule-declaration
  export-declaration
  use-declaration
  link-declaration
  config-macros-declaration
  conflict-declaration
```

乍一看这组成部分也忒多了点，我们着重看一下几个常用的。

**header** 声明：这个比较简单，就是表明有哪些 header 组成这个 module。不过除了最普通的声明，还有 exclude header，表明把哪个 header 排除在外。更常见的是umbrella header，它表明这个 header 包含了当前目录下的所有的其他头文件（包括子目录中的），这种形式我们在 framework 的对外声明头中很常见。

在 header 声明中有两个例外的关键字：private 和 textual。private 顾名思义就是这个 header 不向外部暴露为模块，但内部的子模块之间可以互相引用。 textual 就比较难解释了，必须举个例子：我们常用的 assert 其实不是一个具有实际意义的函数，它只是一个宏，用于预处理期替代某处的 assert 宏引用。假如它被 module 预编译了岂不是很尴尬，所以 textual 关键字还会让这些头文件保持预编译期和 include 关键字同样的处理效果。

**export** 声明：这个比较简单，就是说这个模块会重新导出任何自己导入的其他模块。举个例子：

```c
module MyLib {
  module Base {
    header "Base.h"
  }
 
  module Derived {
    header "Derived.h"
    export Base
  }
}
```

很清晰的就是你在引入 MyLib.Derived时，也能一并引入 MyLib.Base 了。不过我们有时候并不清楚自己写一个模块代码时到底引入了多少个其他模块，所以可以简单地 export * 进行全部导出。

嗯，header 和 export 就是两个常用的模块组成关键字，没别的再需要继续啰嗦了，如果还想了解更多，建议是继续详细阅读 [Clang](https://clang.llvm.org/docs/Modules.html)文档 。

有了上述的基础，我们就可以生产 Modules 了，具体如何生产和最佳实践，我们继续往下看。

# Modules 和 Swift
如果仅仅只是为了提升头文件预处理速度还没必要这么大费周章地搞 Modules 这个东西，我的猜测是 Swift 这个项目开始设计时便考虑了和 C/C++/ObjC 的交互问题，使用 Modules 便可以方便桥接了。

Swift 的 Modules和 我们上面讲的稍微有点不一样，它并不存在 modulemap 这个东西，而是直接编译生成的一个 .swiftmodule 文件。Apple 官方对于 Swift 的模块系统也有一点解释，就是说 Xcode 中的每一个 target 都对应着一个 Swift Module。

我们前面提到 modulemap 最终预编译后产生的是一个二进制的缓存，Swift Modules 也一样，.swiftmodule 文件里面存放的就是一些序列化后的 AST （可能还有些 SIL）。因为 Swift 并没有头文件引入机制，所以 Swift 和 C/C++/ObjC 交互时，通过这种 Modules 机制，从二进制层面上交互会非常便捷。最终进行编译链接便能确定互相调用函数或对象的相对或绝对地址和内存布局了。

（我个人对 Swift 并没有十分了解，所以仅仅从实现层面进行了一些猜测，如有错误还请指出。）

# 在 LLVM9.0前让 Swift 使用 objc静态库

这一段，我介绍一下当前我对使用 objc 和 Swift 混编时一点小小的心得。

当前 LLVM 8.0并不支持让 Swift 编译出静态库，而 objc 的静态库也很难方便地交付给 Swift 项目使用。（如果你喜欢直接在项目里面 embed library 加 bridging header 当我什么都没说）

先介绍一下背景：一个非常庞大的 iOS App 项目，所有的模块、依赖全都由 cocoapods 管理，build target 为 iOS 8.0，如果 all use_framework，那么势必造成 App 启动时载入动态库耗费过多的时间。如果不使用，你就 貌似没办法在模块或主工程内支持 Swift，因为 cocoapods 安装后默认的所有 target 都是静态库。

这时候做一个假设，使用 objc 开发的库或模块，pod 交付时是一个静态库，使用 Swift 交付的pod是一个动态库（framework）。那么我们十分希望这些交付的库和模块之间能够互相调用或依赖。但很明显需要支持 modules 才能让 Swift 库调用静态库。

让 Swift 通过 modules 调用静态库不是什么难事，只要你在编译选项中加上这个静态库的 modulemap 文件，Swift 代码就可以正确地引入静态库依赖了。不过这不是什么最佳实践，如果你两个 pod 互相依赖还要通过脚本去改变 build settings 不是什么好的选择。所以有必要把静态库 pod 的交付产物打包成一个静态库 framework，这样编译器就能自动寻找 modulemap 了！

打包一个静态库（.a 文件）相信不是什么难事，不过不要忘记合并 armv7 /arm64 和 i386/x86_64架构。然后你需要创建一个 MyLib.framework 文件夹，把 MyLib.a 拷贝进去并重新命名为 MyLib。把所有的头文件拷贝到 MyLib.framework/Headers 文件下，把所有的 bundle 拷贝到 MyLib.framework/ 下，然后创建一个  MyLib.framework/Modules 文件夹，开始写入 module.modulemap 文件，怎么写呢？

扫一遍 MyLib.framework/Headers目录，记录一下到底有多少个头文件，然后生成一个 MyLib-umbrella.h 文件，在里面这样写：

```c
#ifdef __OBJC__
#import <UIKit/UIKit.h>
#else
#ifndef FOUNDATION_EXPORT
#if defined(__cplusplus)
#define FOUNDATION_EXPORT extern "C"
#else
#define FOUNDATION_EXPORT extern
#endif
#endif
#endif
 
#import "MyLibSomeFun.h"
//other headers
```

通过扫描目录你可以用脚本写入到 MyLib.framework/Headers/MyLib-umbrella.h 文件里面。值得一提的是，如果你已经写过一个 MyLib.h 这样的 umbrella header，如果里面用了 <>这种引入语法是不行的，因为静态库并不会真的成为 framework 那样的“系统库”，所以这种引入语法会导致 modules 无法编译，你可以通过脚本判断把这个头文件 exclude 掉。

然后继续写入 modulemap文件到MyLib.framework/Modules/module.modulemap ：

```c
framework module MyLib {
  umbrella header "MyLib-umbrella.h"
  export *
  module * { export * }
}
```

这时候你就得到了一个静态库 framework，把它直接拖进你的 target embed libraries 可以直接使用，也可以作为 pod 的 vendor framework 使用。当你在 Swift 代码里面想引用的时候，直接 @import MyLib 就行了。

最终看编译产物的时候，这个静态库 framework 不会拷贝到 Demo.app/Frameworks 下面，因为它不是动态库，不需要运行时 link。它在链接期直接链接进了最终的 binary excutable。而你的 Swift 代码也能正确地链接到它的地址，这样就不需要 bridging header 了。

不过很明显能够看到一些缺点就是：这个静态库 framework 暂时并不支持内部使用 Swift，Swift 代码库或模块仍然需要编译成动态库或直接编译进 excutable。并且，库和模块需要交付打包后的二进制产物，如果用源码一起编译，仍然需要很麻烦地设置 modulemap path。不过交付二进制的好处是当一个工程很大的时候，这些依赖编译就会很快（因为已经编译过了，只需要链接）。

# The End
静态库 framework as Modules 是我最近尝试的一个实践，至少我认为是二进制交付体系下的最佳实践，至少避免了动态库运行时载入过慢这个弊端，仍然支持了 modules，为现有的库将来在 Swift 中的运用铺平了道路。