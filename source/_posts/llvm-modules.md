---
title: LLVM Modules
date: 2017-08-23 10:26:44
tags:
---

Modules are actually the build system released with Xcode5, very early and very old, but still not receiving enough attention.

Until the arrival of Swift.

This article takes you to explore:

1. What is Modules
2. Use and produce Modules
3. Modules and Swift
4. Let Swift use the objc static library before LLVM9.0

<!--more-->

# What is Modules
In fact, the explanation of modules on the web blog is a little bit small (or not detailed enough). Let's first understand the background of modules:

Most software (App) uses many, many libraries to build, so when we write code, we will definitely introduce the header files of these libraries to use the convenient methods they provide, such as this:

```c
#include <SomeLib.h>
```

(Assuming that you already know the similarities and differences between #include and #import, I won’t be too long on this part)

The specific implementation of this header file method is actually handled by the SomeLib library, so we need to provide a parameter "-lSomeLib" to the linker so that the linker can know where the specific implementation is. But there is a bit of a pitfall in this, usually software developers will not notice, Modules is to solve these problems, please continue to read.

Question 1: We all know that the header file introduced by the "#include" keyword actually simply and rudely copy the content of the target header file in. Suppose your header file is included in the header files of M other libraries, and this M On average, N header files are introduced in each header file. The result is that the compiler needs to process M*N header files in the preprocessing stage. This may introduce a very large amount of code in your header file. If you switch to C++, the situation is even worse, because it also contains some template code, which is several times more than C.

Question 2: When you use these keywords "#include", the included header file text will be more complete. So if you define macro definitions with the same name in two different libraries, the result is very subtle, and it is likely that the final runtime result will be unexpected, which makes the preprocessing very "fragile".

Question 3: In order to solve the problem 2, if the developers agree that different libraries have different prefixes, then a definition name will be very verbose, or some developers will use underscores as the beginning of these names, making the entire library The naming of has become very ugly, which is an obstacle for non-C programmers.

Question 4: When to use which tools and libraries to develop software, you can’t understand it just from the header file, because it is not “semantic”, such as which namespaces belong to a specific library, and these namespaces In what order to include, or you only want to import a part of the definition of this library, it is really too difficult to figure out just by preprocessing like "#include".

The above four problems are actually a simple translation of the problems to be solved by Modules in the Clang documentation.

So **Modules** is a semantic method of introducing definitions. It is used to replace the ordinary "#include" series of instructions.

```c
@import WebKit.WebKitLegacy; //in Objective-C
import WebKit.WebKitLegacy //in Swift
```

You can see that both Objective-C and Swift support Modules import very well, and you can introduce API declarations very clearly.

When you use Modules to import, the preprocessor does not use M*N repeated copy and paste like "#include". Instead, it cleverly uses a list to store the list of modules that have been compiled and processed, and the introduction of the declaration will first be searched in this table, and if it is not found, it will be compiled and added. Therefore, the introduction of Modules will only be processed once, which can solve the aforementioned problem of reference flooding.

# Use and produce Modules

Since Xcode5, "-fmodules" has been turned on in build settings by default. Generally speaking, you can use Modules in your code to introduce other libraries. In fact, modules are a map compiled from header files, so Modules can always ensure that the definitions you introduce exist and are meaningful. (Actually, Modules is a technology evolved from precompile headers)

Modules and headers are mapped through a map. This map file is called modulemap. This file semantically describes the physical structure of your library.

Let's take an example, using the std module to describe the C standard library. Then the header files in the C standard library: stdio.h, stdlib.h, math.h can all be mapped to the std module, and they form several submodules: std.io, std.lib, std .math. Through such a mapping relationship, the C standard library can build an independent module. So usually, a library has only one module.modulemap file to describe all its header file mappings.

So what does Modules actually represent during the compilation process? We mentioned earlier that Modules is actually a pre-compilation technology. When a module is imported, the compiler will generate a new child process (not fork) when processing it. This child process has a clean context to compile the module ( In this way, there will be no interference such as namespace conflicts), and then the compilation result of the module will be persisted in the binary cache of this module, so the next time the reference is compiled, it will be very fast. Modules are mapped from header files, so when these header files change, the module will automatically recompile and refresh the cache without our active intervention.

To build a module, a series of settings must be made from the compilation parameters, but these parameters are a bit too many, and a bit difficult to combine artificially, fortunately at least the current version of Xcode 8 can be seamlessly supported (pre-set ), so don’t be too care. Next we look at the key to building modules-the syntax of the modulemap file.

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

You may have encountered this kind of thing when using Swift Package Manager. The syntax of the modulemap file uses a simplified version of C99's lexical processing, but still has the same rules for identifiers, words, string literals, and single-line/multi-line comments. Its reserved words are as follows:

```c
config_macros export private
conflict framework requires
exclude header textual
explicit link umbrella
extern module use
```

Then, let's take a look at the specific composition of the modulemap file, using a simple grammatical deduction paradigm to describe it as follows, that is, it consists of several module declarations.

```c
modulemap:
    module-declaration*
```

The module declaration is more detailed, and the grammatical deduction is also a little more complicated:

```c
module-declaration:
  explicitopt frameworkopt module module-id attributesopt'{' module-member*'}'
  extern module module-id string-literal
```

(Members with the suffix opt are optional)

From the perspective of grammatical derivation, module can either declare a module or pre-declare an external module. However, I haven't seen the second use case for the time being, so this article only discusses the first one.

The explicit keyword is only applicable to submodules, which means that the submodule can be imported only when the import syntax specifies that the submodule is imported, instead of importing the entire module.

The framework keyword indicates that the organization of a module is Darwin's framework, and the modulemap file is directly included in the framework at this time. framework is commonly used by us, so I won’t go into the specific file structure.

There are several keywords available for the attribute option:

The system keyword indicates that this module is a system library, and then the specific header file mapping will be found in the system library path.

The extern_c keyword indicates that this module contains C code and wants to be used as a module in C++. In fact, the effect is the same as when we use extern "C".

Next, continue to introduce the module-member composition of Modules:

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

At first glance, this component is a little bit more exciting, let's focus on a few commonly used ones.

**header** statement: This is relatively simple, it means which headers make up this module. However, in addition to the most common declarations, there is also an exclude header, which indicates which header to exclude. The more common is umbrella header, which indicates that this header contains all other header files in the current directory (including those in subdirectories). This form is very common in the external declaration header of the framework.

There are two exception keywords in the header declaration: private and textual. Private, as the name implies, is that this header is not exposed to the outside as a module, but internal submodules can refer to each other. Textual is more difficult to explain. We must give an example: our commonly used assert is not actually a meaningful function, it is just a macro, used to replace the assert macro reference somewhere in the preprocessing period. Wouldn't it be embarrassing if it was pre-compiled by module, so the textual keyword will keep these header files in the same pre-compilation period as the include keyword.

**export** Statement: This is relatively simple, which means that this module will re-export any other modules imported by itself. for example:

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

It is very clear that when you introduce MyLib.Derived, you can also include MyLib.Base. But sometimes we don't know how many other modules are introduced when we write a module code, so we can simply export * to export all of them.

Well, header and export are two commonly used modules to form keywords. There is nothing else to continue. If you want to know more, I suggest you continue to read [Clang](https://clang.llvm.org/) docs/Modules.html) document.

With the above-mentioned foundation, we can produce Modules. For specific production and best practices, we will continue to look down.

# Modules and Swift
If just to improve the preprocessing speed of header files, there is no need to spend so much time on Modules. My guess is that when the Swift project was designed, the interaction with C/C++/ObjC was considered. Modules can be used. Convenient to bridge.

Swift's Modules are a little bit different from what we said above. There is no such thing as a modulemap, but a .swiftmodule file generated by direct compilation. Apple officially has a little explanation for Swift's module system, which means that each target in Xcode corresponds to a Swift Module.

We mentioned earlier that the final pre-compilation of modulemap will produce a binary cache. The same is true for Swift Modules. The .swiftmodule file contains some serialized ASTs (and possibly some SIL). Because Swift does not have a header file introduction mechanism, when Swift interacts with C/C++/ObjC, through this Modules mechanism, the interaction from the binary level will be very convenient. Finally, compiling and linking can determine the relative or absolute address and memory layout of the functions or objects that call each other.

(I personally don’t know much about Swift, so I just made some guesses from the implementation level. Please point out if there are errors.)

# Let Swift use the objc static library before LLVM9.0

In this paragraph, I would like to introduce my little experience when using objc and Swift to mix together.

Currently LLVM 8.0 does not support Swift to compile static libraries, and objc static libraries are also difficult to easily deliver to Swift projects. (If you like to add bridging header to embed library directly in the project, when I didn’t say anything)

Let me first introduce the background: a very large iOS App project, all modules and dependencies are managed by cocoapods, and the build target is iOS 8.0. If all use_framework is used, it will inevitably cause too much time to load the dynamic library when the App starts. If you don't use it, you seem to be unable to support Swift in the module or main project, because all targets by default after cocoapods are installed are static libraries.

At this time, make an assumption. When using the library or module developed by objc, the pod is a static library when delivered, and the pod delivered by Swift is a dynamic library (framework). Then we very much hope that these delivered libraries and modules can call or depend on each other. But obviously you need to support modules in order for the Swift library to call static libraries.

It is not difficult for Swift to call the static library through modules. As long as you add the modulemap file of this static library in the compilation options, the Swift code can correctly introduce the static library dependencies. But this is not a best practice. If your two pods depend on each other, it is not a good choice to change the build settings through scripts. So it is necessary to package the delivery product of the static library pod into a static library framework, so that the compiler can automatically find the modulemap!

It is not difficult to pack a static library (.a file), but don't forget to merge armv7 /arm64 and i386/x86_64 architectures. Then you need to create a MyLib.framework folder, copy MyLib.a into it and rename it MyLib. Copy all the header files to the MyLib.framework/Headers file, copy all the bundles to MyLib.framework/, then create a MyLib.framework/Modules folder and start writing the module.modulemap file. How to write it?

Scan the MyLib.framework/Headers directory, record how many header files there are, and then generate a MyLib-umbrella.h file, write it like this:

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

By scanning the directory, you can write scripts into the file MyLib.framework/Headers/MyLib-umbrella.h. It is worth mentioning that if you have already written an umbrella header like MyLib.h, it will not work if you use the <> import syntax, because static libraries will not really become "system libraries" like frameworks. , So this kind of import syntax will cause modules to fail to compile, you can exclude this header file through script judgment.

Then continue to write the modulemap file to MyLib.framework/Modules/module.modulemap:

```c
framework module MyLib {
  umbrella header "MyLib-umbrella.h"
  export *
  module * {export *}
}
```

At this time, you will get a static library framework, drag it directly into your target embed libraries, you can use it directly or as a vendor framework for pod. When you want to quote in Swift code, just @import MyLib directly.

When finally looking at the compiled product, this static library framework will not be copied to Demo.app/Frameworks, because it is not a dynamic library and does not require runtime link. It is directly linked into the final binary excutable during the link period. And your Swift code can correctly link to its address, so there is no need for bridging header.

However, it is obvious that some shortcomings can be seen: this static library framework does not support internal use of Swift for the time being, and the Swift code library or module still needs to be compiled into a dynamic library or directly compiled into excutable. In addition, libraries and modules need to deliver packaged binary products. If they are compiled together with source code, it is still troublesome to set the modulemap path. However, the advantage of delivering binary is that when a project is large, these dependencies will be compiled quickly (because it has already been compiled, only linking is required).

# The End
The static library framework as Modules is a practice I recently tried. At least I think it is the best practice under the binary delivery system. At least it avoids the disadvantage of slow loading of dynamic libraries during runtime.