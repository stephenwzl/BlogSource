---
title: Brief explanation of Flutter principle
date: 2018-05-14 11:28:23
tags:
---

![](/images/flutter.png)
Flutter is a mobile application development framework launched and open sourced by Google, focusing on cross-platform, high fidelity, and high performance. Developers can develop apps through Dart language, and a set of codes runs on both iOS and Android platforms. Flutter provides a wealth of components and interfaces, and developers can quickly add native extensions to Flutter. At the same time, Flutter also uses the Native engine to render the view, which can undoubtedly provide users with a good experience.
<!--more-->

Okay, the above official speech is finished, let's start to understand the implementation principle of Flutter.

# Basic principles of drawing
When it comes to principles, we have to start with the basic principles of screen display images.
We all know that the display refreshes at a fixed frequency, such as 60Hz for iPhone and 120Hz for iPad Pro. When a frame of image is drawn and ready to draw the next frame, the monitor will send out a vertical synchronization signal (VSync), so a 60Hz screen will send out such a signal 60 times in one second.
And generally speaking, in a computer system, the CPU, GPU, and display cooperate in a specific way: the CPU submits the calculated display content to the GPU, and the GPU puts it into the frame buffer after rendering, and then the video controller follows the VSync signal Fetch frame data from the frame buffer and pass it to the display for display.


![Picture from ibireme.com](/images/screen_display.png)


> The picture above is from ibireme.com

Therefore, Android and iOS apps display UI in this way, and Flutter is no exception, and it also follows this pattern. So from this we can see the essential difference between Flutter and React-Native: React-Native and the like only extend and call OEM components, while Flutter renders itself.
In the explanation of Flutter Architecture, Google also provides a more detailed diagram to explain the principle of Flutter:

![](/images/flutter-vsync.png)

This picture explains more clearly: Flutter only cares about providing view data to the GPU, the VSync signal of the GPU is synchronized to the UI thread, and the UI thread uses Dart to build an abstract view structure. This data structure is used for layer synthesis on the GPU thread. View data is provided to the Skia engine for rendering as GPU data, which is provided to the GPU through OpenGL or Vulkan.
   
Therefore, Flutter does not care about the specific work of the display, video controller, and GPU. It only cares about the VSync signal sent by the GPU, calculates and synthesizes the view data between the two VSync signals as quickly as possible, and provides the data to the GPU.

# a few questions  
After understanding the basic concepts of Flutter, there are naturally several questions that need to be resolved.

## 1. Why use Dart?
This is a very interesting question. Flutter chose Dart instead of JavaScript. I think there are mainly the following reasons:

1. Dart has better performance. In the JIT mode, Dart has basically the same speed as JavaScript. But Dart supports AOT. When running in AOT mode, JavaScript is far behind. The speed increase is very helpful for view data calculation under high frame rate.
2. Native Binding. On Android, v8's Native Binding can be implemented very well, but JavaScriptCore on iOS cannot, so if JavaScript is used, the code mode of the Flutter basic framework is difficult to unify. And Dart's Native Binding can be implemented through Dart Lib.
3. Fuchsia OS, a reason that doesn't look like a cause. The built-in application browser of Fuchsia OS uses Dart language as the App development language. And in fact, Flutter is a conceptual subset of Fuchisa OS's application framework. (Flutter source code and compilation tool chain are also full of Fuchsia macros)
4. Dart is a **type-safe** language, with complete package management and many features. Google has gathered so many design experts in the programming language community to develop such a language, which aims to replace JavaScript, so Fuchsia OS has Dart built in. Dart can be used as embedded lib to embed applications, instead of getting updates only with system upgrades, this is also one of the advantages.

## 2. What is Skia?
As mentioned earlier, Flutter only cares about how to build the view abstract structure and provide view data to the GPU. Skia is the way Flutter provides data to the GPU.

Skia is a 2D drawing engine library. Its predecessor is a vector drawing software. Both Chrome and Android use Skia as the drawing engine. Skia provides a very friendly API, and provides friendly and efficient performance in graphics conversion, text rendering, and bitmap rendering. Skia is cross-platform, so it can be embedded in Flutter's iOS SDK without having to study the iOS closed-source Core Graphics / Core Animation.

> Android comes with Skia, so the Flutter Android SDK is much smaller than the iOS SDK.

