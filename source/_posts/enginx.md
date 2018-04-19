---
title: 客户端路由重写引擎 — enginx
date: 2017-08-29 19:50:48
tags:
---
<img src="http://cdn.stephenw.cc/wp-content/uploads/2017/08/TB1MVRfLVXXXXaKaXXXXXXXXXXX-800-545-768x523.jpg" style="max-width: 350px;"/>  
（上面图片来源 心动网络）  
很多手机 App的用户量动辄上千万，客户端的动态化是一个非常热门的话题，我也准备蹭一波热度。

<!--more-->

之前参考天猫团队的实践也做了一层路由层，主要用于业务模块和模块之间、远程通知和模块之间、Web 和客户端模块之间等通过 URL 协议互调。不过很清楚能看得出来每个部分都有自己的 URL 协议格式，如：

```javascript
//Web
https://h5.ele.me/restaurants?id=9527
 
//native client
eleme://restaurant?restaurant_id=9527

```

客户端随着业务模块的增长，整个协议的异构化会非常严重，这就导致很多问题，举几个简单场景：

1. Web 前端开发者，需要通过一系列 hack 的方式判断当前页面是否运行在 Native App 内，根据这个 bool 值决定通过 scheme 调用 Native 页面还是继续跳转 Web 页面。
2. 某个在 App 内写死的 URL scheme调用功能模块出了 bug，只能通过发版更改 URL 解决。
3. 没有几个人清楚调用另一个模块的完整 URL 该怎么写，很可能会拼错，导致线上无法跳转。  


等等，还有一些其他小问题，但上面就足够大了。因此针对这种情况，除了借鉴天猫团队的 Rewrite 引擎，我自己搞了一个 SDK 专门用于 URL 协议的转换，比如把 上文提到的两条 URL 互相转换，但并不是那么难懂的正则表达式，而是语义化的 DSL，我管这个 SDK 叫做 enginx。

# enginx 介绍

看名称而言就是模仿 nginx的相关模块。用法也是差不多，给一个配置文件或字符串，输入 URL 字符串，输出目标字符串，就这么简单。

