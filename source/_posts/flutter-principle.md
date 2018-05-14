---
title: Flutter原理简解
date: 2018-05-14 11:28:23
tags:
---

![](/images/flutter.png)
Flutter 是 Google推出并开源的移动应用开发框架，主打跨平台、高保真、高性能。开发者可以通过 Dart语言开发 App，一套代码同时运行在 iOS 和 Android平台。 Flutter提供了丰富的组件、接口，开发者可以很快地为 Flutter添加 native扩展。同时 Flutter还使用 Native引擎渲染视图，这无疑能为用户提供良好的体验。
<!--more-->  

好了，上面的官腔打完了，我们下面开始去理解 Flutter的实现原理。  

# 绘图基本原理
提到原理，我们要从屏幕显示图像的基本原理开始谈起。  
我们都知道显示器以固定的频率刷新，比如 iPhone的 60Hz、iPad Pro的 120Hz。当一帧图像绘制完毕后准备绘制下一帧时，显示器会发出一个垂直同步信号（VSync），所以 60Hz的屏幕就会一秒内发出 60次这样的信号。  
并且一般地来说，计算机系统中，CPU、GPU和显示器以一种特定的方式协作：CPU将计算好的显示内容提交给 GPU，GPU渲染后放入帧缓冲区，然后视频控制器按照 VSync信号从帧缓冲区取帧数据传递给显示器显示。  


![图自 ibireme.com](/images/screen_display.png)


> 上图来自 ibireme.com  

所以，Android、iOS的 App在显示 UI时是如此，Flutter也不例外，也遵循了这种模式。所以从这里可以看出 Flutter和 React-Native之众的本质区别：React-Native之类只是扩展调用 OEM组件，而 Flutter是自己渲染。  
在 Flutter Architecture的解释中，Google还提供了一张更为详尽的图来解释 Flutter的原理：  

![](/images/flutter-vsync.png)  

这张图解释得更清晰一些：Flutter只关心向 GPU提供视图数据，GPU的 VSync信号同步到 UI线程，UI线程使用 Dart来构建抽象的视图结构，这份数据结构在 GPU线程进行图层合成，视图数据提供给 Skia引擎渲染为 GPU数据，这些数据通过 OpenGL或者 Vulkan提供给 GPU。   
   
所以 Flutter并不关心显示器、视频控制器以及 GPU具体工作，它只关心 GPU发出的 VSync信号，尽可能快地在两个 VSync信号之间计算并合成视图数据，并且把数据提供给 GPU。  

# 几个问题  
了解 Flutter的基本概念后，自然有几个疑问亟待解决。  

## 1. 为什么使用 Dart？  
这是一个很有意思的问题，Flutter选择了 Dart而不是 JavaScript。我觉得主要有以下几个原因：

1. Dart 的性能更好。Dart在 JIT模式下，速度与 JavaScript基本持平。但是 Dart支持 AOT，当以 AOT模式运行时，JavaScript便远远追不上了。速度的提升对高帧率下的视图数据计算很有帮助。
2. Native Binding。在 Android上，v8的 Native Binding可以很好地实现，但是 iOS上的 JavaScriptCore不可以，所以如果使用 JavaScript，Flutter 基础框架的代码模式就很难统一了。而 Dart的 Native Binding可以很好地通过 Dart Lib实现。  
3. Fuchsia OS，看起来不像原因的一个原因。Fuchsia OS内置的应用浏览器就是使用 Dart语言作为 App的开发语言。而且实际上，Flutter是 Fuchisa OS的应用框架概念上的一个子集。（Flutter源代码和编译工具链也充斥了大量的 Fuchsia宏）
4. Dart是**类型安全**的语言，拥有完善的包管理和诸多特性。Google召集了如此多个编程语言界的设计专家开发出这样一门语言，旨在取代 JavaScript，所以 Fuchsia OS内置了 Dart。Dart可以作为 embedded lib嵌入应用，而不用只能随着系统升级才能获得更新，这也是优势之一。

## 2. Skia是什么？
前面提到了 Flutter只关心如何构建视图抽象结构，向 GPU提供视图数据。Skia就是 Flutter向 GPU提供数据的途径。  