## 3. How is Flutter designed?
Having said the basics for so long, we may only know what Flutter has done, but have not yet observed the entire architecture design of Flutter from the side, and understand how Flutter does it.

![](/images/flutter-framework.png)

People who know Flutter in this picture may have seen it in many places. Here is a detailed explanation:

**Flutter Framework**: This is a pure Dart-implemented SDK, similar to the role of React in JavaScript. It implements a set of basic libraries for handling animation, drawing and gestures. It also encapsulates a set of UI component libraries based on drawing, and then distinguishes them according to the two visual styles of `Material` and `Cupertino`. This pure Dart SDK is packaged as a Dart library called `dart:ui`. When we use Flutter to write App, we can directly import this library to use components and other functions.

**Flutter Engine**: This is a pure C++ SDK, which includes Skia engine, Dart runtime, text typesetting engine, etc. But to put it bluntly, it is a runtime of Dart, it can run Dart code in JIT, JIT Snapshot or AOT mode. When the code calls the `dart:ui` library, provide the Native Binding implementation in the `dart:ui` library. But don't forget, this runtime also controls the transmission of VSync signals, the filling of GPU data, etc., and is also responsible for passing client events to the code in the runtime. <br />
** After understanding the basic principles of screen drawing and an overall concept of Flutter, let's take a detailed look at the approximate implementation of Flutter. **

> As my Android knowledge is not good, I only analyze the implementation on the iOS platform. Android can refer to this article to understand the code


# Flutter application running

To understand the principle of Flutter, we start from the entry point and look at the Flutter code. Since the application frameworks are similar, the code of Flutter mentioned below refers to the code of Flutter Engine, not the code of Flutter Dart Framework.

The following figure shows me briefly sorted out the execution sequence of the Flutter application after startup
![](/images/flutter-project.png)

After the application's View Controller is initialized, an abstraction of the Flutter project (hereinafter referred to as project) will be instantiated. The project initializes an abstract instance of the platform view, which is responsible for creating the Flutter runtime (hereinafter referred to as the engine).

When the View Controller is about to display, call project to find and combine Flutter's application resource bundles, and provide the resources to the engine.
The engine will only create a Dart execution environment (lazy loading, hereinafter referred to as Dart Controller) when it really needs to execute the resource bundle, and then set some properties of the view window (this is necessary for the drawing engine).
Then the Dart Controller in the engine will load the Dart code and execute it. During the execution, the native binding implementation of `dart:ui` will be called to provide data to the GPU.

# VSync signal synchronization
To make the view dynamic, it is not enough to only be able to draw the view. You have to know when the hardware sends the VSync signal. By obtaining the VSync signal, calculating and providing data to the GPU to build a dynamic interface. However, the way to obtain the VSync signal of each platform is different. Let's discuss the implementation on iOS to get a glimpse of the leopard.

