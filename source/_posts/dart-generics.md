---
title: Dart 中的泛型
date: 2018-04-27 17:38:31
tags:
---

既然 Dart 1.x版本中类型是可以省略的，那么你是没有必要使用泛型的。但如果你想使代码提示更友好的话，泛型在某种程度上还是非常必要的。  
这就好比 Objective-C中，一个 `NSArray<NSString>`和一个 `NSArray`所获得的效果完全不同似的。

<!--more-->

# 为啥执着于泛型？
举个🌰，如果你想声明一个只能添加字符串的 List，那么你可以使用 `List<String>`声明。这样其他使用你代码的码触就能避免在 List中塞入其他类型的对象，因为一旦塞入，编辑器就会给与提示。

```dart
var names = new List<String>();
names.addAll(['Seth', 'Kathy', 'Lars']);
names.add(42);  // IDE warning 
```

再举一个🌰，用泛型可以提高代码的重用。比方说你要声明一个内存缓存的接口：

```dart
abstract class ObjectCache {
  Object getByKey(String key);
  void setByKey(String key, Object value);
}
```

但这时候你发现你需要一个 String类型的接口，而不是 Object类型的，所以你要再创建一个：

```dart
abstract class StringCache {
  String getByKey(String key);
  void setByKey(String key, String value);
}
```

> double commit，多了一倍代码你的 KPI就会 double了吗？Naive！别人会喷你菜的抠脚

如果你需要再声明一个数字类型的呢？再来一遍吗？很明显泛型就是为这种场景设计的：

```dart
abstract class Cache<T> {
  T getByKey(String key);
  void setByKey(String key, T value);
}
```

和其他语言的泛型一样，这里的 T只是具体类型的替身，你在使用时会用具体的类型替换掉这个 T。

# 使用泛型为集合字面量进行类型检查
我们在写代码中的字面量的时候往往是随心所欲的，这就会导致写错一个字面量的类型就会埋下安全隐患。为字面量添加上泛型时就可以让静态分析器帮忙检查字面量中的错误。

```dart
var names = <String>['Seth', 'Kathy', 'Lars'];
var pages = <String, String>{
  'index.html': 'Homepage',
  'robots.txt': 'Hints for web robots',
  'humans.txt': 'We are people, not machines'
};
```

# 为构造函数添加参数化类型
像集合类型，和 Java类似，你在使用构造函数初始化集合实例对象时可以为它的类型添加约束。
```dart
var names = new List<String>();
names.addAll(['Seth', 'Kathy', 'Lars']);
var nameSet = new Set<String>.from(names);
var views = new Map<int, View>();
```

# 判断集合类型
Dart中集合的类型是具体的，也就是说你通过泛型约束集合后，就是对应的类型。

```dart
var names = new List<String>();
names.addAll(['Seth', 'Kathy', 'Lars']);
print(names is List<String>); // true
```

> 但有时候 List中的 item并不会真的都是 String类型，所以使用 item时最好每一个都要判断一下类型。

# 约束参数类型
有时候你在实现类似通用接口的泛型中期望的类型是某些特定类型时，你可以使用类型约束

```dart
// T 类型必须是 SomeClass类型或者它的子类
class Foo<T extends SomeBaseClass> {
  // ···
}

class Extender extends SomeBaseClass {
  // ···
}

void main() {
  // 可以使用 SomeBaseClass类型
  var someBaseClassFoo = new Foo<SomeBaseClass>();
  var extenderFoo = new Foo<Extender>();
}

```

# 使用泛型函数
刚开始的时候，Dart的泛型只局限于实现 class层面，但后来也支持了泛型函数。

```dart
T first<T>(List<T> ts) {
  // Do some initial work or error checking, then...
  T tmp = ts[0];
  // Do some additional checking or processing...
  return tmp;
}
```

# The End
泛型的玩法其实不少，上文的介绍还是着重与代码重用和类型检查这个层次。更多的用法不仅可以从 C++等其他语言吸收经验，也可以参靠社区的一些讨论。
