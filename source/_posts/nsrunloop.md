---
title: Detailed NSRunloop
date: 2017-06-19 19:02:16
tags:
---
NSRunloop is the most basic event loop mechanism in macOS and iOS, and it can also be understood as a thread abstraction made by Apple engineers. This article will explain the concept and principle of NSRunloop step by step, as well as some applications in actual engineering.
<!--more-->

# NSThread and NSRunloop
Undoubtedly, NSThread and NSRunloop have a one-to-one correspondence. That's fine, but it's not a good explanation. In fact, in the early implementation of Foundation (maybe the current implementation is also), NSRunloop is a reference attribute of the NSThread object, or directly, it can be said to be an ivar (instance variable). NSThread holds a structure called ThreadInfo, a member of this structure is NSRunloop. It is not that NSThread will generate NSRunloop when it is created, it will only appear when it is referenced (lazy loading).

But we should break the stereotype of NSThread, because NSThread is not a thread, it is just the Foundation framework's encapsulation of the pthread operation in the underlying framework (POSIX Thread), and pthread is managed by the current process, and each process maintains a name It is a key structure array. NSThread cannot hold a pthread. It only holds the key of the target thread, and then obtains a pthread reference from the process.

But what is the relationship between pthread and NSRunloop? NSRunloop maintains the content that pthread needs to execute, and formulates the trigger sequence and rules for these content. For details, please continue reading below.

# NSRunloop composition
NSRunloop is the performance of CFRunloop's objc. They are toll-free bridges (that is, conversion without CPU time). The good news is that CoreFoundation is open source. We can view the implementation of CFRunloop. Since Swift Language is open sourced on Github, the Swift implementation version of CoreFoundation has also been placed on Github. This allows us to compare the modern way to look at the code. The address is here: [swift-corelibs-foundation](https://github.com/apple/swift-corelibs-foundation) It contains the latest CoreFoundation implementation and I am very excited.

However, it is not a good idea to look directly at the implementation of CFRunloop, because it is not abstract enough. First, explain how NSRunloop expresses what the thread needs to execute. As shown below:

<img src="/images/Group-2-768x663.png" style="max-width:350px"/>

An NSRunloop can select a Mode each time it is executed, and then execute each Performer in turn, so an NSRunloop can have multiple Modes, but it cannot be switched to other Modes in one execution cycle. By default, there are "two" modes for an NSRunloop, one is called DefaultMode and the other is called CommonMode. But CommonMode is not a real Mode, it represents all the Modes of the current Runloop (but there are still many Modes that will not accept CommonMode's Performer, but usually you can't touch it). For example, if you need to add a Performer to an NSRunloop to do something, and hope that this Performer can get a callback regardless of which Mode, then you need to add it to CommonMode, so that NSRunloop will traverse its own Modes, and then go to Insert your Performer into each Mode set. A typical application case is that you need a timer, you need to add it to CommonMode, so that even if the UIScrollView enters UITrackingRunloopMode when sliding, your timer can still get the correct callback.

NSRunloop will go to sleep after the execution of a Mode cycle. After receiving a wake-up event, it will select which Mode to execute in the next cycle according to the type of wake-up event. Usually events are divided into two types, one is human events, such as code calls and event notifications. There is also port communication, which means it is passed from other processes. The most typical example is the touch screen. In iOS, screen events are first passed to SpringBoard, and then touch screen events are passed to the currently active App process from the port. At this time, the main thread will be "awakened" (the main thread does not actually sleep). If a touch screen event occurs on a ScrollView, the Runloop of the main thread will enter UITrackingRunloopMode at this time.

