---
title: Brief explanation of Flutter Dart Framework principle
date: 2018-05-28 16:58:19
tags:
---

![](/images/flutter-dart-framework-cover.jpg)

From the analysis of the previous article [Flutter Principle](/2018/05/14/flutter-principle/), it can be seen that Flutter Engine provides general drawing capabilities to the `dart:ui` Framework layer. The essence of Flutter rendering UI is in VSync The signal room is constructed as soon as possible and provides view data. Therefore, the essence of Flutter's entire framework is in the Dart UI Framework (more accurately called Flutter Dart Framework), which involves a lot of design ideas and optimization measures, so an article cannot be generalized. So in this article, we can only briefly analyze the general principles of Flutter Dart Framework.

<!--more-->

# The basic process of UI frame drawing
Before starting this article, we still have to understand the general UI framework drawing process so that we can understand Flutter's Dart Framework:

![](/images/render-pipeline.png)

From the figure above, we can see that user input is the signal that drives the view update. When this signal occurs, the first thing that needs to be triggered is the progress update of the animation. The framework needs to start the "build" of the view data (it can also be regarded as the data filling).
After that, the view will be laid out, the size of each part will be calculated, and then “paint” will be performed to generate visual data for each view. The generated visual data cannot be used directly, because there are often many view levels, and it is inefficient to transmit these data directly to the GPU. Therefore, the next step is to perform view synthesis to synthesize multiple view data into one.
The last step is "rasterization". The view data synthesized in the previous step is actually a vector description data. Rasterization helps to truly generate pixel filling data one by one. In Flutter, the rasterization step is placed in the Engine, but we did not mention it before.

After understanding the general process of the UI framework, we can naturally know that the Flutter Dart Framework must do the following things:

1. View data structure (view tree)
2. View data construction
3. Calculation of view layout
4. View composition
5. Data synchronization and communication with Engine


# View structure
Whether it is a relatively low-level UI framework (such as Cocoa, WebKit) or a relatively high-level (React), providing view data to the rendering engine requires a structured view data, commonly known as the "View Tree". The abstraction of Flutter's view structure is divided into three parts: `Widget`, ʻElement`, `RenderObject`.

> Three parts? Where are you familiar? In Cocoa Touch, the view tree is divided into model tree, presentation tree, and rendering tree. In Flutter, you can also look at the meaning of these three parts.

## **Widget**:
Widget is the basic unit of control implementation in Flutter, and its meaning is similar to UIView in Cocoa Touch. The Widget stores the configuration information of a view, including layout and attribute waiting. So it is just a lightweight and directly usable data structure. There is no obvious performance problem when constructing as a structure tree, or even recreating and destroying the structure tree.
## **Element**:
Element is an abstraction of Widget, which actually carries the context data for view construction. The construction system builds RenderObject data by traversing the Element tree. For example, when the view is updated, it will only mark the dirty Element and not the dirty Widget. So Widget is "stateless" and Element is "stateful" (this state refers to the construction state of the framework layer).
## **RenderObject**:
Layout and Paint drawing events will occur in the RenderObject tree, so most of the drawing performance optimizations in Flutter happen here. The data constructed by the RenderObject tree will be added to the LayerTree required by the Engine, and the Engine will synthesize and rasterize the view through the LayerTree, and submit it to the GPU.

Flutter uses these three concepts to split the originally more complex view tree state, data maintenance and construction into a single, easy to centralized management.


# View data construction
This topic is actually quite large and involves a lot of logic. We can start with the application entrance.

## App root view
The entry part of Flutter App occurs in the following code:

```dart
import'package:flutter/material.dart';

// Here MyApp is a Widget
void main() => runApp(new MyApp());

```

The `runApp` function accepts a Widget type object as a parameter. That is to say, in the concept of Flutter, there is only View, and any other logic only serves the data and state changes of the View. There is no ViewController (or Activity). .
Let's see what `runApp` does:

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

In `runApp`, the incoming widget is mounted on the root widget. This `WidgetsFlutterBinding` is actually a singleton, which uses mixin to use other binding services implemented in the framework, such as gestures, basic services, queues, drawing, and so on. Then the method `scheduleWarmUpFrame` will be called. From the annotation of this method, we can know that calling this method will actively construct the view data. The advantage of this is that Flutter relies on Dart's MicroTask to carry out the schedule of the frame data construction task. Here, the whole cycle of "warm-up" is actively called, so that the view data can be provided when the next VSync signal is synchronized recently, instead of Wait until the next Tick of MicroTask.

Then let's look at what the `attachRootWidget` function does:

```dart
void attachRootWidget(Widget rootWidget) {
    _renderViewElement = new RenderObjectToWidgetAdapter<RenderBox>(
      container: renderView,
      debugShortDescription:'[root]',
      child: rootWidget
    ).attachToRenderTree(buildOwner, renderViewElement);
}
```

ʻAttachRootWidget` hands the widget to the bridge `RenderObjectToWidgetAdapter`, through which the Element is created and can hold references to Widget and RenderObject at the same time. Then we know from the above that what happens next is the first view data construction.

**From this part, it can be confirmed that the Flutter application maintains the view data of the entire application through the three tree structures of `Widget`, ʻElement`, and `RenderObject`. **


## When to build view data
The VSync signal is actually transparent at the Flutter Dart Framework layer, and the data construction and update of the view is not driven by the VSync signal. I said in the previous article: **Flutter does not care about the specific work of the display, video controller and GPU. It only cares about the VSync signal sent by the GPU. It calculates and synthesizes the view data between the two VSync signals as quickly as possible, and combines the data Provided to the GPU. ** This statement is actually inaccurate. The accurate statement should actually be:

Flutter does not care about the specific work of the display, video controller, and GPU. It only cares about synthesizing the view data as soon as possible within the VSync signal interval, and waiting for the VSync signal to synchronize to obtain this view data and provide it to the GPU.

So when did Flutter build the view data? The answer has been revealed before: MicroTask loop.

> Is it similar to Cocoa Touch? The main loop of NSRunloop also executes a fixed view update callback through the event loop


`dart:ui` has an abstract `window`, which is not responsible for providing data to LayerTree, but in Flutter, it bears the burden of management and callback view update methods. When rootView (rootElement) is created, ScheduleBinding will call `scheduleFrame`, which will call `window.scheduleFrame`, and in the postFrameCallback of window, `scheduleFrame` will be called again, thus forming a loop.

Through this orderly cycle, Flutter Energy continuously provides frame view data, which can be obtained when the VSync signal is synchronized for subsequent operations in the Engine part.

## How to build view data
As mentioned earlier, the view tree is divided into three categories: `Widget`, ʻElement`, and `RenderObject`. Element holds references to both Widget and RenderObject. This is the major premise of this paragraph.
Widget is a data structure that users can directly manipulate. In Flutter's design concept, Widgets are immutable, so the change of Widget nodes is actually the destruction and re-creation of nodes. But if it is just the most basic manual combination of Widgets, it will be painful to write, so Flutter uses State to control the creation and destruction of Widgets to achieve the purpose of responsive UI programming.

> Savour carefully, is the design concept of Widget similar to React?

After the Widget is updated, the node holding the Widget by the Element will also be marked as dirtyElement. Then in the next cycle of drawFrame, this part of the element tree will trigger performRebuild. After the Element tree is updated, the RenderObject tree can be obtained, and then the process of Layout and Paint will be entered.

# View layout and drawing
In Flutter, the layout and composition of views are the most important part of the entire UI framework. The quality of this part determines the performance of the drawing and the application performance of the framework, so Google has a special TechTalk to explain this principle: [Flutter's renderding pipeline](https://www.youtube.com/watch?v=UUfXWzp0-DU)

![](/images/flutter-pipeline.png)

## Layout calculation
To get the actual view data of each Widget, layout is the first step. The layout can calculate the true size of the space occupied by each node. When constructing the view tree, the size constraint of the node is from the top to the bottom of the tree, and when calculating the layout, traversing the tree is depth first, so the order of obtaining the layout data is bottom-up. Like the Web, the layout of Flutter is also based on the box model, and is designed with reference to many layout schemes (after all, Adam Barth, who is responsible for this part, is also the core developer of the Blink project). The core design of this part is more complicated. Open an article and I won’t expand it here.
![](/images/layout-data-flow.png)

## View drawing
Drawing determines the true appearance of a view node. Different from the layout is the drawing order. The layout will give priority to the calculation of child nodes, and the drawing will give priority to drawing the parent nodes. So it seems that the directions of the two processes are opposite in the data flow.

![](/images/paint-data-flow.png)

What you have to do to draw is to calculate the appearance of a Layer. This is usually not easy, because developers often put various controls on the view, and their levels are different, so the drawing steps are to be done The thing is to determine what a certain part of a Layer looks like, which can also be seen as a partial synthesis of the view. for example:  

![](/images/paint-into-layers.png)

When drawing, each node draws itself first, followed by child nodes. For example, node 2 is a view with a green background. After drawing itself, the child nodes 3 and 4 may be drawn. They may have alpha attributes, so when the layout overlaps after drawing, the red node 5 will be synthesized. So when looking at the data flow direction, the order of obtaining the final Layer is bottom-up instead.
![](/images/paint-target-layer-flow.png)

> A Layer is a piece of vector data. After being mounted on the LayerTree, it will be synthesized and rasterized by the Engine before it can be submitted to the GPU. RenderObject does not correspond to Layer one-to-one, so the paint process is needed to convert RenderObject to Layer

## Performance optimization
The view tree is constantly updated, which means that the layout and drawing are uninterrupted. The layout and drawing we mentioned above are just a drop in the bucket. Flutter has actually done a lot in this regard. Uninterrupted layout and drawing will definitely cost performance, so Flutter also has corresponding optimization solutions (the layout and drawing algorithms in Flutter are good enough The premise).

**Boundary**: Flutter uses the boundary to mark the part of the node that needs to be re-layouted and repainted, so as to avoid other nodes from being polluted or triggering reconstruction. This boundary is called Relayout boundary and Repaint boundary, respectively.
<img src="/images/relayout-boundary.png" width="300px"/>
<img src="/images/repaint-boundary.png" width="300px"/>

**Cache**: To improve performance, cache is also indispensable. In Flutter, almost all Elements will have a key, which is unique. When the subtree is rebuilt, only the parts with different keys will be refreshed. The reuse of node data is to rely on the key to obtain from the cache.

> Flutter's rendering mechanism can be used as a textbook, this article can only be a glimpse of the leopard


## Mount LayerTree
The abstraction of `Layer` is real in Flutter. After the layer is generated by paint, the `RenderObject` will be mounted to the LayerTree through the `SceneBuilder` of `dart:ui`.

```dart
  /// Override this method to upload this layer to the engine.
  ///
  /// The `layerOffset` is the accumulated offset of this layer's parent from the
  /// origin of the builder's coordinate system.
  void addToScene(ui.SceneBuilder builder, Offset layerOffset);
