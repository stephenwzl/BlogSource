---
title: 认识 Dart语言
date: 2018-04-20 15:06:19
tags:
---

<img src="/images/dart-logo.svg" style="max-width: 400px;"/>
<br/>
  
Dart 是一门开源的编程语言，由 Google Chrome团队开发并维护。  
经过一段时间简单的尝试使用，确实可以让人感觉到这是一门旨在改进客户端开发体验的编程语言，为苦于 JavaScript已久的我们带来了一丝曙光。  
本文介绍 Dart语言基础的上手使用。

<!--more-->

# 安装 Dart
由于手头没有 Linux的机器，所以简介一下 macOS环境的安装。  
Dart 目前虽然已经发展到了 **Dart2**，增加了许多新特性，但本质没有太大的变化，为了更好地了解 Dart的标准，我们直接安装默认的 Dart SDK。  

> 请确保已经安装了 homebrew

打开终端 App，输入：

```shell
brew tap dart-lang/dart
brew install dart
```

> 截止发文日，Dart 默认的 SDK仍不是 Dart2，不过不用担心，并没有太大变化，我们从 1开始看起。

# 选择 IDE
工欲善其事必先利其器，挑选一个合适的 IDE是非常重要的。Atom、Sublime Text、VSCode、Intellij IDEA都有 Dart插件。笔者挑选了 Intellij IDEA 作为本文的 IDE，并且安装了 `Dart Plugin`。  
打开 IDEA的 `Preference` -> `Languages & Frameworks` -> `Dart`，配置 Dart SDK 的 path: 
![](/images/dart-intellij-idea.png)

配置好后，IDEA就能正确地给你提示标准库函数，并且能帮你做语言分析，写起来就很舒服了。



# 第一行 Dart

为了简单地体验一下 Dart，我们直接新建一个 `demo.dart`文件就可以了，可以在 IDEA中打开。一个基本的 hello world 如下：

```dart
// demo.dart

void main() {
  print('hello world');
}
```

然后在命令行中执行 `dart demo.dart`，命令行就会输出:

```
dart demo.dart ↵
hello world
```

Dart 和绝大多数编译型语言一样，都要求 main作为语言执行的入口(entrypoint)，而不是像 JavaScript那样解释执行。所以显而易见，Dart是编译型语言。  
但我们在运行 demo.dart时并没有先编译，这是因为 Dart通过 VM来运行，在载入源码时先编译成字节码，再通过字节码执行，这样的好处是执行速度会比 JIT代码快很多，弊端就是代码的载入到真正执行的速度就比较慢了。所以，Dart也支持编译成 AOT代码，这样代码载入运行的速度能得到很大的提升。

# Dart 语法初探
在知道如何简单地运行 dart代码后，我们来看一下 Dart的语法。
## 变量和类型
在 Dart中，我们可以用 `var`或者具体的类型来声明一个变量：

```dart
var name = 'Bob';
int num = 9527;
```

默认情况下，未初始化的变量的值都是 `null`

```dart
int count;
print(count == null); 	// true
```
这样我们就不用担心在一些情况下无法断定一个传递过来的变量到底是 `undefined` 还是`null`了。因为在 Dart中，所有未定义的情况都是 null。  

Dart 是类型安全的语言，并且所有类型都是对象类型，所有的类型的顶层都来自于 `Object`，理所当然地，`null`也是一个对象。所以，`var`关键字只是一个类型的引用，使用它所声明的对象只是该对象类型的一个引用，这个过程由编译器完成，并不需要关心。  
Dart 拥有比较简单的内置类型，如 Number, String, Boolean, List, Map, Runes, Symbols，就不作一一介绍了。

## 函数和闭包
函数是 Dart执行的基本单元，在函数外的表达式不能获得正确的执行。  
一个基本的函数如下：

```dart
void sayHello() {
  print('hello');
}
```

格式和大部分语言一样就不作介绍了，值得一提的是，如果返回值是 `void`，那么 `void`是可以省略的。  
如果你的函数体只有一行返回值，你可以用**箭头函数**：

```
bool isZero(int num) => num == 0;
```

函数其实也是一个对象，你可以把**函数作为参数**传递或者赋值：

```dart
var loudify = (msg) => '!!! ${msg.toUpperCase()} !!!';
print(loudify('hello') == '!!! HELLO !!!');
```

当然，你也可以用**匿名函数**来迭代 List: 

```dart
var list = ['apples', 'bananas', 'oranges'];
list.forEach((item) {
  print('${list.indexOf(item)}: $item');
});
```

接下来要提到 Dart中对作用域比较友好的一点了，就是**语义化的作用域**，这一点在闭包中体现得淋漓尽致：

