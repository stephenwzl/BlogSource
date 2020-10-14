---
title: ARC of objc
date: 2017-07-04 10:20:05
tags:
---

The full name of ARC has been known by various Internet blogs Amway for so many years (Auto Reference Counting), and the Chinese name is Auto Reference Counting. This is a relatively simple garbage collection mechanism. What is garbage? That is, anything that cannot be reached by variables or pointers in the heap allocation process is called garbage. This part of the recovery and redistribution is called garbage collection.

Today, I will discuss the implementation of reference counting garbage collection in objc (I hope to fully understand the implementation of ARC can be bypassed through a wild road article).

<!--more-->
# How is garbage generated?
We assume that the layout of an object in memory is a structure like this:

```c
struct SomeObject {
  char *firstName;
  char *lastName;
}
```

What it really does during initialization is this:

```c
typedef struct SomeObject * SomeObjectRef;
SomeObjectRef obj = (SomeObjectRef)malloc(sizeof(struct SomeObject);
obj->firstName = (char *)malloc(10);
obj->lastName = (char *)malloc(15);
```

Remember that a pointer can always only point to the first address of a certain memory structure, so what it can mark is only a memory of a type and size, and for the fixed-length memory you apply for from malloc, only the first address is always marked. When popping the stack, the pointer (stack variable) you use to mark the heap address of the structure can undoubtedly return the space it occupies with the popping stack, and those allocated on the heap will be forgotten by anyone. Do not know where they "live", this piece of memory is not reusable, so a "leak" occurs. To solve these leaks, write this code before popping the stack:

```c
free(obj->firstName);
free(obj->lastName);
free(obj);
```

Sometimes in order to avoid wild pointers, an obj = NULL;

Obviously, a total of 7 lines of code were written, and half of them were used to manage memory. I have to say that this is a very cumbersome thing. Moreover, the structure of an objc object is much more complicated than the above thing. If you follow this method, it will definitely cause more than half of the 10 lines of code to be "free", which is simply a headache.

# objc reference type and reference count

Let's calm down from the context just now. Recall the atypical reference type Block we encountered earlier. We have explained before how it changes from a literal to a structure and then to a stack variable, and finally becomes a heap variable in the process of referencing or returning. It has a function pointer called Block_copy. This function moves a Block pointer to the heap and increases the reference count of the Block by 1. If the Block is already on the heap, it directly increases the reference count by 1. Correspondingly, there is also a function pointer called Block_release, which is responsible for decrementing the reference count of the block by 1. When the reference count reaches 0, the block will be destroyed from the heap.

Block is an atypical reference type, but it is almost the same as other reference types in objc. It depends on reference counting to determine whether an object should be destroyed. Then we think about the basic structure of the objc object, as we have mentioned in the previous article, then the "Block_release" method of an objc object can be correspondingly abstracted into the [NSObject release] method.

Well now, objc's reference types (also called objective-c pointer types) have methods for managing reference counting and destruction. This formed the MRC (Manual Reference Counting) that was used years ago. I have to say that reference counting is a very simple way to manage reference type memory. This does not require you to allocate a memory specifically for indexing objects that need to be released like mark-sweep, and you don’t need to worry about stopping everything when starting GC. The program, or worry that because the recursive algorithm is not good enough, the stack is too deep to cause the stack to burst. Because the object managed by reference counting only performs simple addition and subtraction to calculate the number of times it is referenced, and when the number of times is 0, it is called to destroy, only one stack frame is needed, and the existing program does not need to stop.

# From MRC to ARC

Since MRC is so good, why do we need ARC?

Obviously, all users are not from the Assembly and C era. It is likely that their teacher taught a programming language with simple syntax and complete GC like Java or JavaScript. Because not everyone has a background in computer science, it is likely to have a background in software engineering management or other aspects, and pay more attention to engineering and efficiency.

Indeed, in terms of engineering and efficiency, the programming language of the C Family is indeed a drag. If it is not for Xcode to help you manage the project structure, you will find that writing a Make file is enough to give you a headache. Going deeper, when you write an application with a very complex logic, you have to think about how to manage the memory. The code written like this is almost impossible to work with other people. You can look back and see how the getter and setter of a UIViewController property in the MRC era were written.