Skia是一个 2D的绘图引擎库，其前身是一个向量绘图软件，Chrome和 Android均采用 Skia作为绘图引擎。Skia提供了非常友好的 API，并且在图形转换、文字渲染、位图渲染方面都提供了友好、高效的表现。Skia是跨平台的，所以可以被嵌入到 Flutter的 iOS SDK中，而不用去研究 iOS闭源的 Core Graphics / Core Animation。  

> Android自带了 Skia，所以 Flutter Android SDK要比 iOS SDK小很多。

## 3. Flutter是如何设计的？ 
上面说了这么久的基础，我们可能只知道 Flutter做了什么，始终都还没有从侧面观察 Flutter的整个架构设计，了解 Flutter如何去做。  

![](/images/flutter-framework.png)  

这张图了解过 Flutter的人可能很多地方都看过，这边来详细解释一下：  

**Flutter Framework**: 这是一个纯 Dart实现的 SDK，类似于 React在 JavaScript中的作用。它实现了一套基础库， 用于处理动画、绘图和手势。并且基于绘图封装了一套 UI组件库，然后根据 `Material` 和`Cupertino`两种视觉风格区分开来。这个纯 Dart实现的 SDK被封装为了一个叫作 `dart:ui`的 Dart库。我们在使用 Flutter写 App的时候，直接导入这个库即可使用组件等功能。

**Flutter Engine**: 这是一个纯 C++实现的 SDK，其中囊括了 Skia引擎、Dart运行时、文字排版引擎等。不过说白了，它就是 Dart的一个运行时，它可以以 JIT、JIT Snapshot 或者 AOT的模式运行 Dart代码。在代码调用 `dart:ui`库时，提供 `dart:ui`库中 Native Binding 实现。 不过别忘了，这个运行时还控制着 VSync信号的传递、GPU数据的填充等，并且还负责把客户端的事件传递到运行时中的代码。   <br />  
** 在了解屏幕绘图的基本原理和 Flutter的一个整体概念后，我们下面详细地来看一下 Flutter的大概实现。**  

> 由于我 Android知识不咋样，所以仅分析 iOS平台上的实现。Android可以参考本文思路理解代码


# Flutter应用的运行  

要理解 Flutter的原理，我们从 entry point开始看 Flutter的代码。由于应用框架大同小异，所以下文提及 `Flutter的代码`即指代 Flutter Engine的代码，而非 Flutter Dart Framework代码。  

下图是我简单整理了一下 Flutter应用启动后的执行顺序  
![](/images/flutter-project.png) 

在应用的 View Controller 初始化后，会实例化一个 Flutter project的抽象（以下简称 project）。project会初始化一个 platform view的抽象实例，这个抽象实例会负责创建 Flutter 的运行时（以下简称 engine）。  

当 View Controller将要显示时，调用 project查找和组合 Flutter的应用资源 bundle，并且把资源提供给 engine。  
engine在真正需要执行资源 bundle时才会创建 Dart执行的环境（懒加载，以下简称 Dart Controller），然后设置视图窗口的一些属性等东西（这是绘图引擎必需的）。  
然后 engine中的 Dart Controller会加载 Dart代码并执行，执行的过程中会调用 `dart:ui`的 native binding实现向 GPU提供数据。

# VSync信号的同步
要让视图动态化，仅仅能实现视图绘制还不行，还得知道硬件何时发送了 VSync信号，通过获取 VSync信号，计算并给 GPU提供数据来构建动态化的界面。不过每个平台的 VSync信号的获取方式不太一样，我们这里讨论一下 iOS上的实现，以此管中窥豹。   

> 源码获取于构建参见[附录]()

在 `flutter/shell/platform/darwin/ios/framework/source/FlutterView.mm`实现里面可以看到，在 UIView的 CALayer delegate中调用了 `SnapshotContentsSync`函数，这个函数会回调到 GPU线程，从 GPU线程执行获取 `LayerTree`，计算并合成位图，然后把位图信息传递给 Skia引擎，Skia引擎通过 `CGContextRef`把位图信息传递给 GPU。  

```objc
- (void)drawLayer:(CALayer*)layer inContext:(CGContextRef)context {
  SnapshotContentsSync(context, self);
}

```

