---
title: Dart class
date: 2018-04-25 16:50:33
tags:
---

Dart is a true object-oriented programming language, and its inheritance is implemented by classes and mixins. Every object is an instance of a class, and every class is derived from the ʻObject` class. And mixin means that all classes are single inheritance, but through mixin, a class can use structures in other non-parent classes.

<!--more-->

# Instance variables
Declaring instance variables is no different from other languages. In Dart, all instance variables will implicitly generate getters, and instance variables that are not marked as final will also implicitly generate setters.

```dart
class Point {
  int _x;
  get x => _x;
  set x(int ​​value) => _x = value;
}
```

If you assign a value to the instance variable when you declare it, the instance variable will be called earlier than the constructor.

# Constructor
In [previous article](/2018/04/20/dart-getting-start/), Dart's constructor has been briefly introduced, so it is assumed that the reader knows some basic usage. Let's take a look at some of the more interesting aspects of the constructor in Dart.

## Syntactic sugar
If your constructor just assigns initial values ​​to some instance variables, you can use syntactic sugar to reduce the amount of code.

```dart
class Point {
  num x, y;
  // Syntactic sugar
  Point(this.x, this.y);
}
```

## Default constructor
If no constructor is declared when declaring the class, Dart provides a default constructor, namely `ClassName()`.
But the **constructor does not inherit**, which means that the constructor of the parent class cannot be used by the subclass. If the subclass does not provide its own constructor, it will only generate a default constructor.

## Named constructor
Using named constructors is more semantic:

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

## Redirection constructor
The constructor can be redirected to other constructors

```dart
class Point {
  num x, y;
  Point(this.x, this.y);
  // refirecting constructors
  Point.bottom(num x): this(x, 0);
  Point.left(num y): this(0, y);
}
```

## Constant constructor
Sometimes you want to generate a constant object through the constructor

```dart
class ImmutablePoint {
  static final ImmutablePoint origin =
      const ImmutablePoint(0, 0);

  final num x, y;

  const ImmutablePoint(this.x, this.y);
}
```

## Initialization list
Before the body of the constructor is actually executed, you still have the opportunity to assign values ​​to some instance variables, which requires an initialization list.
  
```dart
class Point {
  num x, y;
  // initializer list
  Point.fromData(Map data): x = 0, y = 0 {
    print('$data');
  }
}
```

## Factory pattern constructor
The unprecedented design pattern in Dart appears in the grammar. The constructor declared by the `factory` keyword can better help us understand the design pattern of this class.

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
> The factory constructor does not have permission to access the this keyword

So when using this class, you can do this:

```dart
var logger = new Logger('UI');
logger.log('Button clicked');
```

## Constructor order after inheritance
As mentioned earlier, **constructors will not inherit**, so subclasses need to declare their own constructor after inheriting from the parent class. By default, the subclass constructor body will call the parent class's parameterless constructor at the beginning, and then continue to execute the subsequent logic.

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
Output

in point
in face

*/
```

But if you need to call a specific parent class initialization method when initializing a subclass, then you need to declare which parent class constructor to call in the initialization list

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
Output

data is null
in face

*/

```

# Method
The function of an object is also called a method, which expresses the behavior of the object.

## Example method
Like other programming languages, the instance method has the authority to get this reference, so I won’t introduce it here.


## Abstract method
Abstract methods are recommended to be used only in abstract classes (warning will be generated in ordinary classes). Abstract classes can be used as a protocol for other developers to follow and implement the abstract methods.

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

## Operator override
In addition to overriding methods, operators can also be overridden. The specific operators that can be overridden are as follows:

```dart
<  +  |  []  >  /  ^  []=  
<=  ~/  &  ~  >=  *  <<  ==  
-  %  >>
```


Xiàmiàn shì yīgè lìzi:

```Dart
class Vector {
  final int x, y;

  const Vector(this.X, this.Y);

  /// Overrides + (a + b).
  Vector operator +(Vector v) {
    return new Vector(x + v.X, y + v.Y);
  }

  /// Overrides - (a - b).
  Vector operator -(Vector v) {
    return new Vector(x - v.X, y - v.Y);
  }
}

void main() {
  final v = new Vector(2, 3);
  final w = new Vector(2, 2);

  // v == (2, 3)
  assert(v.X == 2&& v.Y == 3);

  // v + w == (4, 5)
  assert((v + w).X == 4&& (v + w).Y == 5);

  // v - w == (0, 1)
  assert((v - w).X == 0&& (v - w).Y == 1);
}
```

rúguǒ nǐ xūyào fù xiě `==`yùnsuàn fú, nǐ hái xūyào chóng xiě yīxià `hashCode`de getter, yīnwèi Dart tōngguò hashCode lái pànduàn duìxiàng de xiāngděng xìng.

# Yǐn shì jiēkǒu
qíshí Dart de lèi de shēngmíng yě huì yǐn shì dì shēngmíngliǎo zhège lèi de suǒyǒu shǔxìng zǔchéng de jiēkǒu. Suǒyǐ jiǎshè class A bìng bù zhīchí class B de API de shíhòu, jiù kěyǐ shǐyòng A lái shíxiàn B de jiēkǒu, bǎ A dàngzuò B lái yòng.

```Dart
// A person. The implicit interface contains greet().
Class Person {
  // In the interface, but visible only in this library.
  Final _name;

  // Not in the interface, since this is a constructor.
  Person(this._Name);

  // In the interface.
  String greet(String who) =>'Hello, $who. I am $_name.';
}