Now that I was fed up with the endless retain/release of MRC, ARC also came into being. The essence of ARC is not a product of the language level, but the memory management code originally written by you is done by the compiler for you. The front end of the compiler will correctly guess where retain/release should be used and insert these codes for you, thus eliminating the need for repetitive work over and over again in MRC.

# ARC related syntax

Obviously, sometimes the compiler cannot help you insert the correct retain/release just by guessing, so ARC still has some syntax. Most obviously, when you declare a property, you have keywords that modify the property: weak, assign, strong, copy, unsafe_unretained, retain. Simply put, they represent the processing method when the property encounters that piece of object memory. There are also some markup syntax, such as: __weak, __unsafe_unretained, etc. I think everyone knows it, so there is no need to be verbose.

However, there is another memory management problem, which is the ownership of memory when an objc object is turned to C. This is a very dangerous question. C has no reference counting. When an objc object is converted, who should manage the memory? If this problem is not clearly divided, it is a very dangerous thing. So there are several bridge syntaxes for objc: \_\_bridge, \_\_bridge_retained, \_bridge\_transfer. This is a very interesting topic. \_\_bridge means that only the pointer of the objc object is obtained, and the original reference count is whatever it is. Then users who have obtained the C pointer should be careful not to use this pointer when the object may have been destroyed. \_\_bridge\_retained means that while obtaining the objc object pointer, the reference count of the object is increased by 1, which means that the user should call release at the same time to decrement the reference count by 1 after the end of use, otherwise the reference count of this object is likely to be forever It cannot be returned to zero, but this at least guarantees that the object will not be destroyed when the user gets the pointer. \_\_bridge\_transfer is even more interesting. When the objc object is converted in this way, the compiler will not insert ARC code for this object in the subsequent code. That is to say, the reference count is completely handed over to the person who obtains the C pointer reference, and the management right is "handed over".

ARC is very "smart" through the above grammar to tell users what attention to use.

# Automatic release pool

In the previous article, we said that when [NSObject release] is called, the reference count is 0, and the object will be released. That's right, you will never use it again after release. But sometimes, you need some delayed release operations. For example, you weakly reference a newly generated object. You can't let this object be released when the scope ends, right? What's the point of that? But you can't force him to quote him for some reason, what should you do? At present, almost all object generation of ARC uses a delayed release mechanism called autoreleasepool.

What is this? Literally, it is called an automatic release pool.

What is it for? It is used to release the automatically managed memory objects added to the auto-release pool at an appropriate time. (This kind of official language is just farting)

In iOS App, you can clearly see from the main function that the entire AppDelegate will be in an automatic release pool. In subsequent operations, many objects will be added to the closest automatic release pool after initialization. And the interesting point is that every NSThread will have a default auto-release pool, so auto-release objects generated in most threads will be added to this release pool. And we know that NSThread will execute a RunLoop loop, and the automatic release pool is actually released at the moment RunLoop is about to exit. This cleverly avoids the problem of the program being stopped when the garbage collection mechanism is working: there is no task. What's wrong with running, recycling garbage?

What kind of thing is the automatic release pool? In essence, it is a table structure. Every object added to the auto-release pool will be recorded by it (a pointer). Before the auto-release object is recorded, this table will also add a "sentinel" object, which is paging. A very useful simple operation in managing storage. Because when the automatic release pool performs the release operation, it is necessary to avoid areas that the operation does not need to touch. It is more useful when the automatic release pool is nested. Each automatic release pool is only responsible for releasing to its own "sentinel".

Although the automatic release pool is a very simple thing in concept, the specific implementation details are a bit complicated, and we don't need to explore too much. Because it's useless if you want to understand its principle, it's good to be able to write objc correctly. If you want to build your own GC, why bother to learn something so backward.

# Written at the end

If you can see my nonsense here, it also means that you are a person who is out of low-level interest (because you are so boring you are still watching). I am very curious about what evil did the iPhone OS cause to be served by the "big punishment" of objc (if you know, please tell me). Isn't it good to write C++?

And many Frameworks, such as CFNetwork, libclosure, JavascriptCore, etc., are implemented in C++, but in the end they wrap a layer for objc, isn't it driving backwards?

Fortunately, Swift, the language used to replace C++, represents the future.