> The source code is obtained in the build see [Appendix](#Appendix)

In the implementation of `flutter/shell/platform/darwin/ios/framework/source/FlutterView.mm`, you can see that the `SnapshotContentsSync` function is called in the CALayer delegate of UIView. This function will call back to the GPU thread, from the GPU thread Execute to obtain `LayerTree`, calculate and synthesize the bitmap, and then pass the bitmap information to the Skia engine, and the Skia engine passes the bitmap information to the GPU through the `CGContextRef`.

``ʻobjc
-(void)drawLayer:(CALayer*)layer inContext:(CGContextRef)context {
  SnapshotContentsSync(context, self);
}

```

```C++
// Call back to GPU thread calculation, and here a native version of Dart future is used to synchronously wait for the GPU thread execution result
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

// The content of SnapshotContentsSync is too long, so I won’t repeat it, let’s take a brief look:
{
  // Get LayerTree
  flow::LayerTree* layer_tree = rasterizer->GetLastLayerTree();
  if (layer_tree == nullptr) {
    return;
  }
  // Get the size of composable layer
  auto size = layer_tree->frame_size();
  if (size.isEmpty()) {
    return;
  }
  SkCanvas canvas(bitmap);

  {
    // Composite layer
    flow::CompositorContext compositor_context(nullptr);
    auto frame = compositor_context.AcquireFrame(nullptr, &canvas, false /* instrumentation */);
    layer_tree->Raster(frame, false /* ignore raster cache. */);
  }

  canvas.flush();

  // Draw the bitmap to the system provided snapshotting context.
  // Pass the bitmap to the GPU buffer of the system using Skia
  SkCGDrawBitmap(context, bitmap, 0, 0);
}
```

However, one thing that is not sure is that in the iOS Architecture, there is no clear mention that CALayerDelegate is synchronized with Vsync. But what is certain is that CALayerDelegate is concurrent and multithreaded, which can be reflected in CATiled Layer, and the data of CALayer Delegate is indeed submitted to the GPU buffer to display the screen image.


# Flutter Engine composition
Let's review this picture now (I swear I am not trying to make up the content):

![](/images/flutter-vsync.png)

We found the VSync source, and found the place where Skia submits the data to the GPU. Both of these are black boxes for us, so we don't care. And UI Thread's Dart is temporarily out of the scope of the current engine discussion. So what we want to analyze now is all the components that provide capabilities to the UI layer.

In my understanding, the entire Flutter Engine can be roughly divided into three parts: Dart UI, Runtime, and Shell, let's come together.

## 1. Dart UI
Dart UI is a Native Binding of the `dart:ui` library implemented in C++, and UI Lib is also the main entry point for Dart GUI applications.
Dart UI provides general drawing capabilities such as `window`, `text`, `canvas`, and `geometry` to the upper layer. When Runtime calls Dart UI, Dart UI executes according to the passed main entrypoint and renders images to window.
It is worth mentioning that Dart UI also provides the upper layer with the drawing ability of `compsiting`, which is actually an encapsulation of Skia's Scene. When the upper layer calls `compsiting`, it will actually generate or mount nodes to LayerTree. Then Skia is assisted in Scene synthesis bitmap through the data structure of LayerTree.

> LayerTree is the layer abstract class in the flow library. flow is a general drawing data structure abstract library in the chrome project, which can be adapted to other drawing engines.

## 2. Runtime
The Runtime of Flutter Engine can be said to be more complicated. It is not that there are too many codes, but it uses a lot of Delegate mode, and the platform-related code is partly implemented by the Shell.

Runtime is responsible for creating the Dart runtime, and the environment for running in different development stages is different. During the development period, the Dart Snapshot VM was kept in check mode, and the production environment was Dart AOT Runtime.

> Dart Snapshot VM and Dart JIT VM are fundamentally different. Dart Snapshot refers to tokenized Dart scripts, not human readable. The JIT VM runs the source code directly in script mode. Obviously Snapshot VM is slightly faster than JIT VM.

![](/images/flutter-runtime.png)

In the above figure, the red box part is the logic executed in the Runtime part. The engine abstraction is in the Shell layer, and ui_dart_state is in the Dart UI layer.
We can see that Runtime will be called by the Shell layer to generate a runtime controller instance. This instance manages the properties of the current drawing window `window`, a Dart VM instance, and a delegate, which can connect the Shell layer and the Dart UI Layer communication, and is responsible for the delivery of events.

## 3. Shell
The Shell mentioned here is actually a "shell". This shell is a combination of Runtime, third-party tool libraries, platform features, etc., to implement the logic of calling and executing Flutter applications.

1. Shell encapsulates an engine abstraction. This abstraction can call Runtime and implement the Delegate in Runtime to provide data and callbacks to Runtime.
2. Shell also encapsulates the abstraction of platform view, and specifically implements platform view. The performance in iOS specific code is to follow the Delegate method and provide the management of UIView instances.
3. Shell provides encapsulation of some basic tools, such as Future, which can implement the same execution logic of Future in `dart:io`, and is also responsible for processing VSync signals, UI, and GPU Thread callbacks.
4. Shell provides a package for obtaining LayerTree from the engine and calling rendering methods.

<br />

In general, Dart UI provides Dart with the ability to call Native drawing, Runtime provides Flutter with the ability to call Dart and Dart UI, and Shell is the master. Shell combines them and generates data from them. Realize rendering.

![](/images/flutter-engine-design.png)

# Several questions Again
Due to the huge amount of code, File by File is also a very large job, not to mention Line by Line, so I won't repeat it after understanding the principle. Focus on answering a few questions:

## 1. Can Flutter be dynamically updated?
The original version does not work. The theory is feasible. Dynamic distribution means that Dart source code needs to be run in JIT or JIT Snapshot mode, and Flutter's production build is AOT code, so the original version does not work. But Flutter's debug build is run by JIT Snapshot and can be updated dynamically.
So, how to do both production build and JIT Snapshot execution? You can set mode = release and AOT = false in the build option of Flutter Engine SDK, then the typed Engine SDK will not include Dart AOT Runtime. And you need to pay attention to the compilation method of Flutter CLI TOOL, you need to compile the final production code in Snapshot mode.
It is worth mentioning that the execution performance of JIT Snapshot method may be slightly worse, and 60fps may not be reached.

## 2. Why is the Flutter SDK very large?
The volume of Flutter application consists of two parts: application code and SDK code. The application code is the code compiled by Dart. If it is made to be dynamically distributed, then this part can be ignored.
The SDK code is relatively large, it is a bit helpless. The components of the SDK include Dart VM, Dart standard library, libwebp, libpng, libboringssl and other third-party libraries, libskia, Dart UI library, and then add icu_data. It may be under single arch (iOS ), the SDK volume reaches 40+MB. Among them, only Dart VM (not including the standard library) reached 7MB.
Flutter SDK is a dynamic framework. Such a large binary volume may cause dynamic linking to take a long time. And if it is statically linked, the larger App may cause the TEXT segment to exceed the standard.

## 3. Can Flutter run multiple instances?
In theory it is possible. Although the Shell layer of the Flutter Engine is hard-coded, it will only run a Flutter View (only a Runtime), but this can be changed, and only a few logic changes are required. The only thing to worry about is the memory consumption of multiple instances.

## 4. Is it possible to remove the features of Flutter and only embed Dart in the application?
feasible. Dart is undoubtedly an excellent programming language, and I have tried to separate Dart as a universal SDK. The Dart SDK is hosted in the chromium project and provides the option of cross building. The original version provides the Android Build script. However, the original version does not work on iOS, and it is guessed that it is mainly a problem with the standard library. In the Flutter iOS project, the Dart standard library provides a completely different implementation. Moreover, it is too difficult to separate the Dart VM and standard library from Flutter. And under a single arch, the volume of Dart VM plus standard library is> 10MB.


# The End
The brief explanation of Flutter principle in this issue is here. In fact, it is mainly to talk about the implementation of Flutter Engine. As for Flutter's UI Framework, please wait a few days for another article. 😜

# Appendix
**Build Flutter Engine (for iOS)**
1. Fork the ʻengine` project
2. Setting up the development environment
    * Mac (Xcode 9.0)
    * Python >= 2.7.10
    * Install depot_tools (Google Toolchain)
        
        ```
        git clone https://chromium.googlesource.com/chromium/tools/depot_tools.git
        ```
        Modify .bashrc (zsh modify .zshrc), add depot_tools to the environment variable
        
        ```
        export PATH=$PATH:<Replace the directory path here with your address>/depot_tools
        ```
        After the above operations are completed, restart the command line client to make the settings take effect
    * Make sure your command line has tool commands `curl` and `wget`
    * If you are using a Mac, make sure to install `brew install ant`

3. Pull your engine project locally, the storage directory is called engine
4. Create a file `.gclient` in the engine directory, and fill in the content as follows:
    ```
    solutions = [
        {
            "managed": False,
            "name": "src/flutter",
            "url": "git@github.com:<here to replace your github ID>/engine.git",
            "custom_deps": {},
            "deps_file": "DEPS",
            "safesync_url": "",
        },
    ]
    ```
5. Execute the command `gclient sync` in the ʻengine` directory, this operation requires a command line network proxy, it is recommended to use ShadowSocksX-NG
6. After step 5 is completed, there will be an additional src directory under the engine directory, this directory is where the code is actually written and compiled. In this directory, add the git upstream source:
    ```
    git remote add upstream https://github.com/flutter/engine.git
    ```
    Before proceeding to the next step, make sure that the code is up to date, perform fetch -> pull rebase upstream master

7. Under the src directory, execute the command: `./flutter/tools/gn --ios --simulator --unoptimized` to generate the compiled file
8. In the src directory, execute the compilation command: `ninja -C out/ios_debug_sim_unopt`
9. After the compilation is complete, you can find the `Flutter.framework` file in the ʻout/ios_debug_sim_unopt` directory, and you can integrate it into the iOS project
10. Open ʻall.xcworkspace` to view the Flutter source code