// An implementation of the Person interface.
Class Impostor implements Person {
  get _name =>'';

  String greet(String who) =>'Hi $who. Do you know who I am?';
}

String greetBob(Person person) => person.Greet('Bob');

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

# Lèi de jìchéng
yòng `extends`guānjiàn zì shíxiàn lèi de jìchéng, yòng `super`diàoyòng fù lèi shǔxìng

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
    super.TurnOn();
    _bootNetworkInterface();
    _initializeMemory();
    _upgradeApps();
  }
  // ···
}
```

## jìchéng fù xiě
lèi de shílì fāngfǎ zài bèi jìchéng hòu kěyǐ fù xiě, jùtǐ fāngfǎ shì shǐyòng `@override`zhùjiě.

```Dart
class Point {
  num x, y;
  Point(this.X, this.Y);

  void say() {
    print('$x $y');
  }

}

class Face extends Point {

  @override
  void say() {
    super.Say();
    print('xxxx');
  }
}
```

## noSuchMethod()
dāng diàoyòng zhě chángshì diàoyòng yīgè gāi lèi shàng bù cúnzài de fāngfǎ shí, gāi fāngfǎ jiù huì qǐ dào zuòyòng, nǐ kěyǐ fù xiě zhège fāngfǎ gàosù diàoyòng zhě.

```Dart
class A {
  // Unless you override noSuchMethod, using a
  // non-existent member results in a NoSuchMethodError.
  @Override
  void noSuchMethod(Invocation invocation) {
    print('You tried to use a non-existent member: ' +
        '${Invocation.MemberName}');
  }
}
```

# méi jǔ lèixíng
méi jǔ lèixíng yěshì yī zhǒng class

shēngmíng méi jǔ lèixíng

```dart
enum Color {red, green, blue}
```

huòqǔ méi jǔ lèixíng de zhí

```dart
// méi jǔ lèixíng de jùtǐ zhí hé shùzǔ yīyàng yǒu yīgè index
assert(Color.Red.Index == 0);
assert(Color.Green.Index == 1);
assert(Color.Blue.Index == 2);

// méi jǔ lèixíng de suǒyǒu zhí yě kěyǐ zhuǎnhuàn chéng shùzǔ
List<Color> colors = Color.Values;
assert(colors[2] == Color.Blue);
```

# Mixin
mixin shì lèi wúfǎ duō jìchéng shí de yī zhǒng jiějué fāngfǎ.
Yào shǐyòng mixin, zhǐ xūyào `with`guānjiàn zì

```dart
class Musician extends Performer with Musical {
  // ···
}

class Maestro extends Person
    with Musical, Aggressive, Demented {
  // ... 
}
```

Suīrán yǔyán céngmiàn shàng,mixin de shíxiàn bǐjiào fùzá, bùguò zài shǐyòng yǔyì shàng,mixin qíshí jiùshì xiǎngxiàng dì nàyàng:

```Dart
class S {
  twice(int x) => 2* x;
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
  print(a.Thrice(1));  // class B's twice
  print(c.Twice(1));   // class S's twice
}
```
 
guānyú Mixin, hòuxù hái huì zhuānmén kāi yī piān wénzhāng jièshào guānyú Dart Mixin de shèjì.

# Lèi biànliàng hé lèi fāngfǎ
tōngguò `static`guānjiàn zì, nǐ kěyǐ shíxiàn class céngjí de fāngfǎ hé biànliàng

## lèi biànliàng

```dart
class Queue {
  static const int initialCapacity = 16;
  // ···
}

void main() {
  assert(Queue.InitialCapacity == 16);
}
```

lèi zhōng de jìngtài biànliàng zhǐyǒu zài bèi yòng dào de shíhòu cái huì chūshǐhuà

## lèi fāngfǎ

```dart
import'dart:Math';

class Point {
  num x, y;
  Point(this.X, this.Y);

  static num distanceBetween(Point a, Point b) {
    var dx = a.X - b.X;
    var dy = a.Y - b.Y;
    return sqrt(dx* dx + dy* dy);
  }
}

