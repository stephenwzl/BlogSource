---
title: objc的 Runtime
date: 2017-06-29 19:24:04
tags:
---

<img src="/images/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7-2017-06-29-%E4%B8%8B%E5%8D%881.42.46.png" style="max-width: 350px;"/>  

objc 的 Runtime 是一个被网络热帖说烂了的词，好事者还喜欢给它套上一个“黑魔法”的帽子。然而，这个世界上没有什么魔法，一切都是可以从科学角度解释的，它也不仅仅是在头文件里面罗列的几个 C函数。本文我们来探讨一下 objc 的 Runtime 是什么。

<!--more-->

# Objective-C的本质
很多编程语言都有运行时，比如 JVM之于 Java，JavascriptCore/v8之于 Javascript等等（其实这里我们应该称它为运行环境）。然而单单 objc显得尤为特殊，因为它的运行时既不是一个虚拟机，也不是一个解释器，仅仅只是一个 库 （在 iOS系统里面表现为 libobjc.A.dylib 记不太清，如有误请指正)。仅仅依靠一个库，就完成了一门编程语言的运行环境，岂不是有点令人惊讶？

并没有什么好惊讶的，严格意义上来说，Objective-C并不能算上一门真正的编程语言，Clang也是这么说的。Clang称 Objective-C只是一门 C的方言，只不过这门方言 “方”到认不出来，以至于可以称它为一门语言。从概念上说，Objective-C就是 C，各位 Cocoa Touch框架下的开发者写了这么长时间，并没有从语言层面上变得多“高级”。所以在 objc中写 C的代码无缝衔接，自然而然，objc在于 C++混编时也变得容易。

# Objective-C 的实现
这一段的标题真的有点过大，我并不能在一片文章里面说清楚 Objctive-C的实现原理，但至少能通过一些相对简单的例子让读者知道：哦，objc大概就是那样的，那这篇文章的目的就达到了。

那 objc到底大概是个什么样的实现法呢？学过编译原理的我们都知道，一门编程语言的实现需要经历编译器的前端和后端两个大的过程。objc作为一门方言，它和 C共用了后端（也就是没有后端），它仅仅经过编译器前端，被重写为了 C/C++代码，加上一个 runtime动态库，便能实现面向对象的编程了。

objc的前端是什么样的我们无从得知，但 Clang肯定是知道的。从一些 Clang的文档片段中可以得知，objc的语法推导和我想象的一般简单，比如用于描述闭包的语法推导：

```
Block_literal_expression ::= ^ block_decl compound_statement_body
block_decl ::=
block_decl ::= parameter_list
block_decl ::= type_expression
```

那么 Clang在翻译到一个 Block的时候，自然要调用 objc_create_block(xxx)之类的方法（这只是个例子，并不存在这样的方法，闭包的运行时称作 libclosure）。那么一个 Block的字面量便会被重写作一段 C代码编译，在运行时依赖函数创建一个 Block实例（结构体指针）。

