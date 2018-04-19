---
title: 了解 Clang AST
date: 2018-01-08 20:07:59
tags:
---

Clang 作为 世界上最好的编译器前端（可能没有之一），散发着独特的令人着迷的魅力，让人无法抗拒想去更多地了解它。
通过本文，你大概能知道：

* 什么是 Clang AST
* Clang AST 有那些东西
* 怎么搭建 LLVM/Clang开发环境
* 站在巨人的肩膀上（通过 Clang提供的功能编写一个小工具）

跟随本文的教程，你可能需要以下的基础准备：

* 有基本的 C/C++编程基础
* 直接或间接地使用过 Clang，看得懂一些命令行参数是干什么的
* 有一点抽象编程的思考能力

<!--more-->


# 什么是 Clang AST
大家都知道 AST（Abstract Syntax Tree）是编译前端执行过程中必不可少的中间产物，你写的代码被编译器前端翻译成了一种抽象的，可用有限的语言描述的树状结构的东西。
Clang AST也没有脱离这个本质，但它跟其他编译器所生成的 AST相比多了很多非常有用的东西，这些东西可以让 Clang AST表现得就像普通的 C++代码变量，让操作 AST和从 AST获取信息非常变得非常容易。

不过在深入了解 Clang AST前，我们适当补充一些 AST相关的知识：如何用 AST来表示代码？

## 代码的抽象表示
代码的量和编写形式是无限的，但是 AST所能表示形式是有限的，如何用有限的形式表示无限的代码？这是一个问题。所以我们需要用一种抽象的形式来表示代码。
为了找个比较简单的研究案例，这里贡献一下拙作 enginx关于语法抽象代码的一个例子：

```c
typedef enum {
  ENGINX_BOOLEAN_VALUE = 1,
  ENGINX_STRING_VALUE,
  ENGINX_INT_VALUE,
  ENGINX_NULL_VALUE,
  ENGINX_IDENTIFIER_VALUE
} ENGINX_VALUE_TYPE;
```