```

This is an abstract interface. Different Layers (such as ContainerLayer, TextureLayer, etc.) handle their own logic after inheriting the abstract class, but they will eventually call methods like `builder.addXXX` to actually mount the passed Layer. .

> The next thing I mentioned in the previous article, Skia directly synthesizes the Bitmap from LayerTree when the VSync signal is synchronized, and submits it to the GPU (after rasterization)


# Flutter's communication mechanism
After reading the above, you can probably understand how Flutter builds and renders colorful views through a fixed data structure. But as mentioned at the beginning of the article, user input is the source of driving view updates, so we need to specifically discuss how Flutter obtains user input and communicates with the platform.

## User input
There are many sensors on today's mobile devices, and screen input can also be counted as one. So here to discuss gesture input as an example.
Flutter captures the user's screen input through the Native method. These data are encapsulated into data packets and sent to the `_dispatchPointerDataPacket` method in `dart:ui` through Dart's runtime hook. This method will directly distribute the screen input data to the `window`, so in Flutter, we can regard the touch time as occurring on the global `window`.

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

After receiving the data packet, Flutter recognizes the coordinates of the touch point and recognizes the gesture through mathematical calculation. This is almost the same logic as the Native framework.

## Native Communication
But in addition to the specific input of gestures, there are other sensors and event delivery that are abstracted more generically, called `MethodChannel`. In this way, users can avoid writing Native Binding every time they need to develop a Native extension. The principle of MethodChannel is almost the same as the transmission of gestures, but the API will be more general.
![](/images/PlatformChannels.png)

In this tutorial [PlatformChannels](https://flutter.io/platform-channels) demonstrates how to obtain battery power through Native and pass it to the Flutter application through PlatformChannel. And this PlatformChannel, like gesture data distribution, has a different method:

```c++
void Window::DispatchPlatformMessage(fxl::RefPtr<PlatformMessage> message) {
  tonic::DartState* dart_state = library_.dart_state().get();
  if (!dart_state)
    return;
  tonic::DartState::Scope scope(dart_state);
  Dart_Handle data_handle =
      (message->hasData())? ToByteData(message->data()): Dart_Null();
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

Any user-defined extension can be realized through this abstract concept of opening a Channel for request/response.


# The End
Flutter is the collective name of engine and Dart Framework. Currently Flutter is still in the beta stage. Although the general principle will not change, some details will still have many changes. This article can only look at a rough idea, because the amount of information is too large.
Well, the principle analysis is over. In the following use, we will discuss some of Flutter's design concepts in detail.