# CFRunloop composition
In fact, up to this point, we already have a clear understanding of NSRunloop, but the reason we need to continue to dig down is that the implementation of NSRunloop does not completely follow the concept of CFRunloop, which also buried some holes. [Haha, can't think of it.jpg]

Take a closer look at CFRunloop, its conceptual hierarchy and granularity are almost completely different from NSRunloop. It is gratifying that the basic unit of an event loop is still Mode. The difference is that the performer in Mode (called modeItem in CoreFoundation) ) Is divided into several dimensions: Source, Timer, Observer. So when the callback of NSTimer is added to Runloop, it will eventually be added to the Timer collection. However, the collection of Source can be divided into more detailed: Source0 and Source1. Source0 is an artificially added execution action, such as executing a selector on the current thread. Source1 is a port event and can only be triggered by the port. Observer is a callback. Runloop will call back the corresponding Observer at a certain stage of execution. It is worth mentioning that NSRunloop abstracts these three as Performer, but it does not use CoreFoundation API when operating the underlying structure of Runloop. It seems to be implemented only through memory alignment. This leads to **NSRunloop is not thread-safe**. But the CoreFoundation API is thread-safe.
**CFRunloopSourceRef** is the structure pointer of CFRunloopSource, this structure looks like this:

```c
struct __CFRunLoop {
    CFRuntimeBase _base;
    pthread_mutex_t _lock;			/* locked for accessing mode list */
    __CFPort _wakeUpPort;			// used for CFRunLoopWakeUp 
    Boolean _unused;
    volatile _per_run_data *_perRunData;              // reset for runs of the run loop
    pthread_t _pthread;
    uint32_t _winthread;
    CFMutableSetRef _commonModes;
    CFMutableSetRef _commonModeItems;
    CFRunLoopModeRef _currentMode;
    CFMutableSetRef _modes;
    struct _block_item *_blocks_head;
    struct _block_item *_blocks_tail;
    CFAbsoluteTime _runTime;
    CFAbsoluteTime _sleepTime;
    CFTypeRef _counterpart;
};
```

This time we will not explain each field one by one. After all, many fields are some Context used at runtime, which will only cause some interference to our understanding of CFRunloop. We can see that there is a thread lock in the structure, and the comment also talks about the locks used to acquire the resources of modes. Then there is a \_wakeUpPort field to mark which port this Runloop wakes up from. The _perRunData field is used to abstractly represent the data necessary for each Runloop when it runs. Volatile also tells the compiler well that the content of this pointer will often change, so don't do any read-write optimization. Next, _commonModes is used to mark which modes are Common, and _commonModeItems is used to mark which added items are in CommonModes. _currentMode is well understood, it is the specific Mode executed in a Runloop cycle.

Next we should take a look at the three King Kong Source, Timer and Observer in CFRunloop. They exist in CFRunLoopMode, and its structure is like this:

```c
struct __CFRunLoopMode {
    CFRuntimeBase _base;
    pthread_mutex_t _lock; /* must have the run loop locked before locking this */
    CFStringRef _name;
    Boolean _stopped;
    char _padding[3];
    CFMutableSetRef _sources0;
    CFMutableSetRef _sources1;
    CFMutableArrayRef _observers;
    CFMutableArrayRef _timers;
    CFMutableDictionaryRef _portToV1SourceMap;
    __CFPortSet _portSet;
    CFIndex _observerMask;
#if USE_DISPATCH_SOURCE_FOR_TIMERS
    dispatch_source_t _timerSource;
    dispatch_queue_t _queue;
    Boolean _timerFired; // set to true by the source when a timer has fired
    Boolean _dispatchTimerArmed;
#endif
#if USE_MK_TIMER_TOO
    __CFPort _timerPort;
    Boolean _mkTimerArmed;
#endif
#if DEPLOYMENT_TARGET_WINDOWS
    DWORD _msgQMask;
    void (*_msgPump)(void);
#endif
    uint64_t _timerSoftDeadline; /* TSR */
    uint64_t _timerHardDeadline; /* TSR */
};
```

We still try to look at the parts that can be understood. First, I saw another _lock, which is also used to lock when accessing CFRunloopMode resources. Then a Mode will also have a name and a status mark whether it is stopped. Then, you can see source0, source1, observers, timers from the field names. In addition, we can also see that Runloop can also execute dispatch source and queue, but unfortunately their field type is not CFMutableSetRef, which means that a Runloop Mode can only execute one dispatch source/queue.

