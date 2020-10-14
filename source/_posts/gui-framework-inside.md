---
title: GUI Framework Inside
date: 2018-11-06 16:10:59
tags:
---

> Reminder before reading: This article is only a personal opinion, any similarity is purely coincidental, please correct me if there is any error

GUI has been developed for almost 30 years. Now this technology has basically matured. Various GUI frameworks are basically the same. Below is a list of popular GUI frameworks:

* Cocoa/Cocoa Touch (Apple, macOS/iOS)
* Windows Presentation Foundation (Microsoft, Windows)
* Android GUI (Google, Android)
* WebKit (Apple, Safari)
* Blink (Google, Chrome)
* Flutter (Google, Android/iOS/Fuschia/Chrome)

<!--more-->

> Movies and displays:

>Film is one of the greatest inventions in modern times. Its principle is the "persistence of vision" of the human eye. After the visual phase of an object disappears, it can stay in the retina for a short time for 0.1-0.4s. When rotating, a series of static images will create a continuous visual impression due to visual persistence.

>Modern displays generally use a refresh rate of 60Hz (>=60), which requires the display processor (GPU) to provide 60 displayable image data within 1s. When the display processor is too late to process so much, it can only provide When the number is less than 60, many people can feel the freeze (frame drop), when the value is less than 30 or even less than 24, basically everyone can feel the freeze (frame drop).


# General form of GUI framework
GUI is an extension of movie (animation). It shows the user an interactive interface in the form of animation and provides visual feedback to the user's operation.
It can be known from the above that the GUI framework should provide the GPU with at least 60 static images within 1s, and the GUI framework itself also needs to complete a certain amount of calculations within 1s to complete the user's interaction requirements.

**The basic GUI unit for user operations is controls**. Buttons are controls, pictures are controls, and even windows are controls. The application provides users with a rich and colorful interactive interface through the arrangement and combination of controls.
**Controls are also the basic unit of rendering in GUI frameworks**. The GUI framework describes the data structure of the interface through the control tree, and determines the size and appearance through the style properties of the controls.

