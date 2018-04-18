---
title: 详解 NSRunloop
date: 2017-06-19 19:02:16
tags:
---
NSRunloop是 macOS和 iOS中最基本的事件循环机制，也可以理解为 Apple工程师们做的线程抽象。本文将逐步讲解 NSRunloop的概念和原理，以及在实际工程中的一些运用。
<!--more-->

# NSThread和 NSRunloop
无疑，NSThread和 NSRunloop是一一对应的。这么说没毛病，但也算不上好的解释。实际上，在早期 Foundation的实现中（可能现在的实现也是），NSRunloop是 NSThread对象的一个引用属性，或者直接一点，可以说就是一个 ivar (instance variable)。NSThread持有一个叫做 ThreadInfo的结构体，这个结构体的一个成员就是 NSRunloop。并不是 NSThread在创建的时候就会产生 NSRunloop，它只会在被引用的时候才会出现（懒加载）。

但我们应该打破一下对 NSThread的刻板印象，因为 NSThread并不是线程，它只是 Foundation框架对 底层框架中 pthread操作的封装（POSIX Thread），而 pthread是由当前进程管理的，每个进程都会维护一个称之为 Key的结构体数组，NSThread并不可能持有 pthread，它只持有目标线程的 key，然后从进程获得 pthread的引用。

但 pthread和 NSRunloop又扯上了什么关系？NSRunloop维持着 pthread所需要执行的内容，并且制定了这些内容的触发顺序和规则。具体是什么，请继续阅读下文。

