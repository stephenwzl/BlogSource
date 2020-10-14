---
title: block of objc
date: 2017-06-11 17:21:29
tags:
---
Without knowing the true face of Block, it can be said that the daily processing of Block-related logic can be said to be cautious and walking on thin ice. After a while of confusion and madness, I dug into the Clang document's [explanation] of Block implementation (https://clang.llvm.org/docs/Block-ABI-Apple.html). Although it is not 100% complete compared to the current Block implementation, it can still be seen.
<!-- more -->

From the ABI level, Block is composed of a specific memory layout (aka layout) and runtime methods. Let's first look at the structure description of the document:

```c
struct Block_literal_1 {
    void *isa; // initialized to &_NSConcreteStackBlock or &_NSConcreteGlobalBlock
    int flags;
    int reserved;
    void (*invoke)(void *, ...);
    struct Block_descriptor_1 {
    unsigned long int reserved; // NULL
        unsigned long int size; // sizeof(struct Block_literal_1)
        // optional helper functions
        void (*copy_helper)(void *dst, void *src); // IFF (1<<25)
        void (*dispose_helper)(void *src); // IFF (1<<25)
        // required ABI.2010.3.16
        const char *signature; // IFF (1<<30)
    } *descriptor;
    // imported variables
};
```

From the first element of this structure, I saw a familiar variable name "isa", which is similar to the isa pointer of the objc class, indicating the type of Block. From the comments, it can be known that the isa pointer of the Block may point to \_NSConcreteStackBlock or \_NSConcreteGlobalBlock when it is initialized. This indicates that the memory type of the block when it is first created may be on the stack or global.

