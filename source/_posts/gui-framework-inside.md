---
title: GUI Framework Inside
date: 2018-11-06 16:10:59
tags:
---

> 阅前提醒：本文仅属个人观点，如有雷同纯属巧合，如有错误请指正。

GUI 差不多已经发展了近 30年，到现在这项技术已经基本成熟，各种 GUI框架基本已经大同小异，下面是流行的 GUI框架一览：

* Cocoa/Cocoa Touch （Apple, macOS/iOS)
* Windows Presentation Foundation (Microsoft, Windows)
* Android GUI (Google, Android)
* WebKit (Apple, Safari）
* Blink (Google, Chrome)
* Flutter (Google, Android/iOS/Fuschia/Chrome)

<!--more-->

> 电影与显示器:

>电影是近代最伟大的发明之一，它的原理是人眼的“视觉暂留”，一个物体的视相消失后可以在视网膜短暂停留 0.1-0.4s，当电影胶片以 24格每秒匀速转动时，一系列静态画面就会因为视觉暂留而造成一种连续的视觉印象。

>现代显示器一般以 60Hz(>=60)的刷新频率为主，这就要求显示处理器(GPU)在 1s内提供 60张可供显示的图片数据，当显示处理器来不及处理这么多只能提供少于 60这个数量时，很多人就能感觉到卡顿（掉帧），当这个值少于 30甚至少于 24时，基本上所有人都能感受到卡顿（掉帧）。


# GUI框架的一般形式
GUI是电影（动画）的一种延伸，它以动画的形式向用户展示可交互界面，对用户的操作进行视觉上的反馈。
从上面可以知道，GUI框架理所应当向 GPU在 1s内提供至少 60次静态图像，GUI框架本身也需要在 1s内完成一定量的计算，以完成用户的交互需求。

**用户操作的基本 GUI单位是控件**，按钮是控件，图片是控件，甚至窗口也是控件。应用通过控件之间的排列与组合为用户提供丰富多彩的可交互界面。  
**控件也是 GUI框架进行渲染的基本单位**，GUI框架通过控件树描述界面的数据结构，通过控件的样式属性来确定大小和外观。  

早期的 GUI框架通过设计出不同的控件后，让控件自己决定绘制的样式，这样做虽然没有什么问题，但通常效率都不够高。不同控件都会存在很多类似的绘制代码（代码冗余），并且在真正绘制时让 GPU疲于奔命，而且每个控件的绘制周期都比较独立，很难从框架总体上去进行管控和定位问题。  
经过多年的迭代和发展后，GUI框架发展成如今这种大同小异的结构：
![现代 GUI框架结构](https://ws1.sinaimg.cn/large/8696f529ly1fwx63r4baaj20ci0ci0t5.jpg)

* Widgets（控件，也叫 View Tree），它是用于描述用户界面原始数据的树状结构。通常这一层根本不关心绘制，它只关心用户对数据的操作。
* Render Tree，它是一种更为抽象的树状数据结构，一般来说它是和上一步的 View Tree结构相同，并且它不关心原始数据，只关心控件的布局和大小。通过这一步计算出控件布局后才能真正地确定控件的外观。
* Layer Tree 跟 Render Tree是相对应的，这一步会主动触发 Render Tree中每个元素的外观渲染，在已知控件大小和位置的情况下决定每个控件的真正外观。但 Layer Tree的树状结构不是和 Render Tree一一对应的，Layer Tree有可能因为 Layer合并优化导致一层的 Render Tree叶子节点最终只对应一个 Layer。
* 在已经决定好控件的大小位置以及长相后，剩下的工作就需要把这些东西组合起来显示到屏幕上。这一步原理比较简单，就是将前一步的 Layer合并成一张 Bitmap，这是一种最简单的图像存储形式。将 Bitmap光栅化后便可以提交给 GPU渲染。


# 渲染流程
从宏观上来总结大多数 GUI框架的渲染流程，除了部分框架在处理 Animation的时机有所不同，基本都可以总结为下图：  

![](https://ws1.sinaimg.cn/large/8696f529ly1fwxc4h5m60j20ci0cidg5.jpg)  
_（该图选自 Flutter Rendering Pipeline技术专题演讲）_  

# Layout
Layout阶段主要负责计算出视图的大小和位置。
### Webkit / Blink
Webkit 依靠 CSS（Cascading Style Sheet）实现布局。  
CSS有以下特点：  

* 选择器：确定样式的作用元素
* 样式：确定视图的布局、大小、背景等外观。
* 变形、变换和动画：用于相对复杂的动画或外观形式

看起来 CSS对于 Webkit而言不止只是一个布局功能的实现，而且还是绘制以及动画的实现。CSS为 Webkit提供了框模型布局以及 Flex布局，通过这两个布局的组合基本上可以实现任意形式的布局需求。

### Cocoa / Cocoa Touch
Cocoa 以及 Cocoa Touch早期一直依赖简单的框（frame）模型布局，直到 iOS设备的形式开始丰富起来时，才把 AutoLayout这一布局搬上历史舞台。  
当然不乏有第三方框架将 FlexBox的布局搬到 Cocoa Touch 上，但其依赖的根本还是框模型布局，并没有被 GUI框架内置参与到渲染流程中（这也无伤大雅）。

### Android GUI
Android仅仅只支持了框模型布局，不过很早 Google 工程师就认识到这是远远不够的，所以 Android有着更多预定义的布局模型（比如 LinearLayout、GridLayout）来帮助开发者实现更复杂的布局。

### Flutter 
Flutter可以说和 Blink是同宗同源的，但是它的布局模式却集各家所长。 Flutter同时支持框布局，Flex布局，以及类似于 Android的预定义布局模型。  
但 Flutter和 Android GUI的布局都有个不太明显的缺点：布局模型也成为了控件之一，这看起来有一些奇怪，很明显布局不能渲染内容。

# Paint
paint 阶段主要负责计算出视图的内容。 
![](https://ws1.sinaimg.cn/large/8696f529ly1fwy5ceq7ihj20e40ci74o.jpg) 

一般来说，GUI框架在这个阶段需要调用图形相关的功能或函数来表达出每一层（Layer）的内容数据。如果绘图操作由 CPU计算完成，那么称之为软件绘图。如果由 GPU完成，那么称之为硬件加速绘图。通常这两种绘图 CPU居多，但混合的情况是经常有的。CPU只能处理 2D绘图，当碰到 3D的情况时只能由 GPU完成这部分工作。

### Cocoa / Cocoa Touch
macOS 一直以来都依靠 Quarz 2D (Core Graphics）来渲染视图，直到 iOS上才使用了更高性能、更现代化设计的 Core Animation.  
在渲染 3D时，可以使用 OpenGL ES，不过在 iOS 12上已经去除了支持，改为自家的 Metal引擎。

### Android GUI、Flutter 和 Blink
三者都来自 Google，因此他们的 2D渲染引擎都采用了 Skia。  
3D绘图采用了流行的 OpenGL(ES)  

### WebKit
WebKit的绘图实现就比较抽象了，从设计之初为了支持跨平台，把各平台有差异的绘图接口抽象为统一接口：PlatformGraphicsContext. 这一个接口在 macOS/iOS 平台的实现就是 CoreGraphics，在 Android的实现就是 Skia.
3D绘图的道理也是一样，WebKit抽象出了 PlatformGraphicsContext3D的接口。

# Composite
通常在 Paint阶段渲染的 Layer所使用的像素(pixel)都远远超过了屏幕所能承载的，很明显在显示一屏内容时不需要这么多像素，GPU没必要为额外那么多没用的像素数据执行计算，所以需要 Composite这一步进行 Layer合成。  
由于 Paint阶段已经决定了每一个 Layer的外观数据存在内存中，所以合成阶段只需要从内存中取数据计算，决定某一块具体显示哪一个 Layer的数据。这一阶段过后得到的将是一份矢量图数据，在进行光栅化后提交给 GPU执行渲染即可。


# 设计一个 GUI框架？
在做 GUI框架各部分的选型之前，先来看一下目前各种 GUI框架的比较

|比较项|Paint| Layout | SDK |
|:---:|:---:|:---:|:---:|
|React-Native| / | Frame+FlexBox| Javascript |
| Flutter | Skia | Frame+FlexBox+LayoutWidgets| Dart |
|Cocoa(Touch)|CoreAnimation| Frame+AutoLayout|objc/Swift|
|Android GUI| Skia| LayoutWidgets+Frame| Java |
|WebKit|Portable*| Frame+FlexBox| Javascript|

（从上表可以看出 React-Native不能算是一个 GUI框架，最多算是一个应用层的SDK，帮助你在GUI框架之上构建控件。）

### Paint的选型
我们很多人都希望多个平台的应用只需要写一份代码，所以 Paint的选型尤为重要。目前只有 WebKit和 Flutter实现了跨平台，前者通过抽象出平台的绘图接口，后者使用跨平台的绘图库，两者均可以移植到不同的平台。  
不过非要作比较，我个人还是比较认可 WebKit的方案。WebKit理论上可以移植到任何系统，而 Flutter如果 Skia库不支持的话就不能移植。并且 WebKit的方案可以减少应用的体积，Skia的二进制文件还是太大了。  

这一轮 PK我觉得还是 Portable*的抽象绘图接口胜出。

### Layout的选型
Layout关系着开发者的体验。事实已经证明 Frame + FlexBox是非常友好而且非常高效的布局方式，就算面对极其复杂的布局方式，也只需要框架层提供部分布局控件给予辅助即可。  

### SDK的设计
在设计 SDK前，我们需要先确定使用什么编程语言。我们希望跨平台，动态化，入门门槛较低，那么似乎只有 JavaScript和 Dart可选。但仔细一想发现 Dart的运行时还是太大了，而 JavaScript的运行时早已被各个平台内置，我们只需要引用即可。  

WebKit DOM API经过时间的证明已经是个沉重的历史包袱。若不是这种界面构建方式，React、Vue之类的框架也不会如今这般火热。说到底 GUI开发也是编程，仅仅描述界面是不够的，所以 SDK应当是一个类似于 UIKit之类的编程框架。

最终，我心目中理想的 GUI框架可能是这样的：

![](https://ws1.sinaimg.cn/large/8696f529ly1fwydv9w4vjj20s00p0wfx.jpg)

这样设计可能符合我的几点要求：  

* 跨平台，用 Portable的抽象图形API实现绘图，基本可以抹平平台差异  
* 轻量级，没有额外图形库的引入，也没有额外语言运行时的引入，除了数据结构和渲染逻辑，没有额外的体积占用
* 门槛低，用 Javascript作为 SDK实现，怕是没有门槛再低的办法了
* 动态化，JIT执行的语言写业务可以动态化
* 历史包袱小，GUI框架本身可以随着应用升级而升级，有问题可以及时修复发版。可以说这样设计其实就是一个没有历史包袱、嵌入式的、去掉了一大堆网络等模块只关心渲染的迷你 WebKit了。


# The End
GUI Framework的设计其实一点儿也不简单，View Tree、Layout、Paint任何一个阶段都是一个非常大的课题。上文只是一种看法和畅想，要做一些尝试还是太费时费力了，有这时间不如多玩几把荒野大表哥。