但从概念上，enginx 把一个 URL 划分为同 server，同 location 这些域来进行匹配操作，概念上也是和 nginx 非常像的。
![](http://cdn.stephenw.cc/wp-content/uploads/2017/08/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7-2017-08-29-%E4%B8%8B%E5%8D%884.04.34-768x610.png)


具体的代码引用我就不多作解释了，看一下如何通过一个配置文件完成一条 URL 的重写：


```javascript
server {
  domain: "ele.me";
  scheme: "https";
  port: "443";
 
  location "/" {
    if (equal $scheme "https") {
      return "https://help.ele.me";
    }
    return "https://h5.ele.me";
  }
}
```

这样我们可以解决上面所说的几个问题：

1. 可以通过重写 h5 domain 的 URL 到 Native 路由，Web页面在跳转时即可不用代码判断，直接跳转 web URL 也能唤起 Native 页面
2. 某个模块出 bug，可以通过下发新的配置重写该URL，让它跳不过去，为补救争取时间。
3. 通过重写 SDK，URL 的拼凑逻辑完全可以落库落到配置文件代码，后面可以实现 UI 化，傻瓜式配置。

## URL 匹配

URL 的匹配是一个很重要的过程，因为你不同的 url 想要重写成不同的效果，它们对应的操作也不相同。enginx 最顶层的域是 “server”，一个 server 对应着同一个 domain 下面的所有 URL。

但很显然并不是同 domain 的 URL 都会做同样的重写操作，所以还会有 location 的区别，location 就是针对 URL path所作的正则匹配，匹配完成才能进入 location scope操作。

所以上面的配置文件写的内容便能匹配 https://ele.me 这条 URL，最终 rewrite 的结果就是 https://help.ele.me。 server 字段的 domain 能匹配应该能理解， location 的 “/”的意思就是根路径，使用过 nginx 的同学应该都知道。

## 指令

除了能够匹配到想要重写的 URL，还需要有一系列指令对 URL 进行操作才能完成重写。因此我在写 SDK 的时候给配置文件定义了这些指令的能力，比如：

1. encode/decode， 对一个字符串或变量进行 URL encode/decode
2. match， 使用正则表达式从一个字符串或变量提取变量
3. parse，解析类似于 k=v 的字符串形式提取变量
4. var，定义变量
5. greater/equal/smaller，用于 if 代码块的比较操作
6. return，返回重写的字符串

## 变量
指令进行操作时用于代指一些字符串字面量的标识符，包括了内置变量和定义变量。

内置变量在一条 URL 被匹配到的时候就会生成，比如 $scheme, $host, $query_string 等都是关于 URL 本身的信息。

定义变量有可能是你定义的，也可能是一些指令操作生成的。

## 字符串模板
其实 URL 本身还是字符串的操作，所以任何变量、定义、返回值都是字符串模板替换其中的变量而来。这样就能达到拼接字符串的目的。

## 引入 enginx

enginx 一共实现了 2版。第一版使用 C++实现，用 JSON 格式下发配置，和上文说得有一点点不一样。第二版使用纯 C 实现，完全新的 DSL 格式的配置文件。

enginx 完全是跨平台的，目前封装了 iOS pod, Android module，Node.js 的 npm管理的库。

第一版在 饿了么 App 中使用半年有余，未出现过任何 crash、内存泄露。但大量使用了正则、引入了 STL 库，在性能上不是很满意，峰值大概只有 5000次/s 左右，代码打包体积也比较大。因此实现了第二版，第二版纯 C 实现，能支持更高级的语法格式，代码包体积下降了 90%，峰值速度提高了 6-7倍。

目前使用 enginx 比较简单，直接跟随 README 做就好了：[https://github.com/stephenwzl/enginx](https://github.com/stephenwzl/enginx)

# enginx 的实现
第一版实现非常简单，完全可以自行切换 1.0分支查看。

第二版实现略微复杂一些，分为几个部分：配置文件解析器，语法树生成，内存管理，运行时。

## 配置文件解析
这一部分看起来吓人，其实比较简单，因为并不是真正的手写递归下降解析器。enginx 的解析器使用了 Bison 工具（Yacc 和 lex），通过语法推导式和 token 定义以及状态转移控制来描述具体的语法。

整体语法比较简单，我也并不想把这部分称作 “编译器”。可以自行在 repo 的 Lexer/enginx.l 和 Lexer/enginx.y 查看。

## 语法树生成
这一部分更吓人，其实也比较简单，主要是链表的运用，简单讲解一下思路：

在 includes/enginx_dev.h 里面可以看到 enginx 语法树的节点定义，包括了几大基本类型：value, location, expression, statement, server，他们本质都是结构体。

在 Bison 的代码进行语法树解析的时候，直接调用我预先写好的挂载语法树节点的函数即可，这些函数可以 chain server/location 以及 expression。所以当配置文件解析完毕时语法树也就生成了。

## 内存管理
因为不是专门为 iOS 平台所写，所以没有使用 NSAutoReleasePool 来管理内存，而是自己简单模仿造了一个。主要作用是使用接口分配一段被标记过的内存，在运行时结束时调用 check 方法检查有没有无意义的内存。本质也不难，可以在 memory 文件夹下看到具体实现。

## 运行时
运行时的本质是拿到 URL 和语法树的根节点，进行匹配，然后执行语法树里面的表达式逻辑。这部分也比较简单，也就不多做介绍了，可以直接在 implements/enginx.c 里面找到。

## The End

enginx 是我一个跨平台的练手之作，可能在文档、通用性上没考虑那么多，但至少解决了语义化配置的问题。

别再用正则表达式重写各种各样的 URL 了