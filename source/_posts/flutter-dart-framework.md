---
title: Flutter Dart Framework原理简解
date: 2018-05-28 16:58:19
tags:
---

![](/images/flutter-dart-framework-cover.jpg)

从前一篇文章[Flutter原理](/2018/05/14/flutter-principle/)的分析可知，Flutter Engine向 `dart:ui` Framework层提供了通用的绘图能力，Flutter渲染 UI的本质就是在 VSync信号间尽快地构建并提供视图数据。所以 Flutter整个框架的精髓在 Dart UI Framework（更准确地叫作 Flutter Dart Framework）里，这里面涉及了非常多的设计思想、优化措施，所以一篇文章是无法概括的。 所以在本篇文章中，我们只能概要性地分析一下 Flutter Dart Framework的大致原理。

<!--more-->

# UI框架绘图的基本流程
在开始本文前，我们还是要了解一下通用的 UI框架绘图的流程，以便我们理解 Flutter的 Dart Framework：

![](/images/render-pipeline.png)

从上图中我们可以看到，用户的输入才是驱动视图更新的信号。当这个信号发生后，首先需要触发的就是动画的进度更新，框架需要开始视图数据的“build”（也可以看做数据的填充）。  
在这之后，视图才会进行布局，计算各个部分的大小，然后进行“paint”，生成每个视图的视觉数据。生成的视觉数据并不能直接用，因为往往视图层级非常多， 这些数据直接向 GPU传递很低效，所以接下来需要进行视图合成，将多个视图数据合成为一个。  
最后一步进行“光栅化”，前一步得到合成的视图数据其实还是一份矢量描述数据，光栅化帮助把这份数据真正地生成一个一个的像素填充数据。在 Flutter中，光栅化这个步骤被放在了 Engine中，但是我们前文并没有提到。  

在了解 UI框架的通用流程后，我们自然而然能知道 Flutter Dart Framework这一层肯定做了以下这些事：

1. 视图的数据结构（视图树）
2. 视图数据的构建 
3. 视图布局的计算 
4. 视图的合成
5. 与 Engine的数据同步和通信


# 视图的结构
无论是比较底层的 UI框架（如 Cocoa、WebKit）还是比较上层的 （React），在向绘制引擎提供视图数据都需要一份结构化的视图数据，俗称为“视图树”(View Tree)。Flutter 的视图结构的抽象分为三部分：`Widget`, `Element`, `RenderObject`。  

> 三部分？在哪熟悉是不是？ Cocoa Touch中，视图树被分为 模型树、呈现树、渲染树， 在 Flutter中你也可以看待这三部分的意义。

## **Widget**:   
Widget是 Flutter中控件实现的基本单位，其意义类似于 Cocoa Touch中的 UIView。Widget里面存储了一个视图的配置信息，包括布局、属性等待。所以它只是一份轻量的，可直接使用的数据结构。在构建为结构树，甚至重新创建和销毁结构树时都不存在明显的性能问题。  
## **Element**:   
Element是 Widget的抽象，它其实承载了视图构建的上下文数据。构建系统通过遍历 Element树来构建 RenderObject数据，比如视图更新时，只会标记 dirty Element，而不会标记 dirty Widget。所以 Widget“无状态”，而 Element“有状态” （这个状态指框架层的构建状态）。
## **RenderObject**:  
在 RenderObject树中会发生 Layout、Paint的绘制事件，所以 Flutter中大部分的绘图性能优化发生在这里。RenderObject树构建的数据会被加入到 Engine所需的 LayerTree中，Engine通过 LayerTree进行视图合成并光栅化，提交给 GPU。  

Flutter通过这三种概念，把原本比较复杂的视图树状态、数据的维护和构建拆分得更单一、易于集中治理。


# 视图数据的构建  
这个话题其实比较大，涉及到的逻辑也很多。我们可以先从应用入口开始看起。

## 应用的 root view
Flutter App 入口的部分发生于如下代码：  

```dart
import 'package:flutter/material.dart';

// 这里的 MyApp是一个 Widget
void main() => runApp(new MyApp());

```