objc的这种编译模式是很容易想到并且实现出来的，我的一个使用 DSL语法重写 url的 C库 [enginX](https://github.com/stephenwzl/enginx)也是这样实现的“方言”，和 objc有一点点不同的地方是为了免去编译，我直接写了一个解释器。如果你觉得我的代码看起来很糟糕也没关系，推荐一本书叫做《自制编程语言》，前桥和弥先生所著，有中文翻译版。如果你觉得这本书讲得太浅，还可以看一遍“虎书”，叫做 《现代编译原理 C 语言描述》。这两本书都非常通俗易懂，如果能看完任何一本，都对 objc 的实现理解有很大的帮助。

读到这里，我觉得你很容易就能想到 objc的 runtime.h里面陈列的 C函数都是干嘛用的了。比如说 class_createInstance(Class cls, size_t extraBytes)这个函数，它会在你写了 [NSObject new]之类的实例化方法处被调用，其他的大体上也是差不多的。其实我认为到这里，每个人都应该能知道：“哦，objc大概就是那样的啊”。因为接下去要说的一些，是关于 objc在面向对象等各种特性上实现的一些细节。

# Objective-C 的对象
我们直接来看 objc 2.0的解释吧，1.0已经停留在 iOS 2.0/ OS X 10.5了，没有讨论的意义了。既然 C 不是面向对象的，objc 又仅仅只是一门方言，那么 objc 理应是不会存在“对象”这种东西的。实际上一个 objc 的 Class 就是一个结构体的指针类型，可以看到 runtime 头文件的定义如下：

```c
struct objc_class {
    Class isa  OBJC_ISA_AVAILABILITY;
 
#if !__OBJC2__
    Class super_class                                        OBJC2_UNAVAILABLE;
    const char *name                                         OBJC2_UNAVAILABLE;
    long version                                             OBJC2_UNAVAILABLE;
    long info                                                OBJC2_UNAVAILABLE;
    long instance_size                                       OBJC2_UNAVAILABLE;
    struct objc_ivar_list *ivars                             OBJC2_UNAVAILABLE;
    struct objc_method_list **methodLists                    OBJC2_UNAVAILABLE;
    struct objc_cache *cache                                 OBJC2_UNAVAILABLE;
    struct objc_protocol_list *protocols                     OBJC2_UNAVAILABLE;
#endif
 
} OBJC2_UNAVAILABLE;
 
typedef struct objc_class * Class;
```

很明显，你在 objc 源文件里面用到的 Class 类型就是一个结构体的指针，在你的 objc 应用进程载入的时候，这些 Class就会被初始化好放在一个全局的地方，所以一个 [NSObject_instance isKindOfClass:someClass] 就是比较了这个实例所指向的 Class 的指针而已，指针的字面量就是数字，这种比较是非常容易的。当然你也可以用 objc_allocateClassPair 这种方法在运行时创建一个新的 Class，因为 Class 自己就是实例，它并不是真正的“类型”，（很可能 [因为不排除 Apple 工程师们做其他 tricky 的事]）你在编译的代码中写的那些类也是这样在进程开始时创建的。所以创建一个 Class 和创建一个一个普通的引用变量没什么两样。

这时候 objc Class 的实例对象就很尴尬了，因为它只是实例的实例，并不真的是 Class“类型”，从 [objc-runtime](https://opensource.apple.com/tarballs/objc4/) 的源代码可以看到 objc_object 是另一个结构体：

```c
struct objc_object {
private:
    isa_t isa;
 
public:
 
    // ISA() assumes this is NOT a tagged pointer object
    Class ISA();
 
    // getIsa() allows this to be a tagged pointer object
    Class getIsa();
 
    // initIsa() should be used to init the isa of new objects only.
    // If this object already has an isa, use changeIsa() for correctness.
    // initInstanceIsa(): objects with no custom RR/AWZ
    // initClassIsa(): class objects
    // initProtocolIsa(): protocol objects
    // initIsa(): other objects
    void initIsa(Class cls /*nonpointer=false*/);
    void initClassIsa(Class cls /*nonpointer=maybe*/);
    void initProtocolIsa(Class cls /*nonpointer=maybe*/);
    void initInstanceIsa(Class cls, bool hasCxxDtor);
    //很多其他函数和变量
    ....
};
```

现在能看到 objc 是如何标识自己的 Class 类型了，只是用了一个 isa_t 的结构体，这个结构体的成员变量有 Class 的变量。接下来一个叫 initClassIsa(Class cls)的方法（和initIsa 一样），它只是把传递的 Class 取了一下地址，本质就是记了一下 Class 的首地址（所以我猜想 Class 存放在全局应该没有错）。然后 changeIsa 这个函数，应该就是我们平时用的 objc_setClass 的原型了。

那么问题来了，仅通过这些东西怎么实例化一个 objc 的对象？凭“克村新鲜的空气”吗？当然不是。objc的真正对象的一个最小实现（NSObject，其实就是一个协议），它是一个 rich的 objc_class类型的，为什么这么说？看一下 object_class的 rich定义：

```c
struct objc_class : objc_object {
    Class superclass;
    const char *name;
    uint32_t version;
    uint32_t info;
    uint32_t instance_size;
    struct old_ivar_list *ivars;
    struct old_method_list **methodLists;
    Cache cache;
    struct old_protocol_list *protocols;
    // CLS_EXT only
    const uint8_t *ivar_layout;
    struct old_class_ext *ext;
//非常长，下面的方法不一一例举了
}
```

可以看到一个真正的 Object，它的前一部分的布局总是 objc_object那个结构体，存储了一个有关自身信息的结构。而 objc_object里面存的 Class信息，只是一个精简版的 objc_class（其实为了省内存）。那么在一个 objc的对象被实例化后，它就是这样一个结构体指针，它也存储了这些东西 ，那么我们就可以获取到它的 ivar，methodList，遵循的 protocol等等，这时候我觉得你肯定应该明白 runtime那些 获取实例对象的 ivar什么的方法是怎么干的了。（这个object_class的定义是稍微老一点的，新的包含了一些和 Swift交互的东西，容易造成一些干扰）

# Objective-C的方法调用
通过上面我们不难理解objc的对象如何完成继承，实例变量存储，但问题来了，OOP编程中方法调用是一个很重要的概念，每个实例都是有方法的，没有的话和面向过程有何区别？而在结构体中只能存储函数指针，如果一个对象的方法特别多，那这个对象的实际内存布局的结构体岂不是要变得特别长，不就不具有通用性了吗？

在这个问题上，objc解决得很 SmallTalk。因为目前来讲函数的调用几乎就是那么几种解决办法：静态地址，虚表映射，消息转发。前面两者都需要一个方法在编译期就要确定绝对位置或相对位置，而消息转发不用，只需要运行时查找就好了。这也是 SmallTalk的一个特点，对象之间的通信是发消息。objc的发消息函数是 objc_msgSend，再清楚不过了。而在 objc_class的结构体上，还有一个叫做 method List的指针变量，它维护着一个描述方法的链表，一个方法的描述是这样的：

```c
struct objc_method {
    SEL method_name                                          OBJC2_UNAVAILABLE;
    char *method_types                                       OBJC2_UNAVAILABLE;
    IMP method_imp                                           OBJC2_UNAVAILABLE;
}
```

SEL其实就是一个字符串了，代表着一个方法的名字，查找时也由名字匹配。method_types是方法调用所需参数的类型描述，也可以叫方法签名。在 C的调用约定中，参数个数，类型和顺序一致，一个函数才可以被正确调用。IMP其实就是一个函数指针了，指向这个函数真实存在的栈地址。objc_msgSend函数会把调用者和方法名（SEL）和参数传进来，查找并匹配正确后才会发生压栈调用。

消息转发的调用也有缺点。比如每次查链表寻找方法速度肯定慢，为此 objc_class里面有一个cache成员变量，其实就是用缓存来加速查找罢了，大家都能想到的办法。但还有问题，假如方法查不到，而我真的又在运行时发生这样的调用怎么办？NSObject的默认实现里给了一个机会就是 forwardingTargetForSelector这个方法，这几乎是面试题“必考”了。objc的设计者会尝试征询一下用户的意见：这个方法我找不到该怎么办？消息发给谁？假如你告诉了他：发给 A这个对象吧！那么他会在 forwardInvocation里面尝试找 A的这个方法，生成一个调用的抽象 NSInvocation来调用。但你告诉他的这个 A仍然可能不接收你的消息，这时候便会发生 doesNotRecognizeSelector的调用，如果你不复写，那么就会抛出一个 exception。所以说 doesNotRecognizeSelector几乎是你最后的机会来挽救这条消息，但这通常并不靠谱，因为消息很可能已经给 A了，A不认识后它会调用到 doesNotRecognizeSelector，在一些时候复写这儿并不管用。还是按照设计者的思路来吧。

# Property和 Protocol
这俩家伙并不是什么新的类型，我觉得只是 objc实现的途中顺带的两个 “util”。Property给了用户自动合成 ivar和 getter，setter方法的机会，这会在编译阶段就会帮用户完成，并且，在 objc_object的结构里还会加上一个 property_list的链表，告诉运行时用户添加了哪些 Property（这也是“必考”）。

而 Protocol就更简单了，它就是一个 objc_object指针，不过和对象不太一样的地方是，Protocol被限制了。遵循某个 Protocol的对象，在它的 superClass里面并不能找到这个 Protocol，原因是和 property_list一样，它被附加在了一个叫 protocol_list的链表里，所以不会出现在继承链上。而且 Protocol的 property并不能像正常的对象一样获得编译器的“优待”。

（好吧，关于 Property和 Protocol这一段确实是偷懒了，因为 objc的设计模式很明显了，在这儿并没有体现出什么新意，再啰嗦下去都有点烦了）

# 写在最后
objc的 Runtime代表的并不是什么 “黑魔法”或者仅仅那几个 C函数API，它是 objc的核心设计思想，或许 Apple开源它正是希望开发者不要因为这门方言搞得大家焦头烂额，又或许开源它的那一刻，Swift正好已经在进行中了。但不管怎么样，C/C++永远是静态编译语言的经典，甚至可以说，学好 C/C++，走遍天下都不怕，还会因为一门方言焦头烂额？