# Observe the Main Runloop of the App
Through the toll-free bridge, we can intuitively see what the Main Runloop of an iOS App looks like by default. For this reason, I copied a copy of the structure declaration of \_\_CFRunloop and renamed it to _RD_CFRunloop (RD for runloop demo). Other related data structures that cannot be automatically linked are also copied, plus RD prefix. Then I can get the detailed information of mainRunloop upright.

```c
//Through toll-free bridge
_RD_CFRunLoop *currentRunloop = (__bridge _RD_CFRunLoop *)[NSRunLoop mainRunLoop];
 
//Of course you can also use the CoreFoundation API and then force the pointer
_RD_CFRunLoop *currentRunloop = (void *)CFRunLoopGetMain();
```

Then we directly interrupt the point and preview the values ​​of these structures in the Xcode console.
![](/images/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7-2017-06-17-10.17.02-768x607.png)

At this time, you can directly print the value of _commonModes to see

![](/images/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7-2017-06-17-10.23.54.png)

You can see that there are only two commonModes of mainRunloop, one is UITrackingRunloopMode, and the other is kCFRunLoopDefaultMode. Then we can also take a look at _currentMode (not listed in the figure), the name is UIInitializationRunLoopMode (the call stack at this time is in app didFinishLaunching), this Mode is obviously not commonMode, so the callback we add to commonMode will not exist in this mode in.

Next, we can look at what is in \_commonModeItems. According to the above, the callback in _commonModeItems will definitely be executed by the Mode marked as commonMode. However, there are more _commonModeItems, we can convert it to NSMutableSet, and use the object-oriented API to look at them in a loop.

```c
NSMutableSet *commonModeItems = (__bridge NSMutableSet *)runloop->_commonModeItems;
[commonModeItems enumerateObjectsUsingBlock:^(id obj, BOOL *stop) {
  NSLog("%@", obj);
}];
```

But this is a bit long, I will make a simple list:

source0:

* PurpleEventSignalCallback, order = -1
* FBSerialQueueRunLoopSourceHandler, order = 0
* \_\_handleEventQueue, order = -1
* \_\_handleHIDEvnetFetcherDrain, order = -2


source1:

* \_ZL26power_notify_port_callbackP12__CFMachPortPvls1\_, port = 2407, order = 0
* MIG Server, port = 26891, order = 0
* \_ZL27change_notify_port_callbackP12__CFMachPortPvlS1, port = 1b03, order = 0
* PurpleEventCallBack, order = -1
* MIG Server, port = 21775, order = 0


observer:

* \_beforeCACommitHandler, order = 1999000
* \_ZN2CA11Transaction17observer_callbackEP19__CFRunLoopObservermPv, order = 2000000
* \_UIGestureRecognizerUpdateObserver, order = 0
* \_wrapRunLoopWithAutoreleasePoolHandler, order = -2147483647
* \_wrapRunLoopWithAutoreleasePoolHandler, order = 2147483647
* \_afterCACommitHandler, order = 2001000


These are the modeItems that Apple will get callbacks in every commonMode execution cycle when implementing the mainRunloop of an App (the real simulator and different system versions may be different). Although it cannot be explained in full, it can be seen in general from some keywords that some functions carried by mainRunloop such as the serial queue of the main thread, processing hardware events, processing global event queues and so on. But we should take a look at what modes the mainRunloop has:

```objc
NSMutableSet *modes= (__bridge NSMutableSet *)runloop->——modes;
[modes enumerateObjectsUsingBlock:^(id obj, BOOL *stop) {
  NSLog("%@", obj);
}];
```

These values ​​are printed:

* UITrackingRunLoopMode
* GSEventReceiveRunLoopMode
* kCFRunLoopDefaultMode
* UIInitializationRunLoopMode
* kCFRunLoopCommonModes

Among them, kCFRunLoopCommonModes is just a placeholder. It is not really a Mode that can be executed by a Runloop cycle. It is just that when a new item is inserted into its collection, other Modes marked as commonMode will also be added to this new one. Item. UIInitializationRunLoopMode can only capture the _currentMode pointing to it when our current app didFinishLaunching. Once the app is initialized, this Mode will no longer be used. As for GSEventReceiveRunLoopMode, let's just wait and see its changes.

