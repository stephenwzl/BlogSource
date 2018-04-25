---
title: Dart 的类
date: 2018-04-25 16:50:33
tags:
---

Dart 是一门真正的面向对象的编程语言，它的继承是由 class和 mixin实现的。每个对象都是一个 class的实例，并且每个 class都由 `Object`这个 class派生。而 mixin意味着所有的 class都是单继承的，但通过 mixin，一个 class里面可以使用其他非父类中的结构。  

<!--more-->

# 实例变量
声明实例变量和别的语言没什么两样。在 Dart中，所有的实例变量都会隐式地生成 `getter`，而非 `final`标记的实例变量还会隐式地生成 `setter`。

```dart
class Point {
  int _x;
  get x => _x;
  set x(int value) => _x = value;
}
```

假如你在声明实例变量的时候就给它赋值了，那么这个实例变量的调用会比构造函数还要早。

# 构造函数
在[前文](/2018/04/20/dart-getting-start/)已经简单地介绍过 Dart的构造函数，所以这里假定读者已知一些基本的用法。我们来看看 Dart中构造函数一些比较有意思的地方。

## 语法糖
假如你的构造函数只是给一些实例变量赋初值，可以使用语法糖减少代码量。

```dart
class Point {
  num x, y;
  // 语法糖
  Point(this.x, this.y);
}
```

## 默认构造函数
如果在声明类的时候没有声明构造函数，Dart 提供一个默认的构造函数，即 `ClassName()`.   
但**构造函数不会继承**，也就是说父类的构造函数子类不可以使用，子类不提供自己的构造函数的话也只会生成一个默认的构造函数。

## 具名构造函数
使用命名的构造函数会更有语义：

```dart
class Point {
  num x, y;

  Point(this.x, this.y);

  // Named constructor
  Point.origin() {
    x = 0;
    y = 0;
  }
}
```

## 重定向构造函数
构造函数可以重定向到其他构造函数

```dart
class Point {
  num x, y;
  Point(this.x, this.y);
  // refirecting constructors
  Point.bottom(num x) : this(x, 0);
  Point.left(num y) : this(0, y);
}
```

## 常量构造函数
有时候你希望通过构造函数生成一个常量对象

```dart
class ImmutablePoint {
  static final ImmutablePoint origin =
      const ImmutablePoint(0, 0);

  final num x, y;

  const ImmutablePoint(this.x, this.y);
}
```

## 初始化列表
在构造函数的函数体真正执行之前，你还有机会给一些实例变量赋值，这就需要用到初始化列表。
  
```dart
class Point {
  num x, y;
  // initializer list
  Point.fromData(Map data): x = 0, y = 0 {
    print('$data');
  }
}
```

## 工厂模式构造函数
Dart 中史无前例地设计模式出现在了语法上，通过 `factory`关键字声明的构造函数可以更好地帮我们理解该类的设计模式。

```dart
class Logger {
  final String name;
  bool mute = false;

  static final Map<String, Logger> _cache = <String, Logger>{};

  factory Logger(String name) {
    if (_cache.containsKey(name)) {
      return _cache[name];
    } else {
      final logger = new Logger._internal(name);
      _cache[name] = logger;
      return logger;
    }
  }

  Logger._internal(this.name);

  void log(String msg) {
    if (!mute) print(msg);
  }
}
```
> factory 构造函数没有权限访问 this 关键字 

这样在使用这个类的时候，就可以这样：

```dart
var logger = new Logger('UI');
logger.log('Button clicked');
```

## 继承后的构造函数顺序
前面提到 **构造函数不会继承**，所以子类在继承父类后需要自行声明构造函数。默认情况下，子类构造函数体一开始时就会调用到父类的无参构造函数，然后才会继续执行后续逻辑。  

```dart
class Point {
  Point() {
    print('in point');
  }
}

class Face extends Point {
  Face() {
    print('in face');
  }
}

main() {
  new Face();
}
 
/*
输出

in point
in face

*/
```

但如果你在初始化子类的时候需要调用特定的父类初始化方法，那么你需要在初始化列表中就声明调用哪一个父类构造函数  

```dart

class Point {
  num x, y;
  Point() {
    print('in point');
  }

  Point.fromData(Map data): x = 0, y = 0 {
    print('data is $data');
  }
}

class Face extends Point {
  Face(): super.fromData(null) {
    print('in face');
  }
}

main() {
  new Face();
}

/*
输出

data is null
in face

*/

```

# 方法
对象的函数也叫做方法，这种函数表现了该对象的行为。

## 实例方法
和其他编程语言一样，实例方法才有权限获取 this引用，这里就不多做介绍了。


## 抽象方法
抽象方法推荐只在抽象类中使用（在普通类中会产生 warning）。抽象类可以作为一种协议，让其他开发者来遵守，并且实现其中的抽象方法。

```dart
abstract class Doer {
  void doSomething(); // Define an abstract method.
}

class EffectiveDoer extends Doer {
  void doSomething() {
    // ...
  }
}
```

## 运算符覆写
除了可以覆写方法，运算符也可以被覆写，具体可以被覆写的运算符如下：

```dart
<  +  |  []  >  /  ^  []=  
<=  ~/  &  ~  >=  *  <<  ==  
-  %  >>
```

下面是一个例子：