```C++
// 回调到 GPU线程计算，并且这里用了一个 Dart future的 native版来同步等待 GPU线程执行结果
void SnapshotContentsSync(CGContextRef context, UIView* view) {
  auto gpu_thread = blink::Threads::Gpu();

  if (!gpu_thread) {
    return;
  }

  fxl::AutoResetWaitableEvent latch;
  gpu_thread->PostTask([&latch, context, view]() {
    SnapshotContents(context, [view isOpaque]);
    latch.Signal();
  });
  latch.Wait();
}

// SnapshotContentsSync 内容太长不赘述，精简看一下：
{
  // 获取 LayerTree
  flow::LayerTree* layer_tree = rasterizer->GetLastLayerTree();
  if (layer_tree == nullptr) {
    return;
  }
  // 获取可合成图层大小
  auto size = layer_tree->frame_size();
  if (size.isEmpty()) {
    return;
  }
  SkCanvas canvas(bitmap);

  {
    // 合成图层
    flow::CompositorContext compositor_context(nullptr);
    auto frame = compositor_context.AcquireFrame(nullptr, &canvas, false /* instrumentation */);
    layer_tree->Raster(frame, false /* ignore raster cache. */);
  }

  canvas.flush();

  // Draw the bitmap to the system provided snapshotting context.
  // 把位图使用 Skia传递给系统的 GPU缓冲区
  SkCGDrawBitmap(context, bitmap, 0, 0);
}
```

不过有一点不太确定的是，iOS的 Architecture中，并没有地方明确地提到 CALayerDelegate是与 Vsync同步的。不过可以确定的是，CALayerDelegate是并发多线程的，这个在 CATiled Layer那里可以得到体现，而 CALayer Delegate的数据确实提交给了 GPU缓冲区实现了屏幕图像的显示。


# Flutter Engine的组成 
我们现在再来回顾一下这张图（我发誓我不是为了凑内容）：

![](/images/flutter-vsync.png) 

我们找到了 VSync源，找到了 Skia把数据提交给 GPU的地方，这两处对我们来说都是黑盒，所以就先不管了。而 UI Thread的 Dart，暂时不在目前的 engine讨论范畴内。所以我们现在要分析的是给 UI 层提供能力的所有组件。  

在我的理解中，整个 Flutter Engine可以粗略地划分为三个部分：Dart UI、Runtime、Shell，我们一一道来。

## 1. Dart UI
Dart UI是一个 C++实现的 `dart:ui`库的 Native Binding，并且 UI Lib也是 Dart GUI程序的应用主要入口。  
Dart UI向上层提供了 `window`、`text`、`canvas`、`geometry`等通用的绘图能力， Runtime在调用 Dart UI时，Dart UI根据传递的 main entrypoint 来执行并且向 window渲染图像。  
值得一提的是，Dart UI还向上层提供了 `compsiting`的绘图能力，这其实就是一个 Skia的 Scene的封装，上层在调用 `compsiting`时其实就会生成或挂载节点到 LayerTree上。然后通过 LayerTree的数据结构辅助 Skia进行 Scene合成位图。  

> LayerTree是 flow库中的图层抽象类。flow 是 chrome项目中通用的绘图数据结构抽象库，可以适配到其他绘图引擎上。

## 2. Runtime 
Flutter Engine的 Runtime可谓比较复杂，并不是代码多，而是使用了非常多的 Delegate模式，将平台相关的代码交由 Shell部分实现。  

Runtime负责创建 Dart的运行时，并且在不同的开发阶段运行的环境也不一样。开发时期是保留 check mode的 Dart Snapshot VM，而生产环境是 Dart AOT Runtime。  

> Dart Snapshot VM 和 Dart JIT VM有着本质的区别。Dart Snapshot是指 token化的 Dart脚本，并非 human readable的。而 JIT VM 是直接以 script方式运行源代码。很明显 Snapshot VM要比 JIT VM稍微快一些。

![](/images/flutter-runtime.png)

上图，红框部分是在 Runtime部分执行的逻辑。engine的抽象处于 Shell层，而 ui_dart_state处于 Dart UI 层。  
我们可以看到 Runtime会由 Shell层调用生成一个 runtime controller 实例，这个实例管理着当前的绘图窗口`window`的属性，一个 Dart的VM 的实例，以及一个 delegate，这个 delegate能打通 Shell层和 Dart UI层的通信，并且负责事件的传递。