```dart
Function makeAdder(num addBy) {
  return (num i) => addBy + i;
}

void main() {
  // Create a function that adds 2.
  var add2 = makeAdder(2);

  // Create a function that adds 4.
  var add4 = makeAdder(4);

  print(add2(3) == 5);
  print(add4(3) == 7);
}
```

在上述代码中，`add2`变量获得了一个 `(num i) => 2 + i;`的闭包， `add4`获得了一个 `(num i) => 4 + i;`的闭包。 这个逻辑完全和你的代码语义一致，如果在别的语言中，你可能就不会这么幸运了。  
语义化的作用域给我们带来的好处就是你可以放心地编码，而不用担心语言中的坑。

## 运算符
Dart 的运算符和绝大部分语言的运算符都一样，所以你可以用熟悉的方式编码。不过 Dart多了几个有意思的运算符。

**`?.`运算符**：假设 Person这个类有 `sayHello()`函数，p是 Person的一个实例。那么 `p?.sayHello()` 表示 p 为 null的时候避免抛出 exception  
**`is`运算符**：如果 p是 Person这个类的实例，那么 `p is Person`返回 true  
**`??`运算符**：`a ?? b`表达式表示如果 a不为 null，返回 a的值，否则返回 b
**`??=`运算符**：`b ??= value`表示如果 b为 null则给 b赋值 value，否则不赋值

## 流程控制
Dart 保持了标准的流程控制语法：`if-else`, `for`, `while`, `do-while`, `break/continue`, `switch-case`, `assert`.
值得一提的是 `assert`，它可以阻止你的代码执行流程（和 C中的 assert类似），不过它只在 Dart的 check mode有用（类似于开发环境），在 production mode无效。

## 类
在 Dart中，声明并使用一个类很简单：

```dart
class Point {
  num x;
  num y;
}

void main() {
  var point = new Point();
  point.x = 4; 
  assert(point.x == 4); 
  assert(point.y == null); 
}
```

**构造函数**：Dart的构造函数很方便，你可以很方便地写

```dart
class Point {
  num x, y;

  // 常见的标准构造函数
  Point(num x, num y) {
    this.x = x;
    this.y = y;
  }
  
  // 语法糖，作用和上面一样
  Point(this.x, this.y);
  
  // 可以命名一个特殊的构造函数
  Point.origin() {
  	x = 0;
  	y = 0;
  }
  
  // 初始化成员列表，这里不能使用 this关键字给 x/y 赋值
  Point()
    : x = 0,
      y = 0 {
    print('初始化成员列表');
  }
}
```

值得一提的是，Dart 没有 `public`、`private`这些关键字，方法名前面加上 `_`即可作为 private方法使用。默认都是 public。  
Dart 还支持抽象类，这样我们就可以来约定一些协议，然后具体的实现放到别的地方去。

```dart
abstract class AbstractContainer {
  // 这里可以定义构造函数、成员变量、方法等待
  void updateChildren(); // 抽象方法.
}

class SpecializedContainer extends AbstractContainer {
  void updateChildren() {
    // 这里提供方法的实现，这样这个方法就不是抽象的了
  }
  // 未实现的抽象方法会抛一个 warning，但并不妨碍对象实例化
}
```
Dart 的类很强大，篇幅限制，后续再开文介绍更强大的特性。

## 错误处理 
Dart 的错误处理也很 Modern，由于 Dart是类型安全的，所以错误处理可以根据类型，更方便一点：

```dart
// demo
try {
  breedMoreLlamas();
} on OutOfLlamasException {
  // 特定类型的 exception
  buyMoreLlamas();
} on Exception catch (e) {
  // 不确定什么类型的 exception
  print('Unknown exception: $e');
} catch (e) {
  // 没有类型，统一处理
  print('Something really unknown: $e');
}
```

## 库
一门编程语言的生态取决于它的库，来看一看 Dart库导入的语法。

```dart
// 导入 html库
import 'dart:html';

// 只导入一部分
import 'package:test/test.dart';

// 给个 alias
import 'package:lib2/lib2.dart' as lib2;

// 只导入该库中的 foo
import 'package:lib1/lib1.dart' show foo;

// 除了 foo全都导入
import 'package:lib2/lib2.dart' hide foo;

// 懒加载
import 'package:greetings/hello.dart' deferred as hello;
```

如何实现一个库可能讲起来篇幅长一点，所以打算放到另外一篇文章中去。

# The End
本篇文章主要就是探讨一下 Dart语言中略微有点不同的地方，这样初学者在上手时会比较轻松。Dart 其实还有不少高级一点的内容，为了避免混淆，我打算全部都另开文章介绍。  
这样的话我们就可以有一条清晰而又简单的 Dart学习路线。