```dart
class Vector {
  final int x, y;

  const Vector(this.x, this.y);

  /// Overrides + (a + b).
  Vector operator +(Vector v) {
    return new Vector(x + v.x, y + v.y);
  }

  /// Overrides - (a - b).
  Vector operator -(Vector v) {
    return new Vector(x - v.x, y - v.y);
  }
}

void main() {
  final v = new Vector(2, 3);
  final w = new Vector(2, 2);

  // v == (2, 3)
  assert(v.x == 2 && v.y == 3);

  // v + w == (4, 5)
  assert((v + w).x == 4 && (v + w).y == 5);

  // v - w == (0, 1)
  assert((v - w).x == 0 && (v - w).y == 1);
}
```

如果你需要覆写 `==`运算符，你还需要重写一下 `hashCode`的 getter，因为 Dart通过 hashCode来判断对象的相等性。

# 隐式接口
其实 Dart的类的声明也会隐式地声明了这个类的所有属性组成的接口。所以假设 class A并不支持 class B的 API的时候，就可以使用 A来实现 B的接口，把 A当做 B来用。

```dart
// A person. The implicit interface contains greet().
class Person {
  // In the interface, but visible only in this library.
  final _name;

  // Not in the interface, since this is a constructor.
  Person(this._name);

  // In the interface.
  String greet(String who) => 'Hello, $who. I am $_name.';
}

// An implementation of the Person interface.
class Impostor implements Person {
  get _name => '';

  String greet(String who) => 'Hi $who. Do you know who I am?';
}

String greetBob(Person person) => person.greet('Bob');

void main() {
  print(greetBob(new Person('Kathy')));
  print(greetBob(new Impostor()));
}

/*
output:

Hello, Bob. I am Kathy.
Hi Bob. Do you know who I am?

*/
```

# 类的继承
用 `extends`关键字实现类的继承，用 `super`调用父类属性

```dart
class Television {
  void turnOn() {
    _illuminateDisplay();
    _activateIrSensor();
  }
  // ···
}

class SmartTelevision extends Television {
  void turnOn() {
    super.turnOn();
    _bootNetworkInterface();
    _initializeMemory();
    _upgradeApps();
  }
  // ···
}
```

## 继承覆写
类的实例方法在被继承后可以覆写，具体方法是使用 `@override`注解。

```dart
class Point {
  num x, y;
  Point(this.x, this.y);

  void say() {
    print('$x $y');
  }

}

class Face extends Point {

  @override
  void say() {
    super.say();
    print('xxxx');
  }
}
```

## noSuchMethod()
当调用者尝试调用一个该类上不存在的方法时，该方法就会起到作用，你可以覆写这个方法告诉调用者。

```dart
class A {
  // Unless you override noSuchMethod, using a
  // non-existent member results in a NoSuchMethodError.
  @override
  void noSuchMethod(Invocation invocation) {
    print('You tried to use a non-existent member: ' +
        '${invocation.memberName}');
  }
}
```

# 枚举类型
枚举类型也是一种 class

声明枚举类型

```dart
enum Color { red, green, blue }
```

获取枚举类型的值

```dart
// 枚举类型的具体值和数组一样有一个 index
assert(Color.red.index == 0);
assert(Color.green.index == 1);
assert(Color.blue.index == 2);

// 枚举类型的所有值也可以转换成数组
List<Color> colors = Color.values;
assert(colors[2] == Color.blue);
```

# Mixin
mixin是类无法多继承时的一种解决方法。
要使用 mixin，只需要 `with`关键字

```dart
class Musician extends Performer with Musical {
  // ···
}

class Maestro extends Person
    with Musical, Aggressive, Demented {
  // ... 
}
```

虽然语言层面上，mixin的实现比较复杂，不过在使用语义上，mixin其实就是想象的那样：

```dart
class S {
  twice(int x) => 2 * x;
}

abstract class I {
  twice(x);
}

abstract class J {
  thrice(x);
}
class K extends S implements I, J {
  int thrice(x) => 3* x;
}

class B {
  twice(x) => x + x;
}

class A = B with K;

main() {
  var a = new A();
  var c = new K();
  print(a.thrice(1));  // class B's twice
  print(c.twice(1));   // class S's twice
}
```
 
关于 Mixin，后续还会专门开一篇文章介绍关于 Dart Mixin的设计。

# 类变量和类方法
通过 `static`关键字，你可以实现 class 层级的方法和变量

## 类变量

```dart
class Queue {
  static const int initialCapacity = 16;
  // ···
}

void main() {
  assert(Queue.initialCapacity == 16);
}
```

类中的静态变量只有在被用到的时候才会初始化

## 类方法

```dart
import 'dart:math';

class Point {
  num x, y;
  Point(this.x, this.y);

  static num distanceBetween(Point a, Point b) {
    var dx = a.x - b.x;
    var dy = a.y - b.y;
    return sqrt(dx * dx + dy * dy);
  }
}

void main() {
  var a = new Point(2, 2);
  var b = new Point(4, 4);
  var distance = Point.distanceBetween(a, b);
  assert(2.8 < distance && distance < 2.9);
  print(distance);
}
```
看起来和别的编程语言没什么两样。值得一提的是，类方法可以被作为常量使用。

# The End
类是了解面向对象设计的重要途径，从本篇文章你可以发现，Dart的面向对象设计既简单又简洁，但又不失强大性。