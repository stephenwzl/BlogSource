---
title: Runtime of objc
date: 2017-06-29 19:24:04
tags:
---

<img src="/images/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7-2017-06-29-%E4%B8%8B%E5%8D% 881.42.46.png" style="max-width: 350px;"/>

objc's Runtime is a word that has been rotten by hot posts on the Internet, and good people like to put a "black magic" hat on it. However, there is no magic in this world, everything can be explained from a scientific perspective, and it is not just a few C functions listed in the header file. In this article, let's explore what the Runtime of objc is.

<!--more-->

# The essence of Objective-C
Many programming languages ​​have runtimes, such as JVM for Java, JavascriptCore/v8 for Javascript, etc. (In fact, we should call it the runtime environment here). However, objc alone is particularly special, because its runtime is neither a virtual machine nor an interpreter, but just a library (it appears as libobjc.A.dylib in the iOS system, which is not clear. Please correct me if there is any mistake. ). Isn’t it surprising that a programming language's operating environment is completed by only relying on a library?

There is no surprise. In the strict sense, Objective-C is not a real programming language, as Clang says. Clang claims that Objective-C is just a dialect of C, but this dialect is so "so" that it cannot be recognized that it can be called a language. Conceptually, Objective-C is C. Developers under the Cocoa Touch framework have been writing for so long, and they have not become more "advanced" at the language level. So writing C code in objc seamlessly connects, naturally, objc becomes easy when C++ is mixed.

# Objective-C implementation
The title of this paragraph is really a bit too big. I can’t explain the realization principle of Objctive-C clearly in an article, but at least I can let readers know through some relatively simple examples: Oh, objc is probably like that, then this article The purpose of the article is achieved.

So what kind of implementation is objc? Those who have learned the principles of compilation know that the realization of a programming language requires two major processes: the front end and the back end of the compiler. As a dialect, objc shares the back end with C (that is, there is no back end). It only passes through the front end of the compiler and is rewritten into C/C++ code. With a runtime dynamic library, it can achieve object-oriented Programmed.

We don't know what the front end of objc looks like, but Clang definitely knows it. From some Clang document fragments, we can see that the grammatical derivation of objc is as simple as I thought, such as the grammatical derivation used to describe closures:

```
Block_literal_expression ::= ^ block_decl compound_statement_body
block_decl ::=
block_decl ::= parameter_list
block_decl ::= type_expression
```

So when Clang translates to a Block, it will naturally call methods such as objc_create_block(xxx) (this is just an example, there is no such method, the runtime of the closure is called libclosure). Then the literal value of a Block will be rewritten into a piece of C code and compiled, and a Block instance (structure pointer) is created by relying on the function at runtime.