However, there are other types that are not mentioned in the Clang document. From the source code of the latest [libclosure](https://opensource.apple.com/tarballs/libclosure/), you can know that there are actually the following types of Block Species:

```c
void * _NSConcreteStackBlock[32] = {0 };
void * _NSConcreteMallocBlock[32] = {0 };
void * _NSConcreteAutoBlock[32] = {0 };
void * _NSConcreteFinalizingBlock[32] = {0 };
void * _NSConcreteGlobalBlock[32] = {0 };
void * _NSConcreteWeakBlockVariable[32] = {0 };
```
Block's isa will eventually point to the first address of one of their pointer arrays. In this way, comparing Block types is to compare different pointers, which is more convenient.

But our common ones are \_NSConcreteStackBlock, \_NSConcreteGlobalBlock, \_NSConcreteMallocBlock, we can understand Block as a normal variable, because the variables we use are either on the stack, global, or on the heap. As for the other three types, that is something that GC needs to care about. It has nothing to do with us for the time being. Let it go.

Looking at the second element flag in the Block structure, it is obvious that the libclosure gods have set up a flag here so that they can know some specific information at runtime after the block is generated. The following definition can be seen from the runtime source code (this runtime is runtime of libclosure)

```c
enum {
    BLOCK_DEALLOCATING = (0x0001), // runtime
    BLOCK_REFCOUNT_MASK = (0xfffe), // runtime
    BLOCK_NEEDS_FREE = (1 << 24), // runtime
    BLOCK_HAS_COPY_DISPOSE = (1 << 25), // compiler
    BLOCK_HAS_CTOR = (1 << 26), // compiler: helpers have C++ code
    BLOCK_IS_GC = (1 << 27), // runtime
    BLOCK_IS_GLOBAL = (1 << 28), // compiler
    BLOCK_USE_STRET = (1 << 29), // compiler: undefined if !BLOCK_HAS_SIGNATURE
    BLOCK_HAS_SIGNATURE = (1 << 30), // compiler
    BLOCK_HAS_EXTENDED_LAYOUT=(1 << 31) // compiler
};
```

From the comments of these enums, we can know that some are used at runtime, and some are generated at compile time. Before discussing how the Block works (or discussing the function operation of the programming language), we will not look at these operations. Time to choose. Then it is obvious that this flag will mark whether the Block is Global, whether Copy and Dispose are required, whether the call with return value (STRET), whether there is a function signature (SIGNATURE), and whether there is some additional "layout" (EXTENDED_LAYOUT) , Such as the incoming parameters, this will be discussed later.

"First set up a flag in this article, and then I will open a special article to describe the operation of the function"

However, it is also mentioned in the document that some of the values ​​in the enum of the Block flag mentioned above are actually left over from history and are a bit redundant, but they cannot be deleted because the ABI of the C language has been stable for several years. For example, BLOCK_USE_STRET and BLOCK_HAS_SIGNATURE always appear in pairs, and most of the time when making a call, all the caller needs to know is a function signature. As for the return value, it is easy to obtain from the standard Call Interface. The type is in the function signature or even The caller himself is very clear.

Having said that, I think it is necessary to describe the role of function signatures. When the process is started, that is, when the main function is executed, it is obvious that there are parameters passed in, but the parameters are of variable length. In the same way, the parameter passing of the function call in the process is essentially of variable length. The caller only knows the function signature of the target function to determine how many parameters to pass and what each parameter type is. So this also explains why the Global type of Block has no method signature. Because it has no parameters and no return value (or the return value is void), the caller only needs to know the first address of the function execution. Suddenly mentioned this Global type of Block, we will slowly look at the specific implementation of Block later.


Xiān jìxù flag hòumiàn de biànliàng, jiào reserved, hěn xiǎnrán zhè shì yīgè bǎoliú zì, yòng zuò shénme yě bù qīngchǔ, bùguò cóng biānyì qí de chuán zhí lái kàn, dōu shì 0, bìng méiyǒu shíjì yòng tā dì dìfāng, yě méiyǒu zhùshì shuō jiānglái kěnéng huì yòng zuò shénme, suǒyǐ zài yù dào bǎoliú zì de shíhòu, tiàoguò yuèdú jiù hǎole. Bùguò wǒ de cāixiǎng shì flag zìduàn shì tōngguò zuǒ yí yòu yí jìsuàn de, suǒyǐ hěn kěnéng shì pà nèicún yìchū yǐngxiǎng dào shíjì de hánshù zhǐzhēn zhǐxiàng, nàyàng jiù huì yǐngxiǎng Block de shíjì zhíxíng, máfan jiù dàle.

Wǒmen jiēzhe kàn hòu yīgè zìduàn invoke, gùmíngsīyì jí shíjì de hánshù zhǐzhēn, zhè méishénme hǎoshuō de. Zài jiē xiàlái yīduàn jiégòu tǐ bǎ yīxiē guānyú zhège Block de yīxiē miáoshù huòzhě qítā yòng dédào de “context” gěi jìlù xiàlái. Kāitóu biàn shì yīgè bǎoliú zì, ránhòu shì Block de size, jiē xiàlái shì yǒu kěnéng chūxiàn de liǎng gè hánshù zhǐzhēn,copy hé dispose, bāngzhù guǎnlǐ Block biànliàng de yǐnyòng jìshù, tāmen liǎ jǐn yǒu kěnéng zài flag chūxiàn BLOCK_HAS_COPY_DISPOSE jí (flag& 1<<25) shí huì cúnzài. Ránhòu shì Block de fāngfǎ qiānmíng, yě jǐn dāng BLOCK_HAS_SIGNATURE shí cái huì cúnzài. Zuìhòu yīxíng zhùshì gàosù wǒmen, nà kuài nèicún huì cúnzài yīxiē bǔhuò de biànliàng.

Gāngcái zhè duàn qíshí cóng zìmiàn shàng kàn huì ràng rén hú li hútú, hǎo zài Clang wéndàng gěi wǒmen tígōngle yīgè lìzi, guānyú zài 32 wèi xìtǒng xià gǎixiě objc wèi C++de yīgè shíxiàn, tóngyàng de, wǒmen zìjǐ yě kěyǐ shǐyòng Clang gōngjù gǎixiě, jùtǐ de mìnglìng shì:

```
Clang --rewrite-objc [input_file]
```
wǒ zhè biān zhíjiē ná guānfāng wéndàng de lìzi, bǐjiào qīngxī yīxiē:

```C
//before rewrite
^ {printf("hello world\n"); }

//after rewrite
struct __block_literal_1 {
    void*isa;
    int flags;
    int reserved;
    void (*invoke)(struct __block_literal_1*);
    struct __block_descriptor_1*descriptor;
};
//zhùshì 1
void __block_invoke_1(struct __block_literal_1*_block) {
    printf("hello world\n");
}

static struct __block_descriptor_1 {
    unsigned long int reserved;
    unsigned long int Block_size;
} __block_descriptor_1 = {0, sizeof(struct __block_literal_1), __block_invoke_1};

//zhùshì 2
struct __block_literal_1 _block_literal = {
     &_NSConcreteStackBlock,
     (1<<29), <uninitialized>,
     __block_invoke_1,
     &__block_descriptor_1
};
```

zǐxì kàn zhè yīduàn chóng xiě hòu de dàimǎ, chóng xiě qián de Block méiyǒu fǎnhuí zhí, méiyǒu chuán cān, ránhòu Block de shíjì zhíxíng tǐ bèi chóng xiěchéngle yīgè C hánshù, chuán cān wèi Block de jiégòu zìshēn (xiáng jiàn zhùshì 1). Ránhòu shíjì de Block biànliàng bèi shēngmíng wèi _NSConcreteStackBlock,flag shèzhì wèi 1<<29,invoke zhǐzhēn zhǐxiàngle zhíxíng hánshù,descriptor zhǐzhēn zhǐxiàngle qián yībù shēngchéng de jiégòu tǐ.(1<<29 Flag zài yùnxíng shí tōngcháng bèi hūlüè, suǒyǐ zhèyàng fùzhí bìng méiyǒu shé me tèshū yìyì, bìmiǎn uninitialized zhèyàng shèzhì flag yě kěyǐ lǐjiě).
展开
1591/5000
First continue with the variable behind the flag, called reserved. Obviously this is a reserved word and it is not clear what it is used for, but from the value passed at compile time, it is all 0. There is no place to actually use it, and there is no comment. Say what you might use in the future, so when you encounter reserved words, just skip reading. But my guess is that the flag field is calculated by shifting left and right, so it is likely that the memory overflow will affect the actual function pointer, which will affect the actual execution of the block, and the trouble will be big.

Let's look at the latter field, invoke, which is the actual function pointer as the name suggests. There is nothing to say about this. The next section of the structure records some descriptions of this Block or other "context" that are used. At the beginning is a reserved word, then the size of Block, and then there are two function pointers that may appear, copy and dispose, to help manage the reference count of the Block variable. The two of them only have the possibility of appearing BLOCK_HAS_COPY_DISPOSE in the flag (flag & 1<<25) will exist. Then there is the method signature of Block, which only exists when BLOCK_HAS_SIGNATURE. The last line of comments tells us that there will be some captured variables in that memory.

The paragraph just now is actually confusing literally. Fortunately, the Clang document provides us with an example about rewriting objc as an implementation of C++ under a 32-bit system. Similarly, we can also use Clang tools ourselves. Rewrite, the specific command is:

```
clang --rewrite-objc [input_file]
```
Let me take the official document example directly, which is clearer:

```c
//before rewrite
^ {printf("hello world\n");}

//after rewrite
struct __block_literal_1 {
    void *isa;
    int flags;
    int reserved;
    void (*invoke)(struct __block_literal_1 *);
    struct __block_descriptor_1 *descriptor;
};
//Comment 1
void __block_invoke_1(struct __block_literal_1 *_block) {
    printf("hello world\n");
}

static struct __block_descriptor_1 {
    unsigned long int reserved;
    unsigned long int Block_size;
} __block_descriptor_1 = {0, sizeof(struct __block_literal_1), __block_invoke_1 };

//Note 2
struct __block_literal_1 _block_literal = {
     &_NSConcreteStackBlock,
     (1<<29), <uninitialized>,
     __block_invoke_1,
     &__block_descriptor_1
};
```

Take a closer look at this rewritten code. The Block before rewriting has no return value and no parameters. Then the actual execution body of the Block is rewritten into a C function, and the parameters are passed to the structure of the Block itself (see Note 1 for details) . Then the actual Block variable is declared as _NSConcreteStackBlock, the flag is set to 1<<29, the invoke pointer points to the execution function, and the descriptor pointer points to the structure generated in the previous step. (1<<29 flag is usually ignored at runtime, so this assignment has no special meaning, it is understandable to avoid uninitialized so setting the flag).

The Clang document mentions the issue of the isa type of Block. This also shows their original intention when designing the Block: when a Block's literal declaration is globally or declared as static, isa points to \_NSConcreteGlobalBlock. In addition, the isa of Block during initialization points to \_NSConcreteStackBlock. The shortcomings are very obvious. They only exist in the stack frame. After the memory is cleared, it cannot be referenced. Therefore, there is a Block_Copy() method in libclosure. Copy to the heap, at this time isa points to _NSConcreteMallocBlock. Although the document says that there are only a limited number of cases where Block is global, it is obvious that the implementation of LLVM in the ARC era is not like this. If Block is defined locally (inside the function body) and automatic variables are not captured, then these Blocks are also global Type of. So what about capturing automatic variables? Obviously it will become a block on the heap. The block on the stack does not seem to exist under ARC (but from the latest LLVM document, it is found that under ARC, the block is initialized with the stack type, but if it is referenced or returned, it will move To the heap, so "the Block on the stack does not exist under ARC, and it is not wrong").

Let's take a look at how the way Block captures variables affects Block memory management in the ARC era.

The Clang documentation mentions that when capturing variables that are not marked with __block, the const Copy of the variable is directly imported, such as this example in the documentation:

```c
int x = 10;
void (^vv)(void) = ^{ printf("x is %d\n", x);}
x = 11;
vv();
```

Would be rewritten by Clang as:

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
} __block_descriptor_2 = {0, sizeof(struct __block_literal_2) };
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