# RunLoop execution

It’s hard to guess the execution of runloop, but [Apple’s documentation](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/Multithreading/RunLoopManagement/RunLoopManagement.html#//apple_ref/doc/ uid/10000057i-CH16-SW23) still tells us something. The specific sequence is explained briefly:

1. Notify the observer that it has entered the runloop
2. Notify observer timer callback will be triggered
3. Notify the observer that the source of non-port-based input (ie source0) will be executed
4. Execute source0
5. If there are port-based input events (ie source1), execute them, and then skip to step 9
6. Notify the observer that the thread is about to enter sleep
7. Sleeping thread, waiting to wake up. A thread can be awakened by a port-based source or a timer. But the runloop itself also has a timeout time, and it can also be woken up based on this, but this is the internal logic, and the outside is unreachable. The last and most direct one can be directly awakened through CoreFoundation API
8. Notify observer that the thread has just been woken up
9. Process the message received upon wake-up. When awakened by the port source, the event is directly distributed (that is, executed). If awakened by the user's timer, the timer will be executed and the runloop will restart. If it is directly awakened and the runloop does not time out, the runloop will also restart. Eventually will skip to step 2.
10. Notify the observer that runloop is about to exit.

Judging from the above steps, the observer is very popular in the entire execution. This also explains what CFRunLoopObserver will have so many CFRunlLoopActivity flags (see CFRunLoop.h).

In fact, we can look at the source code of the CFRunLoopRunSpecific method (CFRunLoop.m Line 2912-2945). The source code is not included here. A simple interpretation is that the CFRunLoopSpecific method will first make some basic judgments, and then push a per_run_data The structure of for each mode executed, before and after a mode is actually executed, the observer will be notified of two events: kCFRunLoopEntry and kCFRunLoopExit. Execute a mode call CFRunLoopRun method (Line 2590-2908).

Let me briefly explain the flow of the CFRunLoopRun method. At the beginning, it will be judged whether there are sources and timers related to dispatch. They will get callbacks first. Obviously, Apple's Runloop document does not mention this step. Next, the ignorePortWakeUp flag bit of per_run_data will be unset, which means that the port event can be awakened during the real mode execution cycle, which is in line with the document description. Next, the Observer callbacks of kCFRunLoopBeforeTimers and kCFRunLoopBeforeSources will be executed continuously. The next step is to execute the source0 that we love to hear, __CFRunLoopDoSources0 (Line 1936-1994) such an obvious method name, you can see it even if you are blind. Next, a mess of macro judgments is basically talking about handling port input events. The next step is to notify the observer that it is going to sleep, and then it really adjusts the __CFRunLoopSetSleeping method (Apple does not deceive me). Next, there will be a do while(1) loop to listen to port events, and there will also be a sleepStart time stamp. In this do while(1) loop, if a port event is received, or the mark time of sleepStart exceeds the expirTime of the internal structure of the runLoop or the fire time of the timer, the runLoop will be awakened again. Then it will unSetSleeping and notify the observer that the kCFRunLoopAfterWaiting event has occurred. The next thing is a bit repetitive, and the explanation is a bit verbose, so please read the code for yourself.

# Simple use

Now that we know the execution logic of Runloop, we can do something interesting based on this, for example, it can be used to monitor the status of iOS pages. (What is this operation?)

Usually, we request a string of JSON strings from the server, serialize them and submit them to the view for rendering. At this time, Main RunLoop will execute a cycle from wake-up to exit, that is, from kCFRunLoopBeforeWaiting to kCFRunLoopExit. Why is there such a judgment? From the above inspection of the main Runloop Observer, it is obvious that you can see the callback _ZN2CA11Transaction17observer_callbackEP19__CFRunLoopObservermPv. It is registered by Apple and is used to mark the interface update. When it is called back, it will calculate and draw the view. Then we only need to register an observer, which is called back once before it, and then called back again after it, then it can be calculated When you travel time, you know how much time it took for an interface to be marked as updated to when the update was completed.

How to implement it depends on your understanding and talents of Runloop.