void main() {
  var a = new Point(2, 2);
  var b = new Point(4, 4);
  var distance = Point.DistanceBetween(a, b);
  assert(2.8 < Distance&& distance < 2.9);
  Print(distance);
}
```
kàn qǐlái hé bié de biānchéng yǔyán méishénme liǎngyàng. Zhídé yī tí de shì, lèi fāngfǎ kěyǐ bèi zuòwéi chángliàng shǐyòng.

# The End
lèi shì liǎojiě miànxiàng duìxiàng shèjì de zhòngyào tújìng, cóng běn piān wénzhāng nǐ kěyǐ fāxiàn,Dart de miànxiàng duìxiàng shèjì jì jiǎndān yòu jiǎnjié, dàn yòu bù shī qiángdà xìng.
收起
4077/5000
Below is an example:

```dart
class Vector {
  final int x, y;

  const Vector(this.x, this.y);

  /// Overrides + (a + b).
  Vector operator +(Vector v) {
    return new Vector(x + v.x, y + v.y);
  }

  /// Overrides-(a-b).
  Vector operator -(Vector v) {
    return new Vector(x-v.x, y-v.y);
  }
}

void main() {
  final v = new Vector(2, 3);
  final w = new Vector(2, 2);

  // v == (2, 3)
  assert(v.x == 2 && v.y == 3);

  // v + w == (4, 5)
  assert((v + w).x == 4 && (v + w).y == 5);

  // v-w == (0, 1)
  assert((v-w).x == 0 && (v-w).y == 1);
}
```

If you need to override the `==` operator, you also need to rewrite the getter of `hashCode`, because Dart uses hashCode to determine the equality of objects.

# Implicit interface
In fact, the declaration of Dart's class will also implicitly declare the interface composed of all the attributes of this class. So assuming that class A does not support the API of class B, you can use A to implement the interface of B, and use A as B.

```dart
// A person. The implicit interface contains greet().
class Person {
  // In the interface, but visible only in this library.
  final _name;

  // Not in the interface, since this is a constructor.
  Person(this._name);

  // In the interface.
  String greet(String who) =>'Hello, $who. I am $_name.';
}

// An implementation of the Person interface.
class Impostor implements Person {
  get _name =>'';

  String greet(String who) =>'Hi $who. Do you know who I am?';
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

# Class inheritance
Use the ʻextends` keyword to implement class inheritance, and use `super` to call the parent class attribute

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

## Inheritance Overwrite
The instance method of the class can be overwritten after being inherited. The specific method is to use the @override annotation.

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
When the caller tries to call a method that does not exist on the class, the method will work, and you can override this method to tell the caller.

```dart
class A {
  // Unless you override noSuchMethod, using a
  // non-existent member results in a NoSuchMethodError.
  @override
  void noSuchMethod(Invocation invocation) {
    print('You tried to use a non-existent member: '+
        '${invocation.memberName}');
  }
}
```

# Enumeration type
Enumeration type is also a class

Declare enum type

```dart
enum Color {red, green, blue}
```

Get the value of an enumerated type

```dart
// The specific value of an enumeration type has an index like an array
assert(Color.red.index == 0);
assert(Color.green.index == 1);
assert(Color.blue.index == 2);

// All values ​​of enumeration types can also be converted into arrays
List<Color> colors = Color.values;
assert(colors[2] == Color.blue);
```

# Mixin
Mixin is a solution when the class cannot inherit multiple times.
To use mixin, you only need the `with` keyword

```dart
class Musician extends Performer with Musical {
  // ···
}

class Maestro extends Person
    with Musical, Aggressive, Demented {
  // ...
}
```

Although the implementation of mixin is more complicated at the language level, in terms of semantics, mixin is actually as imagined:

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
  print(a.thrice(1)); // class B's twice
  print(c.twice(1)); // class S's twice
}
```
 
Regarding Mixin, a follow-up article will be devoted to the design of Dart Mixin.

# Class variables and class methods
With the `static` keyword, you can implement class-level methods and variables

## Class variables

```dart
class Queue {
  static const int initialCapacity = 16;
  // ···
}

void main() {
  assert(Queue.initialCapacity == 16);
}
```

Static variables in the class are only initialized when they are used

## Class method

```dart
import'dart:math';

class Point {
  num x, y;
  Point(this.x, this.y);

  static num distanceBetween(Point a, Point b) {
    var dx = a.x-b.x;
    var dy = a.y-b.y;
    return sqrt(dx * dx + dy * dy);
  }
}

void main() {
  var a = new Point(2, 2);
  var b = new Point(4, 4);
  var distance = Point.distanceBetween(a, b);
  assert(2.8 <distance && distance <2.9);
  print(distance);
}
```
It looks no different from other programming languages. It is worth mentioning that class methods can be used as constants.

# The End
Classes are an important way to understand object-oriented design. From this article, you can find that Dart's object-oriented design is simple and concise, but it is powerful.