Therefore, these variables cannot be changed when the block is executed, which explains why the block does not need the helper functions in the discriptor to help manage the reference count of the variables. In summary, scalar types, structures, unions, and function pointers will only pass a const copy to the Block reference.

But when the object of const copy is another Block, the situation is a bit more troublesome. First, the actual form of Block is a structure pointer. As mentioned earlier, the scalar type will directly pass a const copied value. For Block, When this const structure pointer is passed in, it cannot be used directly at runtime, because you cannot guarantee that the block pointed to by this pointer can be executed correctly. What if the block is released on the stack long ago? So at this time, the block descriptor will generate copy_helper and dispose_helper, copy the memory corresponding to the passed pointer to the heap, keep the correct reference, and correctly release the memory after the end of use. This piece of logic has a corresponding [rewrite code](https://clang.llvm.org/docs/Block-ABI-Apple.html#imported-const-copy-of-block-reference) in the document. Similarly, for the object type of objc, like the Block type, it will be copied once when it is captured by the Block. In order to let the C type recognize that in addition to the Block being copied, the Object will also be copied. It will also be the Object type when executing the copy helper function. Fill in a type called BLOCK_FIELD_IS_OBJECT so that it will not be mistaken for a Block when the subsequent context is executed.

As mentioned above, in fact, Block capturing variables is fairly simple, but we forgot one thing. Most of the time we use Block as callbacks and we need to change the value of external variables. A const copy simply cannot meet the needs, and we usually write It is also known that if external variables need to be changed, then a mark such as __block needs to be added. Then what is the principle of this mark? The document also has an introduction.

Variables marked by __block will be transformed beyond recognition. Both libclosure and Clang documents have introduced the memory layout of variables after being rewritten:

```c
struct _block_byref_foo {
    void *isa;
    struct Block_byref *forwarding;
    int flags; //refcount;
    int size;
    // helper functions called via Block_copy() and Block_release()
    void (*byref_keep)(void *dst, void *src);
    void (*byref_dispose)(void *);
    typeof(marked_variable) marked_variable;
};
```

You can see that a variable will be rewritten to look like this ghost. The initialization sequence of these variables is as follows:

forwarding points to the start address of this structure
The size variable is set to the size of the structure
flags is set to 0 or 1<<25 (if helper_functions is needed)
helper_functions initialization (if needed)
marked_variable is set to the value of the captured variable
isa is set to NULL
This is how the variable is called inside the block

variable->forwarding.marked_variable = someValue;

Similar to capturing non-__block tag variables, there are also problems with Block and Object types. For example, a Block type:

```c
__block void (voidBlock)(void) = blockA;
voidBlock = blockB;

//rewrited
struct _block_byref_voidBlock {
    void *isa;
    struct _block_byref_voidBlock *forwarding;
    int flags; //refcount;
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

Obviously, Block, as a reference type, will be copied to the heap after being marked as __block, and the captured Block will manage the memory. As you can imagine, the Object type is basically the same.

After reading so much, it is easy to summarize Block's capture rules for variables:

1. The simplest are global variables and static variables, which can be referenced and modified at will, which is the case everywhere.

2. Context variables, that is, stack variables, const copy, cannot be modified. As mentioned earlier, under ARC, in order to maintain the life cycle of the block and facilitate automatic management, the block will be copied to the heap (in fact, the block is conceptualized as a reference type or object). But at this time, Block will generate strong references when capturing reference types
3. The context variables marked with \__block will be moved from the stack variable to the heap to ensure complete life cycle access. Apart from worrying about circular references, ARC can clean up the rest.

ARC hides too many details for us. In fact, we don't need to care about the specific implementation of Block. But if you are worried, just run over and read it again, and you may be more confident when writing relevant code.