This compilation mode of objc is easy to think of and realize. One of my C library [enginX](https://github.com/stephenwzl/enginx) that uses DSL syntax to rewrite url is also a "dialect" implemented in this way , And objc is a little different in order to avoid compilation, I wrote an interpreter directly. If you think my code looks terrible, it’s okay. I recommend a book called "Homemade Programming Language", written by Maebashi Kazuya, with a Chinese translation. If you think this book is too shallow, you can also read the "Tiger Book" called "Modern Compilation Principles C Language Description". These two books are very easy to understand. If you can read any of them, it will be of great help to the realization and understanding of objc.

After reading this, I think you can easily think of what the C functions displayed in runtime.h of objc are used. For example, the function class_createInstance(Class cls, size_t extraBytes) will be called when you have written an instantiation method such as [NSObject new]. Others are basically the same. In fact, I think everyone should be able to know: "Oh, objc is probably like that." Because the next thing I want to talk about is some details about the implementation of various features such as object-oriented objc.

#Objective-C Object
Let's look directly at the explanation of objc 2.0, 1.0 has been stuck on iOS 2.0/ OS X 10.5, and there is no point in discussion. Since C is not object-oriented, and objc is just a dialect, there should be no such thing as "object" in objc. In fact, an objc Class is a pointer type of a structure. You can see that the runtime header file is defined as follows:

```c
struct objc_class {
    Class isa OBJC_ISA_AVAILABILITY;
 
#if !__OBJC2__
    Class super_class OBJC2_UNAVAILABLE;
    const char *name OBJC2_UNAVAILABLE;
    long version OBJC2_UNAVAILABLE;
    long info OBJC2_UNAVAILABLE;
    long instance_size OBJC2_UNAVAILABLE;
    struct objc_ivar_list *ivars OBJC2_UNAVAILABLE;
    struct objc_method_list **methodLists OBJC2_UNAVAILABLE;
    struct objc_cache *cache OBJC2_UNAVAILABLE;
    struct objc_protocol_list *protocols OBJC2_UNAVAILABLE;
#endif
 
} OBJC2_UNAVAILABLE;
 
typedef struct objc_class * Class;
```

Obviously, the Class type you use in the objc source file is a pointer to a structure. When your objc application process loads, these Classes will be initialized and placed in a global place, so a [NSObject_instance isKindOfClass:someClass] just compares the pointer of the Class pointed to by this instance. The literal value of the pointer is a number. This comparison is very easy. Of course, you can also use objc_allocateClassPair this method to create a new Class at runtime, because Class itself is an instance, it is not a real "type", (probably [because Apple engineers are not excluded from doing other tricky things]) The classes you write in the compiled code are also created at the beginning of the process. So creating a Class is no different from creating an ordinary reference variable.

At this time, the instance object of objc Class is very embarrassing, because it is only an instance of an instance, not really a Class "type", from [objc-runtime](https://opensource.apple.com/tarballs/objc4/ ) The source code can see that objc_object is another structure:

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
    // many other functions and variables
    ....
};
```

Now you can see how objc identifies its own Class type, but it uses an isa_t structure, and the member variables of this structure have Class variables. The next method called initClassIsa(Class cls) (same as initIsa), it just takes the address of the passed Class, and essentially remembers the first address of the Class (so I guess it’s not wrong for the Class to be stored globally). Then the function changeIsa should be the prototype of the objc_setClass we usually use.

So the question is, how to instantiate an objc object only through these things? Relying on the "fresh air in the village"? of course not. A minimal implementation of the real object of objc (NSObject, in fact, is a protocol), it is a rich objc_class type, why do you say that? Take a look at the rich definition of object_class:

```c
struct objc_class: objc_object {
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
//Very long, the following methods are not listed one by one
}
```

You can see a real Object, the layout of its previous part is always the structure of objc_object, which stores a structure of information about itself. The Class information stored in objc_object is just a simplified version of objc_class (in fact, to save memory). Then after an objc object is instantiated, it is such a structure pointer, it also stores these things, then we can get its ivar, methodList, compliance protocol, etc. At this time, I think you are sure You should understand how the runtime methods of getting the ivar of the instance object are doing. (The definition of object_class is a bit older, and the new one includes some things that interact with Swift, which is likely to cause some interference)

#Objective-C method call
Through the above, it is not difficult to understand how objc objects complete inheritance and instance variable storage, but the problem is coming. Method invocation in OOP programming is a very important concept. Every instance has methods. If not, what is the process-oriented the difference? In a structure, only function pointers can be stored. If an object has a lot of methods, wouldn't the structure of the actual memory layout of this object become extremely long and not universal?

On this issue, objc solves SmallTalk very well. Because at present, there are almost several solutions for function calls: static addresses, virtual table mapping, and message forwarding. Both of the above require a method to determine the absolute position or relative position at compile time, and message forwarding is not required, just a runtime search. This is also a feature of SmallTalk. The communication between objects is to send messages. The message function of objc is objc_msgSend, which couldn't be more clear. On the structure of objc_class, there is also a pointer variable called method List, which maintains a linked list describing methods. The description of a method looks like this:

```c
struct objc_method {
    SEL method_name OBJC2_UNAVAILABLE;
    char *method_types OBJC2_UNAVAILABLE;
    IMP method_imp OBJC2_UNAVAILABLE;
}
```

SEL is actually a string, representing the name of a method, which is also matched by name when searching. method_types is the type description of the parameters required for method invocation, and it can also be called method signature. In the C calling convention, only when the number, type, and order of parameters are the same, a function can be called correctly. IMP is actually a function pointer, pointing to the actual stack address of this function. The objc_msgSend function will pass in the caller and method name (SEL) and parameters, and the push call will only occur after the search and match are correct.

The invocation of message forwarding also has disadvantages. For example, every time you look up the linked list, the search method is definitely slow. For this reason, there is a cache member variable in objc_class. In fact, the cache is used to speed up the search. Everyone can think of a way. But there is still a problem. What if the method cannot be found, and I really make such a call at runtime? The default implementation of NSObject gives an opportunity to forwardingTargetForSelector, which is almost a "must test" for interview questions. The designer of objc will try to consult users: What should I do if I can’t find this method? Who is the message to? If you tell him: send it to the object A! Then he will try to find this method of A in forwardInvocation and generate an abstract called NSInvocation to call. But the A you told him may still not receive your message. At this time, the doesNotRecognizeSelector call will occur. If you don't duplicate it, an exception will be thrown. So doesNotRecognizeSelector is almost your last chance to save this message, but this is usually not reliable, because the message is likely to have been given to A. After A does not recognize it, it will call doesNotRecognizeSelector. In some cases, copying here does not work. . Let's follow the designer's idea.

# Property and Protocol
These two guys are not a new type, I think they are just two "utils" along the way of objc implementation. Property gives users the opportunity to automatically synthesize ivar, getter, and setter methods, which will help users complete during the compilation phase, and a linked list of property_list will be added to the structure of objc_object to tell users which properties are added at runtime (This is also a "must test").

The Protocol is even simpler, it is an objc_object pointer, but the difference from the object is that the Protocol is restricted. Objects that follow a certain Protocol cannot be found in its superClass. The reason is that, like property_list, it is attached to a linked list called protocol_list, so it will not appear in the inheritance chain. And the property of Protocol does not get the "preferential treatment" of the compiler like normal objects.

(Well, the paragraph about Property and Protocol is really lazy, because the design pattern of objc is very obvious, and there is no new idea here, and it will be a little annoying to continue to talk about it)

# Written at the end
The Runtime of objc does not represent any "black magic" or just a few C function APIs. It is the core design idea of ​​objc. Perhaps Apple open sourced it just because it hopes that developers will not make everyone burn out because of this dialect, or maybe The moment it was open sourced, Swift was already in progress. But no matter what, C/C++ will always be the classic of statically compiled languages. It can even be said that if you learn C/C++ well, you will not be afraid to travel all over the world.