In the early GUI framework, after designing different controls, let the controls decide the drawing style. Although there is no problem in doing so, it is usually not efficient enough. Different controls will have a lot of similar drawing codes (code redundancy), and the GPU is exhausted when actually drawing, and the drawing cycle of each control is relatively independent, it is difficult to control and locate the problem from the framework as a whole.
After years of iteration and development, the GUI framework has developed into a similar structure today:
![Modern GUI Frame Structure](https://ws1.sinaimg.cn/large/8696f529ly1fwx63r4baaj20ci0ci0t5.jpg)

* Widgets (controls, also called View Tree), it is a tree structure used to describe the original data of the user interface. Usually this layer does not care about drawing at all, it only cares about the user's operations on the data.
* Render Tree, which is a more abstract tree-like data structure. Generally speaking, it is the same as the View Tree structure of the previous step, and it does not care about the original data, only the layout and size of the control. After calculating the control layout through this step, the appearance of the control can be truly determined.
* Layer Tree corresponds to Render Tree. This step will actively trigger the appearance rendering of each element in the Render Tree, and determine the real appearance of each control when the size and position of the control are known. However, the tree structure of the Layer Tree does not correspond to the Render Tree one-to-one. It is possible that the Layer Tree's leaf nodes of the Render Tree correspond to only one Layer due to Layer merge optimization.
* After the size, location and appearance of the controls have been determined, the rest of the work needs to be combined and displayed on the screen. The principle of this step is relatively simple, which is to merge the Layer of the previous step into a Bitmap, which is the simplest form of image storage. After the Bitmap is rasterized, it can be submitted to the GPU for rendering.


# Rendering process
Summarizing the rendering process of most GUI frameworks from a macro perspective, except that some frameworks have different timings for processing Animation, they can be basically summarized as the following figure:

![](https://ws1.sinaimg.cn/large/8696f529ly1fwxc4h5m60j20ci0cidg5.jpg)
_(The picture is selected from the Flutter Rendering Pipeline technical lecture)_

# Layout
The Layout stage is mainly responsible for calculating the size and position of the view.
### Webkit / Blink
Webkit relies on CSS (Cascading Style Sheet) to achieve layout.
CSS has the following characteristics:

* Selector: determine the role element of the style
* Style: Determine the layout, size, background and other appearance of the view.
* Deformation, transformation and animation: used for relatively complex animation or appearance forms

It seems that CSS is not only the realization of layout function for Webkit, but also the realization of drawing and animation. CSS provides a frame model layout and a Flex layout for Webkit. The combination of these two layouts can basically achieve any form of layout requirements.

### Cocoa / Cocoa Touch
Cocoa and Cocoa Touch have always relied on a simple frame model layout in the early days. It was not until the form of iOS devices began to enrich, that AutoLayout was brought to the stage of history.
Of course, there are many third-party frameworks that move the layout of FlexBox to Cocoa Touch, but it relies on the frame model layout at all, and it is not built into the GUI framework to participate in the rendering process (this is not harmful).

### Android GUI
Android only supports the frame model layout, but early Google engineers realized that this is far from enough, so Android has more predefined layout models (such as LinearLayout, GridLayout) to help developers achieve more complex layouts.

### Flutter
Flutter can be said to be of the same origin as Blink, but its layout model is based on the strengths of each. Flutter also supports frame layout, Flex layout, and a predefined layout model similar to Android.
However, the layout of Flutter and Android GUI has a less obvious shortcoming: the layout model has also become one of the controls, which seems a little strange, obviously the layout cannot render content.

# Paint
The paint stage is mainly responsible for calculating the content of the view.
![](https://ws1.sinaimg.cn/large/8696f529ly1fwy5ceq7ihj20e40ci74o.jpg)

Generally speaking, the GUI framework needs to call graphics-related functions or functions at this stage to express the content data of each layer (Layer). If the drawing operation is calculated by the CPU, it is called software drawing. If it is done by the GPU, it is called hardware accelerated drawing. Usually these two types of graphics CPUs are the majority, but they are often mixed. The CPU can only handle 2D graphics, and when it comes to 3D, the GPU can only complete this part of the work.

### Cocoa / Cocoa Touch
macOS has always relied on Quarz 2D (Core Graphics) to render views. It was not until iOS that Core Animation with higher performance and more modern design was used.
When rendering 3D, you can use OpenGL ES, but the support has been removed on iOS 12 and changed to its own Metal engine.

### Android GUI, Flutter and Blink
All three are from Google, so their 2D rendering engine uses Skia.
3D drawing uses popular OpenGL(ES)

### WebKit
The drawing implementation of WebKit is relatively abstract. From the beginning of the design, in order to support cross-platform, the different drawing interfaces of each platform are abstracted into a unified interface: PlatformGraphicsContext. The implementation of this interface on the macOS/iOS platform is CoreGraphics, which is implemented on Android. It's Skia.
The same is true for 3D drawing. WebKit abstracts the PlatformGraphicsContext3D interface.

# Composite
Usually, the pixels used by the layer rendered in the Paint stage far exceed what the screen can carry. Obviously, there is no need for so many pixels to display a screen of content, and the GPU does not need to be extra useless pixels. The data is calculated, so the Composite step is required for Layer synthesis.
Since the paint stage has determined that the appearance data of each layer is stored in the memory, the synthesis stage only needs to fetch the data from the memory for calculation to determine which layer of data is displayed in a certain block. After this stage, the vector data will be a vector, which can be submitted to the GPU for rendering after rasterization.


# Design a GUI framework?
Before making the selection of various parts of the GUI framework, let's take a look at the comparison of various GUI frameworks.

|Comparison Items|Paint| Layout | SDK |
|:---:|:---:|:---:|:---:|
|React-Native| / | Frame+FlexBox| Javascript |
| Flutter | Skia | Frame+FlexBox+LayoutWidgets| Dart |
|Cocoa(Touch)|CoreAnimation| Frame+AutoLayout|objc/Swift|
|Android GUI| Skia| LayoutWidgets+Frame| Java |
|WebKit|Portable*| Frame+FlexBox| Javascript|

(It can be seen from the above table that React-Native can not be regarded as a GUI framework, at most it can be regarded as an application layer SDK to help you build controls on top of the GUI framework.)

### Paint selection
Many of us hope that applications on multiple platforms only need to write one code, so the selection of Paint is particularly important. Currently, only WebKit and Flutter implement cross-platform. The former abstracts the drawing interface of the platform, and the latter uses a cross-platform drawing library. Both can be ported to different platforms.
But if you have to compare, I personally agree with the WebKit solution. In theory, WebKit can be ported to any system, but Flutter cannot be ported if the Skia library does not support it. And the WebKit solution can reduce the size of the application, and Skia's binary files are still too large.

In this round of PK, I think Portable*'s abstract graphics interface won.

### Layout selection
Layout is about the developer's experience. Facts have proved that Frame + FlexBox is a very friendly and very efficient layout method. Even in the face of extremely complex layout methods, only the framework layer provides some layout controls to assist.

### SDK design
Before designing the SDK, we need to determine what programming language to use. We want to be cross-platform, dynamic and have a low entry barrier, so it seems that only JavaScript and Dart are available. But I think about it carefully and find that the runtime of Dart is still too big, and the runtime of JavaScript has already been built-in by various platforms, we only need to reference it.

The proof of the WebKit DOM API over time is a heavy historical burden. If it weren't for this way of interface construction, frameworks such as React and Vue would not be as hot as they are today. After all, GUI development is also programming. It is not enough to describe the interface, so the SDK should be a programming framework similar to UIKit.

In the end, my ideal GUI framework might look like this:

![](https://ws1.sinaimg.cn/large/8696f529ly1fwydv9w4vjj20s00p0wfx.jpg)

This design may meet my requirements:

* Cross-platform, using Portable’s abstract graphics API to achieve drawing, basically smoothing out platform differences
* Lightweight, no additional graphics library is introduced, and no additional language runtime is introduced, except for data structure and rendering logic, no additional volume occupied
* Low threshold, using Javascript as the SDK to achieve, I am afraid there is no way to lower the threshold
* Dynamic, the language writing business executed by JIT can be dynamic
* The historical burden is small, the GUI framework itself can be upgraded with application upgrades, and problems can be fixed and released in time. It can be said that this design is actually a mini WebKit that has no historical burden, is embedded, and has removed a lot of network modules and only cares about rendering.


# The End
The design of GUI Framework is actually not simple at all. View Tree, Layout, Paint any stage is a very big topic. The above is just a kind of opinion and imagination, it is too time-consuming and laborious to do some attempts.