在 enginx中，我把所有的用于表示数据的类型都归纳为一个类型：**ENGINX\_VALUE\_TYPE**，这是一种简单的抽象，这样我们在语法分析的时候，只要遇到基础数据类型，我们都可以将它归纳到这里。
同样地，Clang的 AST中，类型的表示就是 Type，具体到某个语言的类型时便可以派生出 **PointerType**（指针类型）、**ObjCObjectType**（objc对象类型）、**BuiltinType**（内置基础数据类型）这些表示。
在 Clang的定义中，编程语言中无外乎包含三个东西：Type(类型），Decl(声明），Stmt（陈述），通过这三者的联结、重复或选择（alternative)就能构成一门编程语言。举个例子，下图的一段代码：

<img src="http://cdn.stephenw.cc/wp-content/uploads/2018/01/BNF.png" style="max-width:350px"/>

FunctionDecl、ParmVarDecl 都是基于 Decl派生的类，CompoundStmt、ReturnStmt、DeclStmt都是基于 Stmt派生的类。）

从上图中可以看到，一个**FunctionDecl**（函数的实现）由一个 **ParmVarDecl**联结 **CompoundStmt**组成。而这个函数的 **CompoundStmt** 由 **DeclStmt**和 **ReturnStmt**联结组成。如果我们继续深挖一层的话，还可以发现这段代码的**ParmVarDecl**由 **BuiltinType**和一个标识符字面量联结组成。
很明显一门编程语言中还有很多其他形态，我们都可以用这种方式描述出来。所以说从抽象的角度看，拥有无限种形态的编程语言便可以用有限的形式来表示。

## AST的结构
毫无疑问，实际的代码产生的 AST结构肯定非常复杂，我们可以先管中窥豹一哈。先写一个最简单的 C代码文件：

```c
// test.c
 int someFunc(int x) {
  int result = x / 42;
  return result;
 }
```

然后，使用 Clang自带的 ast dump插件查看这端 C代码由 Clang分析产生的 AST：

```shell
clang -Xclang -ast-dump -fsyntax-only test.c
```

结果如图所示：

![](http://cdn.stephenw.cc/wp-content/uploads/2018/01/ast-dump.png)

可以看到， Clang AST的最顶层结构叫做 **TranslationUnit**，我们管它叫做“编译单元”。它的子节点前面跟了很多个 **TypedefDecl**，这些都是 Clang的内置定义，可以先不用管。然后我们碰到了 **FunctionDecl**，整体结构和我们上文例举的图基本一致，但是它的层次要更丰富一点，原理一致就不赘述了。

不过，我们从 Clang AST dump中可以看到每一个语法节点包含的信息还是比较多的，比如 **FunctionDecl**在 AST的内存地址是 0x7fecf7017538，它处于 <test.c:2:1, line:5:1> 的位置。这些信息有什么用？这正是我们接下来要讨论的 Clang AST的不同之处。

# Clang AST的作用
如果你阅读过拙作 enginx的 AST源码，你会发现 enginx 的 AST还是 too young too simple的，甚至寻找一个节点都需要遍历整个树，每个节点所携带的信息仅仅只有语义而已，所以 enginx只能基于 AST解释运行，做不了更多其他的事。
我们从上文的 ast dump就能看到 Clang AST每个语法节点包含的信息是很丰富的，其实它的丰富程度远超过想象，所以我们在拿到 Clang AST的时候，用起来不会像 enginx的 AST那样捉襟见肘。
Clang AST 除了拥有像普通的编译器编译意义上的作用，其实更多地给普通开发者带来了参与到编译流程的能力，我们接下来通过一个简单示例展示如何使用 Clang AST的 API。

## 搭建 LLVM/Clang开发环境
网络上关于 LLVM/Clang开发环境的实践比较少，我们可以按照官方的一些文档顺藤摸瓜着去做：

* checkout 最新的 LLVM release分支
* 把 clang checkout 到 llvm/tools/clang 目录
* checkout clang-tools-extra 到 llvm/tools/extra目录
* checkout compiler-rt 到 llvm/tools/compiler-rt 目录
* 创建 build目录
	* 如果习惯使用 cmake，使用命令 cmake /path/to/llvm
	* 如果习惯使用 Xcode，使用命令 cmake -G Xcode /path/to/llvm

## 示例: 基于符号的定义查找
首先解释一下什么是基于符号的定义查找：我们在 IDE中查找一个类的定义时基本上都是基于符号的查找（或者混合文字匹配），这和普通的文字编辑器的匹配是不同的。比如你在 Xcode中点击跳转到一个类的定义，这很显然是基于符号分析实现的，IDE不会仅仅简单地通过文字匹配，假如通过文字匹配，很容易跳转到注释或某一处非常近似的语句中。

很显然，要实现符号的查找和定位，我们需要借助于语义分析来实现。假如 enginx（或者任何一个简单的其他拥有 AST结构的程序）需要实现编辑器这样一个辅助功能，查找一个符号显然只能通过不断地遍历 AST实现，效率很低，但好在 enginx AST的结构非常简单，遍历的开销并不高。但是这样的问题照搬到 C、C++、Objective-C和 Swift这些用于构建庞大应用，动辄数十万行量级的代码程序上，重复遍历的效率就无法令人接受了。
为此 Clang AST 封装了非常多、非常强大的 API可供我们使用，我们可以借助这些 API实现这次的示例: **从 C++代码中 查找一个指定名称为 Demo的 Class，给出具体位置**。

示例代码：	

```c++
namespace ClangTutorial {
    class Demo {}
}
```

### 创建工程
在 `llvm/tools/clang/examples`创建 FindNamedClass文件夹，在 `llvm/tools/clang/examples/CMakeLists.txt`最后添加一行：

```cmake
add_subdirectory(FindNamedClass)
```

在 **FindNamedClass**文件夹添加 main.cpp文件，并添加 CMakeLists.txt文件，修改内容如下：

```cmake
add_clang_executable(findNamedClass main.cpp)
 
target_link_libraries(findNamedClass PRIVATE clangTooling)
```

至此，我们工程文件创建好了，接下来可以正式写代码了。在写之前，需要到你的 build目录再 cmake一遍，确保配置到最新了。

### Clang Tools

好吧，到这时候我们又引入了一个概念叫 Clang Tools。从上面的 CMake配置来看，我们这次要写的示例代码其实是一个独立运行的命令行程序，使用了 **libclangTooling**这个库。所以，Clang Tools就是通过 Clang提供的函数库实现的独立运行的工具。同样类型的 Clang Tools还有 clang-check、clang-fixit、clang-format这些大名鼎鼎的工具。

### RecursiveASTVisitor

在要动手写代码前，我们需要了解一下 Clang的库提供的这样一个封装。在拿到 Clang提供的 AST时，其顶层结构肯定是一个“编译单元”，去查找某个节点我们可以进行遍历，但也可以使用 Clang封装好的访问算法，这个算法的封装就是**RecursiveASTVisitor**。 我们继承这个类，并且实现其中的 **VisitCXXRecordDecl**，那么这个方法就会在访问 **CXXRecordDecl**类型的节点上触发。(CXXRecordDecl 类型用于表示 C++ class/union/struct)

```c++
class FindNamedClassVisitor : public RecursiveASTVisitor&lt;FindNamedClassVisitor&gt; {
public:
  bool VisitCXXRecordDecl(CXXRecordDecl *Declaration) {
    // 可以 dump一下看看到底有什么
    if (Declaration-&gt;getQualifiedNameAsString() == "ClangTutorial::Demo")
        Declaration-&gt;dump();
    
    // 这个返回值表示是否需要接着访问其他节点
    return true;
  }
};
```

### FrontendAction和 ASTConsumer
前面我们已经实现了 **RecursiveASTVisitor**, 按道理已经可以实现基于 Clang AST查找符号的核心逻辑了，但是很明显我们需要一个入口，这个入口需要定义我们如何通过一个编译实例(compiler instance)获得整个 AST以及如何进行 AST的访问。
**FrontendAction**就是一个与编译实例打交道的东西，词法分析、语法分析等过程都被编译实例隐藏了，编译实例会触发 **FrontendAction**定义好的一些方法，并且把编译过程中的详细信息告诉它，比如编译了哪个文件、编译参数等。而 **ASTConsumer**其实是 **FrontendAction**的一个子过程产物，在 **ASTConsumer**中我们便可以拿到整个“编译单元”，调用前面的 Visitor进行访问和查找。

```c++
class FindNamedClassConsumer : public clang::ASTConsumer {
public:
  explicit FindNamedClassConsumer(ASTContext *Context) : Visitor(Context) {}
  virtual void HandleTranslationUnit(clang::ASTContext &amp;Context) {
    // 通过 RecursiveASTVisitor访问 translation unit会遍历 AST所有节点  
    Visitor.TraverseDecl(Context.getTranslationUnitDecl());
  }
private:
  // RecursiveASTVisitor 的具体实现.
  FindNamedClassVisitor Visitor;
};
 
class FindNamedClassAction : public clang::ASTFrontendAction {
public:
  virtual std::unique_ptr&lt;clang::ASTConsumer&gt; CreateASTConsumer(clang::CompilerInstance &amp;Compiler, llvm::StringRef InFile) {
    return std::unique_ptr&lt;clang::ASTConsumer&gt;(new FindNamedClassConsumer(&amp;Compiler.getASTContext()));
  }
}
```

### main函数
为了简化问题，我们直接调用 clangTooling的 runToolOnCode方法，传入代码字符串即开始解析。

```c++
int main(int argc, char **argv) {
    if (argc &gt; 1) {
        clang::tooling::runToolOnCode(new FindNamedClassAction, argv[1]);
    }
}
```

然后我们进行编译和测试

```shell
$ ./findNameClass "namespace ClangTutorial { class Demo {}; }"
```

如图所示

![](http://cdn.stephenw.cc/wp-content/uploads/2018/01/decl-dump.png)

我们已经能实现查找到指定的符号定义并 dump出来了，但是还没想到如何具体地获取它的位置等信息，要解决这个问题我们需要再引入一个概念。

### ASTContext

在前面的代码里面，我们偷偷地用了 **ASTContext**却没有介绍它。ASTContext其实就是编译实例保存所有 AST信息的一种结构，它主要包括编译期间的符号表和 AST原始形式。我们也是直接从 ASTContext里面获取到了整个 “编译单元”。
有了符号表，我们便可以获取到符号在 AST中的原始定义，不过 Clang的 API已经隐式地帮我们做了这件事：

```c++
if (Declaration->getQualifiedNameAsString() == "ClangTutorial::Demo") {
    FullSourceLoc FullLocation = Context->getFullLoc(Declaration->getLocStart());
    if (FullLocation.isValid())
      llvm::outs() << "Found declaration at "
                   << FullLocation.getSpellingLineNumber() << ":"
                   << FullLocation.getSpellingColumnNumber() << "\n";
}
```

只要通过上面代码中几个简单的 API和对象封装，我们便获得了符号定义的原始位置，我们再编译运行一遍，就获得了结果：

```shell
Found declaration at 1:27
```

通过 Clang AST API，其实除了获取符号定义在源文件的位置，我们还可以获取方法的调用关系、类型定义以及源代码内容等，非常强大。

# 写在后面
LLVM在设计之初就被设计为一系列库，Clang也是如此。通过这一系列库，开发者可以实现各种各样强大的功能，玩转编程语言。本文的 Clang AST只涉及 Clang的若干库中的一两个，更多更强大的功能可以访问 [Clang官方文档](https://clang.llvm.org/docs/index.html)查看。

本文参考了以下文章或网站：

* [Clang 6 documentation/Introduction to the Clang AST](http://clang.llvm.org/docs/IntroductionToTheClangAST.html)
* [Clang doc](https://clang.llvm.org/doxygen)
* [Clang 6 documentation/How to write RecursiveASTVisitor based ASTFrontendActions](http://clang.llvm.org/docs/RAVFrontendAction.html)

推荐书籍：

* 《自制编程语言》前桥和弥 著
* 《Modern Compiler Implementation in C》 (美)安佩尔 著
* 《Compiler》 Alfred V.Aho等 著
* 《Getting Started with LLVM Core Libraries》Bruno Cardoso Lopes / Rafael Auler 著

附文中示例完整代码：

```c++
#include "clang/AST/ASTConsumer.h"
#include "clang/AST/RecursiveASTVisitor.h"
#include "clang/Frontend/CompilerInstance.h"
#include "clang/Frontend/FrontendAction.h"
#include "clang/Tooling/Tooling.h"
 
using namespace clang;
 
class FindNamedClassVisitor: public RecursiveASTVisitor<FindNamedClassVisitor> {
public:
    explicit FindNamedClassVisitor(ASTContext *Context) : Context(Context) {}
    
    bool VisitCXXRecordDecl(CXXRecordDecl *Declaration) {
        if (Declaration->getQualifiedNameAsString() == "ClangTutorial::Demo") {
            FullSourceLoc FullLocation = Context->getFullLoc(Declaration->getLocStart());
            if (FullLocation.isValid())
                llvm::outs() << "Found declaration at "
                << FullLocation.getSpellingLineNumber() << ":"
                << FullLocation.getSpellingColumnNumber() << "\n";
        }
        return true;
    }
private:
    ASTContext *Context;
};
 
class FindNamedClassConsumer : public clang::ASTConsumer {
public:
    explicit FindNamedClassConsumer(ASTContext *Context) : Visitor(Context) {}
    
    virtual void HandleTranslationUnit(clang::ASTContext &Context) {
        Visitor.TraverseDecl(Context.getTranslationUnitDecl());
    }
private:
    FindNamedClassVisitor Visitor;
};
 
class FindNamedClassAction : public clang::ASTFrontendAction {
public:
    virtual std::unique_ptr<clang::ASTConsumer> CreateASTConsumer(clang::CompilerInstance &Compiler, llvm::StringRef InFile) {
        return std::unique_ptr<clang::ASTConsumer>(new FindNamedClassConsumer(&Compiler.getASTContext()));
    }
};
 
int main(int argc, char **argv) {
    if (argc > 1) {
        clang::tooling::runToolOnCode(new FindNamedClassAction, argv[1]);
    }
}
```
