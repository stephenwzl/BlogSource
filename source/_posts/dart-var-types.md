---
title: Dart的变量和类型
date: 2018-04-21 12:11:01
tags:
---
在[上一篇文章](/2018/04/20/dart-getting-start/)中我们简单地介绍和认识了 Dart语言。  

在本文中，我们系统地来认识一下 Dart编程语言中的重要组成部分：变量和类型。

<!--more-->

# 变量
在上文中已经知道，定义变量的语法很简单，比如：

```dart
var name = 'Bob';
int line = 0;               // 定义 int类型
String foo = 'Bar';         // 定义 String 类型
List counts = [1, 2, 3];    // 定义 List 类型
```

> 在 Dart中，变量都是引用类型，也就是说所有的变量都是对象，所以 Dart是一门完全面向对象的语言。

Dart 是类型安全的，所以当你使用 `var`关键字定义变量时，本质其实就是具体类型的引用。比如上文代码其实就是一个 `String`类型对象的引用，这个对象的内容是 'Bob'

## 未初始化变量
未初始化的变量它的值都是 null，在 Dart中这一点很叫人放心。

```dart
int lineCount;
assert(lineCount == null);
```

## 类型可选
当你使用 `var`定义变量时，表示类型是编译器推断决定的，当然你也可以用静态类型去定义变量。  
用静态类型去定义变量的好处是你可以更清楚地跟编译器表达你的意图，这样编译器和编辑器就能使用这些类型向你提供 warning或者代码补全这些功能。

不过在 Dart2中，类型不再是可选的。意思是当你使用 `var`定义 name变量时，这个类型便会约束在 String类型，如果赋值其他类型便会出错。如果你不想约束一个变量的类型，那么可以使用 `Object`类型定义或者 `dynamic`关键字定义。  

## final 和 const
如果你想定义不可变的变量，那么在定义前加上 `final`或者 `const`关键字。  

```dart
final nickname = 'ccc';
const int count = 3;
const List list = const [];
```


`const`表示变量在编译期间即能确定值，如果这个变量是 `class`内的，就用 `static const`标记。使用`const`标记变量时，你需要在声明的时候就给定其值。

`final`有些不太一样，用它标记的变量可以在运行时确定，比如：

```
var x = 100;          // 运行时变量，可以是任意其他值
var y = 30;           // 运行时变量，可以是任意其他值

final res = x / y;
```
`final` 其实就是在运行时声明一个不可变的引用，这个引用一旦确定就不可再变。


# 类型
Dart 内置了如下类型：  

* Number
* String
* Boolean
* List
* Map
* Rune
* Symbol

在不引入其他库的情况下，你可以使用这些类型声明变量。

## Number
Dart 的 number类型其实只有两种：int 和 double  
int 的范围是 -2^53 ~ 2^53  
double 是符合 IEEE 754标准的 64位双精度的浮点数。  

`int`和 `double`都是 `num` 的子类型。除了常见的运算符外，你还能使用内置的 `abs()`、`ceil()`、`floor()`等方法。如果还有其他运算方法的需求，你可以引入 `dart:math`库来尝试用一下。  

## String
Dart的 String其实由 UTF-16的代码串组成。和 JavaScript一样，你既能使用单引号也能使用双引号来写字符串的字面量  

```dart
var s1 = 'Single quotes work well for string literals.';
var s2 = "Double quotes work just as well.";
var s3 = 'It\'s easy to escape the string delimiter.';
var s4 = "It's even easier to use the other delimiter.";
```

你还能在字符串中嵌入变量，除了 String类型的变量外，Dart对字符串中嵌入的对象会默认调用它的 `toString()`方法：

```dart
var s = 'string interpolation';
var s1 = 'this is uppercased string: ${s.toUpperCase()}'
```

字符串的拼接直接使用 `+`运算符，这个就不多提了。  
值得一提的是声明多行字符串时，你可以用三个单引号或三个双引号，这有点像 Python：

```dart
var s1 = '''
You can create
multi-line strings like this one.
''';

var s2 = """This is also a
multi-line string.""";
```

## Rune
提到 String，不得不提一下 Rune，Rune其实就是 Dart中的 UTF-32的字符串。  
因为 Unicode给每个单词、字符或者符号都定义了特殊的数字值，所以要拿 String去表示 32位的 Unicode会比较困难。  
通常地，一个 Unicode码的形式是 \uXXXX，这个 XXXX表示 4位长度的十六进制数。比如字符（♥）的 Unicode码是 `\u2665`,但如果要表示的码超过或者少于 4位，就要用花括号括起来，比如 😆 就是 `\u{1f600}`.  

Rune其实就是帮助 String来表示 32位 Unicode的。比如下例：

```dart
main() {
  var clapping = '\u{1f44f}';
  print(clapping);
  print(clapping.codeUnits);
  print(clapping.runes.toList());

  Runes input = new Runes(
      '\u2665  \u{1f605}  \u{1f60e}  \u{1f47b}  \u{1f596}  \u{1f44d}');
  print(new String.fromCharCodes(input));
}
```

输出是：

```
👏
[55357, 56399]
[128079]
♥  😅  😎  👻  🖖  👍
```

很清楚可以看到，String的 `codeUnits`函数只能返回 16位的码，但`Rune`就可以表示 32位。

## Boolean
Dart 的 bool 类型只有两个值： true 和 false  
和其他语言不一样的地方是，当执行判断时，Dart中只有 true才会判定真，其他任何值都为 false  

```dart
var name = 'Bob';
if (name) {
  // 在 Dart中不会执行到这儿
  print('You have a name!');
}
```

## List
你也可以管 List叫 Array，但 Dart中数组就叫 List了。

```dart
var list = [1, 2, 3];
```

Dart的 List其实是有类型的，比如上述代码的类型就是 `List<int>`, 所以你往这里面添加元素必须是 int类型的。  

和其他语言一样，数组索引也从 0开始。同样地，你还可以像 JavaScript那样迭代数组：

```dart
list.forEach((item) {
  print('index: ${list.indexOf(item)}, item:${item}');
});
```

除了一些基本的函数外，Dart其实还有很多方法来操作集合类型，具体的可以参考文档：[Collections in Dart](https://v1-dartlang-org.firebaseapp.com/guides/libraries/library-tour#collections)

## Map
Map类型在其他语言中也叫做字典，即 key-value的集合。在 Dart中声明 Map也很简单：

```dart
var gifts = {
  // Key:    Value
  'first': 'partridge',
  'second': 'turtledoves',
  'fifth': 'golden rings'
};

// 或者

var gifts = new Map();
gifts['first'] = 'partridge';
gifts['second'] = 'turtledoves';
gifts['fifth'] = 'golden rings';

```
同样地，Map也有类型，上述代码即 Map<String, String>类型。但很多时候我们并不会按照既定的泛型使用 Map，如果你仔细阅读了上文，你会发现解决方法也很简单。  
关于 Map的操作方法你也可以参考 [Collections in Dart](https://v1-dartlang-org.firebaseapp.com/guides/libraries/library-tour#collections)

## Symbol

symbo是 Dart中比较特殊的一种类型，你一般不会用到它，但如果你需要获取标识符的字面量时，这个作用就无法估量了。

```dart
var name = 'a';
Symbol s = #name;
```

所以一般情况下，没什么作用，不过这个东西的高级用法我们以后会提到。


# The End
通过变量和类型的了解可以认识到：

* 在 Dart中，所有变量都是对象，而所有对象都是类的实例，null也不例外。所有的类都从 `Object`继承而来
* 尽量不要偷懒，为变量指定类型，这样编译器和编辑器都能更好地帮助你写的代码。