# NSRunloop的构成
NSRunloop是 CFRunloop的 objc的表现，他们是 toll-free bridge（也就是不耗 CPU时间转换）的。值得高兴的是，CoreFoundation是开源的，我们完全可以查看 CFRunloop的实现，自从 Swift Language在 Github开源后，CoreFoundation的 Swift实现版本也放在了 Github上，这让我们可以比较 Modern的方式去看代码，地址在这儿：[swift-corelibs-foundation](https://github.com/apple/swift-corelibs-foundation)  其中包含了最新的 CoreFoundation实现，很兴奋。

不过，直接看 CFRunloop的实现并不是什么好主意，因为它还不够抽象。先解释一下 NSRunloop是如何表示线程所需执行的内容的。如下图：

<img src="http://cdn.stephenw.cc/wp-content/uploads/2017/06/Group-2-768x663.png" style="max-width:350px"/>

一个 NSRunloop可以在每次执行时选择一个 Mode，然后依次执行其中的每一个 Performer，所以一个 NSRunloop可以有多个 Mode，但不能在一次执行周期内切换到其他 Mode。默认情况下，一个 NSRunloop的 Mode有“两个”，一个叫 DefaultMode，另一个叫 CommonMode。但 CommonMode并不是一个真正意义上的 Mode，它代表的是当前这个 Runloop的所有 Mode（但还是有很多 Mode不会接受 CommonMode的 Performer，但通常情况下你碰不到）。比如你需要往一个 NSRunloop添加一个 Performer来做一些事情，并且希望这个 Performer无论在那种 Mode下都能得到回调，那么你需要把它添加到 CommonMode里面去，这样 NSRunloop会遍历自己的 Modes，然后往每一个 Mode的 set里面都插一个你的 Performer。一个典型的应用案例就是你需要一个计时器，你需要添加到 CommonMode，这样即使 UIScrollView在滑动时进入了 UITrackingRunloopMode，你的计时器仍然能得到正确的回调。

NSRunloop在一次 Mode周期执行完毕后这个线程将会进入休眠，在接收到唤醒事件后会根据唤醒的事件类型选择下次周期执行哪个 Mode。通常事件分为两种类型，一种是人为事件，比如代码调用，事件通知。还有一种是端口通信，这意味着是从其他进程传递过来的，最典型的例子是触摸屏，iOS中屏幕事件首先会传递给 SpringBoard，然后从端口把触屏事件传递给当前活跃的 App进程，这时候主线程会被“唤醒”（主线程其实不会休眠），假如触屏事件发生在一个 ScrollView上，这时候主线程的 Runloop就会进入 UITrackingRunloopMode。

# CFRunloop的构成
其实到这儿，我们对 NSRunloop已经有了一个很清晰的了解了，但还需要继续往下挖的原因是 NSRunloop的实现并没有完全按照 CFRunloop的概念来，这也埋下了一些坑。[哈哈，想不到吧.jpg]

仔细瞅一眼 CFRunloop，它的概念划分层次和粒度和 NSRunloop几乎是完全不同的，值得欣慰的是，一次事件循环的基本单位仍然是 Mode，不同的是，Mode里的 performer（在 CoreFoundation这里被叫做 modeItem）被划分成了几个维度：Source、Timer、Observer。  所以在 Runloop添加 NSTimer的回调时，最终会被添加到 Timer这个集合里。不过 Source这个集合划分得还会更细一点：分为 Source0和 Source1，Source0是人为添加的执行动作，比如在当前线程上执行一个 selector等。Source1是端口事件，只会由端口触发。Observer便是回调了，Runloop在执行到某个阶段都会回调对应的 Observer。  值得一提的是，NSRunloop把这三者统一抽象为 Performer，但在操作 Runloop的底层结构时并没有用 CoreFoundation API，貌似只通过内存对齐实现。这就导致 **NSRunloop是非线程安全的**。但CoreFoundation API是线程安全的。
**CFRunloopSourceRef** 是 CFRunloopSource的结构体指针，这个结构体长这样：

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

这次我们就不一个字段一个字段去解释了，毕竟不少字段是在运行时用到的一些 Context，对我们理解 CFRunloop只会造成一些干扰。我们可以看到结构体里面有一个线程锁，注释也讲了用于获取 modes这些资源的锁。然后还有一个 \_wakeUpPort字段，用于标记这个 Runloop由哪个端口唤醒的。_perRunData字段用于抽象地表示每一次 Runloop在 Run的时候所必须的数据，volatile也很好地告诉编译器，这个指针的内容是经常会变的，不要做任何的读写优化。接下来 _commonModes用于标记 哪些 mode是 Common的，_commonModeItems用于标记哪些加入的 item是存在于 CommonModes里面的。_currentMode很好理解，就是一次 Runloop周期内所执行的具体的 Mode。

接下来我们该看一下 CFRunloop中的三大金刚 Source, Timer和 Observer了。他们存在于 CFRunLoopMode中，它的结构是这样的：

```c
struct __CFRunLoopMode {
    CFRuntimeBase _base;
    pthread_mutex_t _lock;	/* must have the run loop locked before locking this */
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

我们还是尽量看能看得懂的部分。首先，又看到了一个 _lock，这同样用于访问 CFRunloopMode的资源时加锁。接着一个 Mode还会有一个名字，一个状态标记是否 stopped。接着，从字段名字就可以看到 source0, source1, observers, timers。除此之外，我们还可以看到 Runloop还可以执行 dispatch的 source和 queue，但很可惜它们字段的类型不是 CFMutableSetRef，也就是说一个 Runloop的 Mode只能执行一个 dispatch source/queue。

# 观察 App 的 Main Runloop
通过 toll-free bridge，我们可以很直观地看一下默认情况下一个 iOS App的 Main Runloop是什么样的。为此，我 copy了一份 \_\_CFRunloop 的结构体声明，重命名为 _RD_CFRunloop (RD for runloop demo)，其他关联到的数据结构，不能自动 link的，也都 copy了一份，加上了 RD前缀。然后我就可以堂堂正正地获取 mainRunloop的详细信息了。

```c
//通过 toll-free bridge
_RD_CFRunLoop *currentRunloop = (__bridge _RD_CFRunLoop *)[NSRunLoop mainRunLoop];
 
//当然也可以用 CoreFoundation API然后指针强转
_RD_CFRunLoop *currentRunloop = (void *)CFRunLoopGetMain();
```

然后我们直接打断点，在 Xcode控制台中先预览一下这些结构有哪些值。
![](http://cdn.stephenw.cc/wp-content/uploads/2017/06/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7-2017-06-17-10.17.02-768x607.png)

这时候可以直接打印一下 _commonModes的值看一下

![](http://cdn.stephenw.cc/wp-content/uploads/2017/06/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7-2017-06-17-10.23.54.png)

可以看到 mainRunloop的 commonMode只有两个，一个是 UITrackingRunloopMode，另一个是 kCFRunLoopDefaultMode。然后我们还可以看一下 _currentMode（图就不列了），name是 UIInitializationRunLoopMode（此时的调用栈处于 app didFinishLaunching），这个 Mode很明显不是 commonMode，所以我们给 commonMode添加的回调不会存在于这个 mode中。

接下来我们可以看一下 \_commonModeItems里面有哪些东西，按上文所述，_commonModeItems里的回调肯定会被标记为 commonMode的 Mode执行。不过 _commonModeItems比较多，我们可以把它转成 NSMutableSet，用面向对象的 API在循环中挨个看。

```c
NSMutableSet *commonModeItems = (__bridge NSMutableSet *)runloop->_commonModeItems;
[commonModeItems enumerateObjectsUsingBlock:^(id obj, BOOL *stop) {
  NSLog("%@", obj);
}];
```

不过这个有点长，我就做一个简单的罗列:

source0:

* PurpleEventSignalCallback, order = -1
* FBSerialQueueRunLoopSourceHandler, order = 0
* \_\_handleEventQueue, order = -1
* \_\_handleHIDEvnetFetcherDrain, order = -2


source1:

* \_ZL26power_notify_port_callbackP12__CFMachPortPvls1\_, port = 2407 , order = 0
* MIG Server, port = 26891, order = 0
* \_ZL27change_notify_port_callbackP12__CFMachPortPvlS1, port = 1b03, order = 0
* PurpleEventCallBack,  order = -1
* MIG Server, port = 21775, order = 0


observer:

* \_beforeCACommitHandler, order = 1999000
* \_ZN2CA11Transaction17observer_callbackEP19__CFRunLoopObservermPv，order = 2000000
* \_UIGestureRecognizerUpdateObserver, order = 0
* \_wrapRunLoopWithAutoreleasePoolHandler, order = -2147483647
* \_wrapRunLoopWithAutoreleasePoolHandler, order = 2147483647
* \_afterCACommitHandler, order = 2001000


这些就是 Apple在实现一个 App的 mainRunloop的时候在每个 commonMode执行周期里都会得到回调的 modeItem（模拟器真机以及不同系统版本可能会有些差异）。虽然不能全部解释，但大体上还能从一些关键字看出，mainRunloop承载的一些功能比如主线程的串行队列，处理硬件事件，处理全局事件队列等等。但我们应该看一下 mainRunloop到底有哪些 mode：

```objc
NSMutableSet *modes= (__bridge NSMutableSet *)runloop->——modes;
[modes enumerateObjectsUsingBlock:^(id obj, BOOL *stop) {
  NSLog("%@", obj);
}];
```

打印出了这几个值：

* UITrackingRunLoopMode
* GSEventReceiveRunLoopMode
* kCFRunLoopDefaultMode
* UIInitializationRunLoopMode
* kCFRunLoopCommonModes

其中 kCFRunLoopCommonModes只是一个占位符罢了，它并不是一个真正可以被一个 Runloop周期可执行的 Mode，只是在往它的集合里插入新的 item时，其他标记为 commonMode的 Mode也一并会加入这个新的 item。而 UIInitializationRunLoopMode只在我们当前 app didFinishLaunching时才能捕捉到 _currentMode指向它，一旦 App初始化完成后，这个 Mode便不会再被使用到了。至于 GSEventReceiveRunLoopMode，我们还是静观其变吧。

# RunLoop 的执行

runloop的执行我们不好去猜，不过 [Apple的文档](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/Multithreading/RunLoopManagement/RunLoopManagement.html#//apple_ref/doc/uid/10000057i-CH16-SW23)还是告诉了我们一些事情。具体的顺序简单地解释一遍：

1. 通知 observer 已经进入 runloop
2. 通知 observer timer回调将被触发
3. 通知 observer，非基于 port的输入的 source （也就是 source0)将会执行
4. 执行 source0
5. 如果有基于 port的输入事件（也就是 source1），执行它们，然后跳到第 9步
6. 通知 observer 线程即将进入休眠
7. 休眠线程，等待唤醒。线程可以被 基于 port的 source唤醒，也可以通过 timer唤醒。但 runloop自身内部也会有超时时间，也可以基于此唤醒，但这是内部的逻辑了，外部不可达。最后最直接的，可以通过 CoreFoundation API直接唤醒
8. 通知 observer 线程刚被唤醒
9. 处理唤醒时收到的消息。通过 port source唤醒的，直接分发这个事件（也就是执行）。被用户的 timer唤醒的，会执行 timer回掉，然后重启 runloop。假如被直接唤醒并且 runloop未超时，runloop也会重启。最终都会跳到步骤 2.
10. 通知 observer runloop即将退出。

从上面的步骤来看，observer在整个执行中很抢戏。这也就解释了 CFRunLoopObserver会什么会有那么多 CFRunlLoopActivity的标志位（参见 CFRunLoop.h）。

其实，我们完全可以去看 CFRunLoopRunSpecific方法的源代码 （CFRunLoop.m Line 2912-2945)，此处就不放源码了，做一个简单的解读就是 CFRunLoopSpecific方法会先做一些基本的判断，然后 push 一个 per_run_data的结构体给每一次执行的 mode，在真正地执行一个 mode前后，会通知 observer两个事件: kCFRunLoopEntry和 kCFRunLoopExit。执行一个 mode调用的是 CFRunLoopRun方法（Line 2590-2908)。

再简单地解释一下 CFRunLoopRun方法的流程。一开始就会判断是否有加入 dispatch相关的 source和 timer，它们会首先得到回调，很明显 Apple的 Runloop文档没有对此一步有所提及。接下来会 unset掉 per_run_data 的 ignorePortWakeUp标志位，也就是说真正的 mode执行周期内可以由端口事件唤醒，这是符合文档描述的。接下来会连续执行 kCFRunLoopBeforeTimers 和 kCFRunLoopBeforeSources的 Observer回调。接下来就是要执行我们喜闻乐见的 source0了，__CFRunLoopDoSources0（Line 1936-1994） 这么明显的方法名，再瞎也看得见了。再接下来，一通乱七八糟的宏判断，大体就是在说处理 port的输入事件。再接下来就是通知 observer 要进入休眠了，然后就真的调了 __CFRunLoopSetSleeping方法（Apple 诚不吾欺）。接下来会有一个 do while(1)循环去监听 port事件，也会有一个 sleepStart的时间标志。在这个 do while(1)循环里，假如收到了 port事件，或者 sleepStart的标志时间超出了 runLoop内部结构体的 expirTime或 timer的 fire time，runLoop会再次被唤醒。然后会 unSetSleeping并通知 observer 发生了 kCFRunLoopAfterWaiting事件。接下来的事就有点重复了，再解释就有点啰嗦，各位自己看代码吧。

# 简单的运用

既然知道了 Runloop的执行逻辑，我们便可以基于此来做一点有意思的事，比如可以用于做 iOS页面状态的监控。（这是什么操作？）

通常地，我们从服务器请求到一串 JSON字符串，序列化后提交给 view渲染，这时候 Main RunLoop会执行一个从唤醒到退出的周期，即从 kCFRunLoopBeforeWaiting 到 kCFRunLoopExit。为什么有这种判断呢？从上文检查 main Runloop的 Observer很明显可以看到 _ZN2CA11Transaction17observer_callbackEP19__CFRunLoopObservermPv这个回调。它是 Apple注册的，用于标记界面更新，在它被回调时会计算和绘制 view，那么我们只需要注册一个 observer，在它之前被回调一次，在它之后再被回调一次，那么就可以计算出时间差，就知道一个界面从被标记为更新到更新完成到底花了多少时间。

具体怎么实现，看各位的对 Runloop的理解和才华了。嘻嘻