`runApp`函数接受一个 Widget类型的对象作为参数，也就是说在 Flutter的概念中，只存在 View，而其他的任何逻辑都只为 View的数据、状态改变服务，不存在 ViewController(或者叫 Activity）。  
接下来看 `runApp`做了什么：

```dart
void runApp(Widget app) {
  WidgetsFlutterBinding.ensureInitialized()
    ..attachRootWidget(app)
    ..scheduleWarmUpFrame();
}

class WidgetsFlutterBinding extends BindingBase with GestureBinding, ServicesBinding, SchedulerBinding, PaintingBinding, RendererBinding, WidgetsBinding {
  static WidgetsBinding ensureInitialized() {
    if (WidgetsBinding.instance == null)
      new WidgetsFlutterBinding();
    return WidgetsBinding.instance;
  }
}
```

在 `runApp`中，传入的 widget被挂载到根 widget上。这个 `WidgetsFlutterBinding`其实是一个单例，通过 mixin来使用框架中实现的其他 binding的 Service，比如 手势、基础服务、队列、绘图等等。然后会调用 `scheduleWarmUpFrame`这个方法，从这个方法注释可知，调用这个方法会主动构建视图数据。这样做的好处是因为 Flutter依赖 Dart的 MicroTask来进行帧数据构建任务的 schedule，这里通过主动调用进行整个周期的 “热身”，这样最近的下次 VSync信号同步时就有视图数据可提供，而不用等到 MicroTask的 next Tick。  

然后我们再来看 `attachRootWidget`这个函数干了什么：  

```dart
void attachRootWidget(Widget rootWidget) {
    _renderViewElement = new RenderObjectToWidgetAdapter<RenderBox>(
      container: renderView,
      debugShortDescription: '[root]',
      child: rootWidget
    ).attachToRenderTree(buildOwner, renderViewElement);
}
```

`attachRootWidget`把 widget交给了 `RenderObjectToWidgetAdapter`这座桥梁，通过这座桥梁，Element被创建，并且同时能持有 Widget和 RenderObject的引用。然后我们从上文就知道后面发生的就是第一次的视图数据构建了。   

**从这一部分能印证前面所说：Flutter应用通过 `Widget`、`Element`、`RenderObject` 三种树结构来维护整个应用的视图数据。**  


## 视图数据的构建时机  
VSync信号其实在 Flutter Dart Framework层是透明的，视图的数据构建和更新并不是由 VSync信号驱动的。我在前文说道： **Flutter并不关心显示器、视频控制器以及 GPU具体工作，它只关心 GPU发出的 VSync信号，尽可能快地在两个 VSync信号之间计算并合成视图数据，并且把数据提供给 GPU。** 这个说法其实是不准确的，准确的说法其实应该是：  

Flutter 并不关心显示器、视频控制器以及 GPU具体工作，它只关心能在 VSync信号间隔时间内尽快地合成视图数据，等待 VSync信号同步来获取这个视图数据提供给 GPU。

那么 Flutter是在什么时机构建视图数据的呢？前面就已经揭晓过答案：MicroTask 循环。  

> 是不是和 Cocoa Touch很像？NSRunloop 的 main loop 也是通过事件循环来执行固定的视图更新回调


`dart:ui`有一个抽象的 `window`，它并不负责向 LayerTree提供数据，但在 Flutter中，它承担了管理和回调视图更新方法的重担。在 rootView(rootElement)被创建时，ScheduleBinding便会调用 `scheduleFrame`，这会调用 `window.scheduleFrame`，而在 window 的 postFrameCallback 中，`scheduleFrame`会再次被调用，这样就形成了循环。  

通过这种有序的周期循环，Flutter能源源不断地提供帧视图数据，VSync信号同步时便能获得这个数据进行 Engine部分后续的操作。

## 视图数据的构建方式
前面提到，视图树被分为三类： `Widget`、`Element`、`RenderObject`，Element同时持有 Widget和 RenderObject的引用。这是本段的大前提。  
Widget是用户能够直接操作的数据结构，Flutter的设计理念中，Widget都是 immutable的，所以 Widget节点的改变其实就是节点的销毁和重新创建。但如果只是最基本的 Widget手动组合的话，写起来会很蛋疼，所以 Flutter使用 State来控制 Widget的创建与销毁，以此来达到响应式 UI编程的目的。  

> 细细品味，Widget的设计理念有没有一点和 React像？

Widget在更新后，Element持有该 Widget的节点也会被标记为 dirtyElement，那么在下一个周期的 drawFrame时，Element树的这一部分子树便会被触发 performRebuild。在 Element树更新完成后，便能获得 RenderObject树，接下来便会进入 Layout和 Paint的流程。

# 视图的布局与绘制
在 Flutter中，视图的布局与合成是整个 UI框架中最重要的部分，这部分的好坏决定了绘图的性能以及框架的应用表现，所以 Google有一个专门的 TechTalk来讲解这个原理：[Flutter's renderding pipeline](https://www.youtube.com/watch?v=UUfXWzp0-DU)  

![](/images/flutter-pipeline.png)  

## 布局的计算  
要获得每个 Widget的真实视图数据，布局是第一步。布局可以计算出每个节点所占空间的真实大小。在构建视图树时，节点的 size约束是从树顶部向底部的，而在计算布局的时候，遍历树是深度优先，所以获得布局数据的顺序是自下而上的。和 Web一样，Flutter的布局也是基于盒子模型，并且参考了众多布局方案设计而成 （毕竟负责这一部分的 Adam Barth也是 Blink项目的核心开发者)，这部分的核心设计比较复杂，我们完全可以另开一篇文章，这里不再展开。    
![](/images/layout-data-flow.png)  

## 视图的绘制 
绘制决定了一个视图节点的真实外观。和布局不太一样的是绘制的顺序，布局会优先计算子节点，而绘制会优先绘制父节点。所以在数据流上看起来两个流程的方向是相反的。

![](/images/paint-data-flow.png)  

绘制要做的事就是计算出一个 Layer的外观，这通常都不简单，因为开发者经常会在视图上放各种各样的控件，并且他们的层级也不一样，所以绘制的步骤要做的事情就是决定一个 Layer的某一部分长什么样，这也可以看做视图的局部合成。举个例子：  

![](/images/paint-into-layers.png)

在进行绘制时，每个节点都会先绘制自身，其次才是子节点。比如节点 2是一个背景色绿色的视图，在绘制完自身后，绘制子节点 3和 4，它们可能具有 alpha属性，所以绘制后当布局重叠时会合成红色的节点 5。所以从数据流方向看的时候，获得最终的 Layer的顺序反而是自下而上的。  
![](/images/paint-target-layer-flow.png)  

> 一个 Layer是一份矢量数据，挂载到 LayerTree后还会经过 Engine的合成和光栅化才能提交给 GPU。RenderObject并不是和 Layer一一对应的，所以需要 paint过程将 RenderObject转化为 Layer

## 性能优化
视图树会不断更新，这就意味着布局和绘制是不间断的。我们上面提到的布局和绘制只是九牛一毛，Flutter其实在这方面做了很多事情，不间断的布局和绘制肯定会非常耗费性能，所以 Flutter也有对应的优化方案（在 Flutter的布局和绘制算法足够优秀的前提下）。  

**边界**：Flutter使用边界标记需要重新布局和重新绘制的节点部分，这样就可以避免其他节点被污染或者触发重建，这个边界被分别叫做 Relayout boundary 和 Repaint boundary。  
<img src="/images/relayout-boundary.png" width="300px"/>
<img src="/images/repaint-boundary.png" width="300px"/> 

**缓存**：要提升性能表现，缓存也是少不了的。在 Flutter中，几乎所有的 Element都会具有一个 key，这个 key是唯一的。当子树重建后，只会刷新 key不同的部分。而节点数据的复用就是依靠 key来从缓存中取得。

> Flutter 的渲染机制可以当做一本教科书，本文只能管中窥豹了


## 挂载 LayerTree
`Layer`的抽象在 Flutter中是真实存在的，`RenderObject`在经过 paint生成 Layer后，会通过 `dart:ui`的 `SceneBuilder`挂载到 LayerTree。

```dart
  /// Override this method to upload this layer to the engine.
  ///
  /// The `layerOffset` is the accumulated offset of this layer's parent from the
  /// origin of the builder's coordinate system.
  void addToScene(ui.SceneBuilder builder, Offset layerOffset);
```

这是一个抽象接口，不同的 Layer（如 ContainerLayer、TextureLayer等）在继承抽象类后自行处理逻辑，但无外乎都会最终调用 `builder.addXXX`这类方法，把传递的 Layer真正地挂载上去。

> 接下来的事情上一篇文章也提过了，Skia在 VSync信号同步时直接从 LayerTree合成 Bitmap，（经过光栅化后）提交给 GPU


# Flutter 的通信机制
读完上面大概能了解到 Flutter怎么通过固定的数据结构构建并渲染丰富多彩的视图。但文章开头说过，用户输入才是驱动视图更新的源泉，所以需要专门讨论一下 Flutter是如何获得用户输入并与平台通信的。  

## 用户输入 
在当今的移动设备上传感器有很多，屏幕输入也可以算作一种。所以这里以讨论手势输入为例。  
Flutter通过 Native方法捕捉用户的屏幕输入，这些数据被封装成一个一个的数据包，通过 Dart的 runtime hook发送给 `dart:ui`中的 `_dispatchPointerDataPacket`方法。这个方法会直接将屏幕输入数据分发给 `window`，所以在 Flutter中，我们可以将触摸时间看作发生在全局的 `window`上。

```c++

// Shell
// 将封装好的手势数据 packet发送给 engine
blink::Threads::UI()->PostTask(fxl::MakeCopyable(
      [ engine = _platformView->engine().GetWeakPtr(), packet = std::move(packet) ] {
        if (engine.get())
          engine->DispatchPointerDataPacket(*packet);
      }));
      

// engine直接 通过 libtonic hook Dart的 _dispatchPointerDataPacket 方法将数据包传递过去
void RuntimeController::DispatchPointerDataPacket(
    const PointerDataPacket& packet) {
  TRACE_EVENT1("flutter", "RuntimeController::DispatchPointerDataPacket",
               "mode", "basic");
  GetWindow()->DispatchPointerDataPacket(packet);
}

void Window::DispatchPointerDataPacket(const PointerDataPacket& packet) {
  tonic::DartState* dart_state = library_.dart_state().get();
  if (!dart_state)
    return;
  tonic::DartState::Scope scope(dart_state);

  Dart_Handle data_handle = ToByteData(packet.data());
  if (Dart_IsError(data_handle))
    return;
  DartInvokeField(library_.value(), "_dispatchPointerDataPacket",
                  {data_handle});
}
```

Flutter在接收到数据包后，识别触摸点发生的坐标，通过数学计算来识别手势。这和 Native框架的逻辑几乎是一样的。

## Native通信
但除了手势这种特定的输入，还有其他传感器和事件的传递被抽象得更通用，叫做 `MethodChannel`。这样就可以避免用户每次需要开发一个 Native extension时都要写 Native Binding了。 MethodChannel的原理和手势的传递几乎是一样的，但是 API会更通用一点。  
![](/images/PlatformChannels.png)  

在这篇教程中 [PlatformChannels](https://flutter.io/platform-channels)示例了一个如何通过 Native 获取电池电量，并通过 PlatformChannel传递给 Flutter应用。而这个 PlatformChannel像手势数据分发那样，换了一个方法：  

```c++
void Window::DispatchPlatformMessage(fxl::RefPtr<PlatformMessage> message) {
  tonic::DartState* dart_state = library_.dart_state().get();
  if (!dart_state)
    return;
  tonic::DartState::Scope scope(dart_state);
  Dart_Handle data_handle =
      (message->hasData()) ? ToByteData(message->data()) : Dart_Null();
  if (Dart_IsError(data_handle))
    return;

  int response_id = 0;
  if (auto response = message->response()) {
    response_id = next_response_id_++;
    pending_responses_[response_id] = response;
  }

  DartInvokeField(
      library_.value(), "_dispatchPlatformMessage",
      {ToDart(message->channel()), data_handle, ToDart(response_id)});
}

```

通过这种打开一个 Channel来进行 request/response 的抽象概念可以实现任何用户自定义的 extension。


# The End 
Flutter是 engine和 Dart Framework的统称，目前 Flutter还处于 beta阶段，虽然说大致原理不会变，但一些细节部分还是会有不少改动。本文只能看个大概，因为，信息量实在是太大了。  
好了，原理分析告一段落，在后面的使用中，我们再来细细探讨 Flutter的一些设计理念。




