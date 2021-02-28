---
title: objc的 block
date: 2017-06-11 17:21:29
tags:
---
在没有认清 Block的真实面目前，日常处理 Block相关的逻辑时可以说是战战兢兢、如履薄冰。在经过一阵迷茫和抓狂后，挖到了 Clang文档对于 Block实现的[解释](https://clang.llvm.org/docs/Block-ABI-Apple.html)。虽然相对于当前的 Block实现不是100%完整，但也可以管中窥豹了。
<!-- more -->

从 ABI层面看，Block由一个特定的内存布局（aka layout）和运行时方法组成，先看一下文档的结构描述：

```c
struct Block_literal_1 {
    void *isa; // initialized to &_NSConcreteStackBlock or &_NSConcreteGlobalBlock
    int flags;
    int reserved;
    void (*invoke)(void *, ...);
    struct Block_descriptor_1 {
    unsigned long int reserved;         // NULL
        unsigned long int size;         // sizeof(struct Block_literal_1)
        // optional helper functions
        void (*copy_helper)(void *dst, void *src);     // IFF (1<<25)
        void (*dispose_helper)(void *src);             // IFF (1<<25)
        // required ABI.2010.3.16
        const char *signature;                         // IFF (1<<30)
    } *descriptor;
    // imported variables
};
```

从这个结构体的第一个元素，我看到了一个熟悉的变量名 “isa”，和 objc class的 isa指针作用类似，表明了 Block的类型。从注释可以得知 Block的 isa指针初始化的时候可能指向的是 \_NSConcreteStackBlock 或 \_NSConcreteGlobalBlock，这表示 Block在刚创建的时候的内存类型，可能是 栈上的，也可能是全局的。

不过，Clang文档没有提到的其他的类型，从最新的 [libclosure](https://opensource.apple.com/tarballs/libclosure/)的源代码 可以得知，其实 Block的类型是有下面这几种的：

```c
void * _NSConcreteStackBlock[32] = { 0 };
void * _NSConcreteMallocBlock[32] = { 0 };
void * _NSConcreteAutoBlock[32] = { 0 };
void * _NSConcreteFinalizingBlock[32] = { 0 };
void * _NSConcreteGlobalBlock[32] = { 0 };
void * _NSConcreteWeakBlockVariable[32] = { 0 };
```
Block 的 isa最终会指向他们其中一个指针数组的首地址，这样一来，比较 Block的类型就是比较不同的指针，会比较方便一点。

不过我们常见的是 \_NSConcreteStackBlock，\_NSConcreteGlobalBlock, \_NSConcreteMallocBlock, 把 Block想象成普通的变量我们就可以理解了，因为我们所用的变量要么是 栈上的，要么是全局的，要么就是堆上的。至于其他三种类型，那是 GC所需要关心的事，暂时和我们无关，先放一边。

再看看 Block 结构里的第二个元素 flag，很明显 libclosure的大神们在这里立了一个 flag，以便生成 Block后在运行时知道一些具体的信息，从 runtime源码看到了如下定义（此 runtime为 libclosure的 runtime）

```c
enum {
    BLOCK_DEALLOCATING =      (0x0001),  // runtime
    BLOCK_REFCOUNT_MASK =     (0xfffe),  // runtime
    BLOCK_NEEDS_FREE =        (1 << 24), // runtime
    BLOCK_HAS_COPY_DISPOSE =  (1 << 25), // compiler
    BLOCK_HAS_CTOR =          (1 << 26), // compiler: helpers have C++ code
    BLOCK_IS_GC =             (1 << 27), // runtime
    BLOCK_IS_GLOBAL =         (1 << 28), // compiler
    BLOCK_USE_STRET =         (1 << 29), // compiler: undefined if !BLOCK_HAS_SIGNATURE
    BLOCK_HAS_SIGNATURE  =    (1 << 30), // compiler
    BLOCK_HAS_EXTENDED_LAYOUT=(1 << 31)  // compiler
};
```

从这些 enum的注释可以得知，一些是运行时用的，还有一些是编译时期就会产生的，在讨论 Block如何运行（或者说讨论编程语言的函数运行）之前，就先不看这些运行时选项了。那么很明显看到这个 flag会标记 Block是否 Global的，是否需要 Copy和 Dispose，是否带返回值的调用（STRET），是否有函数签名（SIGNATURE），以及是否有一些额外的“布局”（EXTENDED_LAYOUT），比如说传入的参数，这个稍后讨论。

“先在本文中立个 flag，以后要专门开一篇文章来描述函数的运行”

不过文档里还提到，关于上文中这个 Block flag的 enum中的一些值其实是历史遗留的，有点冗余的意味，但又不能删，因为 C语言的 ABI已经稳定若干年了。比如 BLOCK_USE_STRET 和 BLOCK_HAS_SIGNATURE总是成对出现的，而大部分时候进行调用的时候，调用者需要知道的只是一个函数签名，至于返回值，从标准的 Call Interface很容易就能获得，类型在函数签名甚至调用者自己也是十分清楚的。

说到这儿，我想有必要描述一下函数签名的作用。当进程被启动时，也就是 main函数执行时显而易见是有参数传进来的，但参数是不定长的。同样的道理，进程内的函数调用本质上的参数传递也是不定长的，调用者只有知道目标函数的函数签名，才能确定传多少参数，每个参数类型是什么。所以这同样解释了 Global类型的 Block为啥没有方法签名。因为它无参数，无返回值（或者说返回值是 void），调用者只需要知道函数执行的首地址就可以了。突然提到这个 Global类型的 Block，我们稍后再慢慢来看一下 Block的具体实现。

先继续 flag后面的变量，叫 reserved, 很显然这是一个保留字，用作什么也不清楚，不过从编译期的传值来看，都是0，并没有实际用它的地方，也没有注释说将来可能会用作什么，所以在遇到保留字的时候，跳过阅读就好了。不过我的猜想是 flag字段是通过左移右移计算的，所以很可能是怕内存溢出影响到实际的函数指针指向，那样就会影响 Block的实际执行，麻烦就大了。

我们接着看后一个字段 invoke，顾名思义即实际的函数指针，这没什么好说的。再接下来一段结构体把一些关于这个 Block的一些描述或者其他用得到的 “context”给记录下来。开头便是一个保留字，然后是 Block的 size，接下来是有可能出现的两个函数指针，copy 和 dispose，帮助管理 Block变量的引用计数，他们俩仅有可能在 flag出现 BLOCK_HAS_COPY_DISPOSE 即 (flag & 1<<25) 时会存在。然后是 Block的方法签名，也仅当 BLOCK_HAS_SIGNATURE时才会存在。最后一行注释告诉我们，那块内存会存在一些捕获的变量。

刚才这段其实从字面上看会让人糊里糊涂，好在 Clang文档给我们提供了一个例子，关于在 32位系统下改写 objc为 C++的一个实现，同样地，我们自己也可以使用 Clang工具改写，具体的命令是：

```
clang --rewrite-objc [input_file]
```
我这边直接拿官方文档的例子，比较清晰一些：

```c
//before rewrite
^ { printf("hello world\n"); }

//after rewrite
struct __block_literal_1 {
    void *isa;
    int flags;
    int reserved;
    void (*invoke)(struct __block_literal_1 *);
    struct __block_descriptor_1 *descriptor;
};
//注释 1
void __block_invoke_1(struct __block_literal_1 *_block) {
    printf("hello world\n");
}

static struct __block_descriptor_1 {
    unsigned long int reserved;
    unsigned long int Block_size;
} __block_descriptor_1 = { 0, sizeof(struct __block_literal_1), __block_invoke_1 };

//注释 2
struct __block_literal_1 _block_literal = {
     &_NSConcreteStackBlock,
     (1<<29), <uninitialized>,
     __block_invoke_1,
     &__block_descriptor_1
};
```

仔细看这一段重写后的代码，重写前的 Block没有返回值，没有传参，然后 Block的实际执行体被重写成了一个 C函数，传参为 Block的结构自身（详见注释1）。然后实际的 Block变量被声明为 _NSConcreteStackBlock，flag设置为 1<<29，invoke指针指向了执行函数，descriptor指针指向了前一步生成的结构体。（1<<29 flag 在运行时通常被忽略，所以这样赋值并没有什么特殊意义，避免 uninitialized这样设置 flag也可以理解）。

Clang文档中提到了关于 Block的 isa类型的问题，这里也表明了他们在设计 Block时候的初衷：当一个 Block的字面量声明在全局或声明为 static的时候，isa是指向 \_NSConcreteGlobalBlock的。除此之外，Block在初始化时的 isa都是指向 \_NSConcreteStackBlock的，缺点就非常明显了，只存在于栈帧中，内存清除后肯定就无法引用到了，所以 libclosure中有 Block_Copy()方法将 Block拷贝到堆上，这时候 isa便指向了 _NSConcreteMallocBlock。文档虽然说了只有有限的一两种情况下 Block是全局的，但显然在 ARC时代 LLVM的实现并未如此，如果Block定义在局部（函数体内部）且未捕获自动变量，那么这些 Block也是全局类型的。那么捕获了自动变量呢？显然就会变成堆上的 Block，栈上的 Block在 ARC下貌似是不存在的（不过从最新的 LLVM文档发现，ARC下，Block初始化时是栈类型的，但如果引用或返回，会移动到堆上，所以“栈上的 Block在 ARC下不存在勉强不算错”）。

我们接下来看一下 Block捕获变量的方式是如何影响到 ARC时代的 Block内存管理的。

Clang文档提到，捕获没有标记 __block的变量时，直接导入了该变量的 const Copy，比如文档中的这个例子：

```c
int x = 10;
void (^vv)(void) = ^{ printf("x is %d\n", x); }
x = 11;
vv();
```

会被 Clang重写为：

```c
struct __block_literal_2 {
    void *isa;
    int flags;
    int reserved;
    void (*invoke)(struct __block_literal_2 *);
    struct __block_descriptor_2 *descriptor;
    const int x;
};

void __block_invoke_2(struct __block_literal_2 *_block) {
    printf("x is %d\n", _block->x);
}

static struct __block_descriptor_2 {
    unsigned long int reserved;
    unsigned long int Block_size;
} __block_descriptor_2 = { 0, sizeof(struct __block_literal_2) };
```

```c
struct __block_literal_2 __block_literal_2 = {
      &_NSConcreteStackBlock,
      (1<<29), <uninitialized>,
      __block_invoke_2,
      &__block_descriptor_2,
      x
 };
```

所以在 Block执行的时候并不能更改这些变量，这也解释了 Block为什么不需要 discriptor里面的 helper functions 来帮助管理变量的引用计数。总结来说，纯量类型、结构体、联合体以及函数指针，都只会传递一份 const copy给 Block引用。

但当 const copy的对象是另一个 Block时，情况就稍微麻烦一点，首先 Block的实际存在形式是一个结构体指针，前面说过纯量类型会直接传递一个 const copy过的值进来，对于 Block，这个 const结构体指针传进来时在 runtime并不能直接用，因为你并不能保证这个指针指向的 Block能正确执行，万一这个 Block在栈上早就释放了呢？所以这时候 Block的 descriptor就会产生 copy_helper和 dispose_helper，把传递进来的指针对应的内存 copy一份到堆上，保持正确的引用，并且在使用结束后正确释放内存。这一段逻辑在文档中有对应的 [rewrite代码](https://clang.llvm.org/docs/Block-ABI-Apple.html#imported-const-copy-of-block-reference) 。同样地对于 objc的对象类型，和 Block类型一样被 Block捕获时会被 copy一次，为了让 C类型认识除了 Block被 copy还有 Object也会被 copy，在执行 copy helper function的时候还会为 Object类型填上一个类型叫 BLOCK_FIELD_IS_OBJECT，以便后续 context执行时不要把它误认为一个 Block。

如上所述的话其实 Block捕获变量还算简单，但我们忘了一点，大部分时候我们把 Block作为回调使用，需要去改变外部变量的值，一个 const copy根本满足不了需求，而且我们平时写的时候也知道，需要外部变量被改变，那么需要加上 __block这样一个标记，那么这个标记是怎样的原理呢？文档同样也有介绍。

__block标记的变量会被改造得面目全非，在 libclosure和 Clang文档里都有介绍变量被改写后的内存布局：

```c
struct _block_byref_foo {
    void *isa;
    struct Block_byref *forwarding;
    int flags;   //refcount;
    int size;
    // helper functions called via Block_copy() and Block_release()
    void (*byref_keep)(void  *dst, void *src);
    void (*byref_dispose)(void *);
    typeof(marked_variable) marked_variable;
};
```

可以看到一个变量会被重写成这个鬼样子，关于这些变量的初始化顺序是这样的：

forwarding指向自己这个结构体的开始地址
size变量被设置为结构体的大小
flags被设置为0或 1<<25（如果需要 helper_functions的话）
helper_functions初始化 （if needed）
marked_variable设置为捕获变量的值
isa设置为 NULL
Block内部在调用这个变量时就是这样的

variable->forwarding.marked_variable = someValue;

和捕获非 __block标记变量类似，此处也有关于 Block类型和 Object类型的困扰。比如一个 Block类型：

```c
__block void (voidBlock)(void) = blockA;
voidBlock = blockB;

//rewrited
struct _block_byref_voidBlock {
    void *isa;
    struct _block_byref_voidBlock *forwarding;
    int flags;   //refcount;
    int size;
    void (*byref_keep)(struct _block_byref_voidBlock *dst, struct _block_byref_voidBlock *src);
    void (*byref_dispose)(struct _block_byref_voidBlock *);
    void (^captured_voidBlock)(void);
};

void _block_byref_keep_helper(struct _block_byref_voidBlock *dst, struct _block_byref_voidBlock *src) {
    //_Block_copy_assign(&dst->captured_voidBlock, src->captured_voidBlock, 0);
    _Block_object_assign(&dst->captured_voidBlock, src->captured_voidBlock, BLOCK_FIELD_IS_BLOCK | BLOCK_BYREF_CALLER);
}

void _block_byref_dispose_helper(struct _block_byref_voidBlock *param) {
    //_Block_destroy(param->captured_voidBlock, 0);
    _Block_object_dispose(param->captured_voidBlock, BLOCK_FIELD_IS_BLOCK | BLOCK_BYREF_CALLER)}

//usage
struct _block_byref_voidBlock voidBlock = {( .forwarding=&voidBlock, .flags=(1<<25), .size=sizeof(struct _block_byref_voidBlock *),
    .byref_keep=_block_byref_keep_helper, .byref_dispose=_block_byref_dispose_helper,
    .captured_voidBlock=blockA )};

voidBlock.forwarding->captured_voidBlock = blockB;
```

很显然，Block作为一种引用类型，在标记为 __block后，会被 copy到堆上，由捕获的 Block管理内存，可想而知，Object类型也是基本一致的。

看了这么多，很容易总结出 Block对变量的捕获规则：

1. 最简单的是 全局变量和 static变量，随意引用和修改，在任何地方都是如此。

2. 上下文变量，也就是栈变量，const copy，不可修改。前面也提到在 ARC下，为了保持 Block的生命周期，便于自动管理，Block会被 copy到堆上（实际上 Block被概念化为一个引用类型或对象）。但此时 Block在捕获引用类型就会产生强引用
3. \__block标记的上下文变量，会从栈变量挪到堆，以保证完整的生命周期访问。除了要担心循环引用外，剩下的事都可以交给 ARC打扫。

ARC为我们隐藏了太多的细节，其实我们大可不必关心 Block的具体实现。但如果担心的话，跑过来看一遍，在写相关代码时可能会更有信心一点。

