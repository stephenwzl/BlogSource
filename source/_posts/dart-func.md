---
title: Dart 的函数
date: 2018-04-23 19:09:46
tags:
---

函数是 Dart程序运行的基本单元，在本篇文章中，我们系统地认识一下 Dart的函数。

<!--more-->
Dart 是一门真正的面向对象的编程语言，所以函数也是一种对象的实例，它也有类型，叫做 `Function`. 这就表示函数也可以赋值给变量，也可以作为函数参数传递。  
一个简单的函数如下：

```dart
bool isZero(int number) {
  return number == 0;
}
```

尽管 Dart官方教程希望你给函数的返回加上类型，但你依旧可以忽略返回类型，让编译器自行推断：

```dart
isZero(int number) {
  return number == 0;
}
```

如果像上文的代码只有一行表达式的话，可以用箭头函数来表达：

```dart
bool isZero(int number) => number == 0;
```

# 可选参数
可选参数其实有两种含义  

## 可选的具名参数
什么是具名？看代码  

```dart
// 定义了一个这样的函数
void enableFlags(bool bold, bool hidden) { 
  // ...
}
// 假如参数很多，类型相近，用的时候就不知道哪个参数对应哪个位置了
enableFlags(true, true);

// 希望是下面这种带参数名的用法
enableFlags(bold: true, hidden: true);
```

要达到具名函数的用法，那就在定义的时候给参数加上 `{}`

```dart
void enableFlags({bool bold, bool hidden}) { 
  // ...
}
```

## 可选的位置参数
和 JavaScript不一样的地方是，Dart**某些位置**可忽略的参数必须在函数定义时用 `[]`符号指定：

```dart
// device参数在调用时可以忽略不传
String say(String from, String msg, [String device]) {
  var result = '$from says $msg';
  if (device != null) {
    result = '$result with a $device';
  }
  return result;
}
```

# 参数默认值
和其他语言一样，你可以给函数形参加上默认值，如果该参数未传递，则默认值为 `null`

```dart
void enableFlags({bool bold = false, bool hidden = false}) {
  // ...
}

// bold = true, hidden = false
enableFlags(bold: true);
```

# main函数
main 函数是一个 dart应用的入口。 main函数返回类型是 `void`，可选参数作为一个 `List<String>`类型传递进来.

```dart
// 调用方法：
// dart demo.dart 1 test
void main(List<String> arguments) {
  print(arguments);

  assert(arguments.length == 2);
  assert(int.parse(arguments[0]) == 1);
  assert(arguments[1] == 'test');
}
```

如果你要使用 Dart来写一个命令行工具，你可以用 [args library](https://pub.dartlang.org/packages/args)来解析命令行参数。

# 匿名函数
上述见到的函数都是命名的，Dart当然也可以使用匿名函数。
比如说使用一个匿名函数迭代 List

```dart
void main() {
  var list = ['apples', 'bananas', 'oranges'];
  list.forEach((item) {
    print('${list.indexOf(item)}: $item');
  });
}
```

# 语义化的作用域
该部分内容比较简单，在 [Dart介绍](/2018/04/20/dart-getting-start/)中已经认识过，不再赘述。

# 返回值
Dart中所有的函数都具有返回值，如果一个表达式或函数的返回值没有指定，那么默认就是 `null`。

# 函数的级联
函数的级联也叫链式调用。Dart 原生语法支持链式调用，只需要使用 `..`

```dart
querySelector('#confirm') // Get an object.
  ..text = 'Confirm' // Use its members.
  ..classes.add('important')
  ..onClick.listen((e) => window.alert('Confirmed!'));
```

# The End