## 3. Shell
这里说的 Shell其实就是 “壳”，这个壳就是组合 Runtime、第三方工具库、平台特性等，实现调用和执行 Flutter应用的逻辑。   

1. Shell 封装了一个 engine的抽象，这个抽象能够调用 Runtime，实现 Runtime中的 Delegate，以此向 Runtime提供数据和回调。  
2. Shell 还封装了 platform view的抽象，并且具体地实现了 platform view，在 iOS特定代码中的表现就是遵循 Delegate方法并提供了 UIView实例的管理。
3. Shell 提供了一些基础工具的封装，如 Future，可以实现 `dart:io`中的 Future相同的执行逻辑，并且还负责了处理 VSync信号，UI、GPU Thread的回调。
4. Shell 提供了从 engine获取 LayerTree的和调用渲染方法的封装。

<br />

总的来说，Dart UI给 Dart提供了调用 Native绘图的能力，Runtime为 Flutter提供了调用 Dart和 Dart UI的能力，而 Shell才是集大成者，Shell将他们组合起来，并且从他们生成的数据中实现渲染。

![](/images/flutter-engine-design.png)

# 几个问题 Again  
由于代码量庞大，不说 Line by Line， File by File也是一项非常庞大的工作，所以大致了解原理后不再赘述。着重解答几个问题：

## 1. Flutter能动态更新吗？  
原版不行。理论可行。动态下发意味着 Dart源代码需要以 JIT或 JIT Snapshot的方式运行，而 Flutter的 production build是 AOT代码，所以原版不行。但 Flutter的 debug build是 JIT Snapshot运行，可以动态更新。  
那么，既要 production build，又要 JIT Snapshot执行，该如何做呢？ Flutter Engine SDK的 build option里面可以设置 mode = release， AOT = false，那么 打出来的 Engine SDK不会包含 Dart AOT Runtime。 并且需要注意 Flutter CLI TOOL的编译方式，需要以 Snapshot方式编译最终的 production代码。  
值得一提的是，JIT Snapshot方式执行性能可能稍差，60fps可能会达不到。  

## 2. Flutter SDK体积为什么非常大？  
Flutter应用的体积由两部分组成：应用代码和 SDK代码。应用代码是 Dart编译后的代码，如果做成可动态下发，那么这部分可以不计。  
SDK代码比较大就有点无奈了，SDK的组成部分有 Dart VM，Dart标准库，libwebp、libpng、libboringssl等第三方库，libskia，Dart UI库，然后再加上 icu_data，可能在单 arch下（iOS），SDK体积达到 40+MB。其中仅仅 Dart VM（不包含标准库）就达到了 7MB。  
Flutter SDK是 dynamic framework，如此大的二进制体积可能会造成动态链接耗时长。而如果静态链接，可能原来比较大的 App很有可能造成 TEXT段超标。  

## 3. Flutter可以跑多个实例吗？  
理论上是可以的。虽然 Flutter Engine的 Shell层写死了只会跑一个 Flutter View（只会跑一个 Runtime），但这是可以改变的，而且只需要少量的逻辑改动。唯一需要担心的就是多个实例的内存消耗。  

## 4. 去掉 Flutter的特性，只嵌入 Dart到应用中，可能吗？  
可行。Dart毫无疑问是一门优秀的编程语言，我也曾尝试将 Dart独立出来作为一个通用的 SDK。Dart SDK托管在 chromium项目中，并且提供了 cross building的选项。原版即提供了 Android Build脚本。但在 iOS上原版行不通，猜测主要是标准库的问题。在 Flutter iOS项目中，Dart 标准库提供了一份完全不同的实现。而且，想要把 Dart VM和标准库从 Flutter剥离出来，太困难了。并且单个 arch下，Dart VM加标准库的体积是 > 10MB的。


# The End
本期 Flutter原理的简解就到这儿，其实主要是高谈阔论了一番 Flutter Engine的实现。至于 Flutter的 UI Framework，稍候几天再水一篇吧。😜