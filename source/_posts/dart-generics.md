---
title: Generics in Dart
date: 2018-04-27 17:38:31
tags:
---

Since types can be omitted in Dart 1.x, you don't need to use generics. But if you want to make code hints more friendly, generics are still necessary to some extent.
This is like in Objective-C, a `NSArray<NSString>` and a `NSArray` have completely different effects.

<!--more-->

# Why are you obsessed with generics?
For example 🌰, if you want to declare a List that can only add strings, then you can use the `List<String>` declaration. In this way, other code touches that use your code can avoid stuffing other types of objects in the List, because once stuffed, the editor will give hints.

```dart
var names = new List<String>();
names.addAll(['Seth','Kathy','Lars']);
names.add(42); // IDE warning
```

One more 🌰, using generics can improve code reuse. For example, you want to declare a memory cache interface:

```dart
abstract class ObjectCache {
  Object getByKey(String key);
  void setByKey(String key, Object value);
}
```

But this time you find that you need an interface of type String, not of type Object, so you have to create another one:

```dart
abstract class StringCache {
  String getByKey(String key);
  void setByKey(String key, String value);
}
```

> Double commit, will your KPI be double if you double the code? Naive! Others will spray your feet

What if you need to declare another number type? Do you do it again? Obviously generics are designed for this kind of scenario:

```dart
abstract class Cache<T> {
  T getByKey(String key);
  void setByKey(String key, T value);
}
```

Like generics in other languages, T here is just a substitute for a concrete type, and you will replace this T with a concrete type when you use it.

# Use generics for type checking for collection literals
When we write the literal amount in the code, we often do whatever we want, which will lead to a wrong type of literal and will bury security risks. When you add a generic type to a literal, you can let the static analyzer help you check for errors in the literal.

```dart
var names = <String>['Seth','Kathy','Lars'];
var pages = <String, String>{
  'index.html':'Homepage',
  'robots.txt':'Hints for web robots',
  'humans.txt':'We are people, not machines'
};
```

# Add parameterized types to the constructor
Like collection types, similar to Java, you can add constraints to its type when you use the constructor to initialize the collection instance object.
```dart
var names = new List<String>();
names.addAll(['Seth','Kathy','Lars']);
var nameSet = new Set<String>.from(names);
var views = new Map<int, View>();
```

# Determine the collection type
The type of the collection in Dart is specific, which means that after you pass the generic constraint on the collection, it is the corresponding type.

```dart
var names = new List<String>();
names.addAll(['Seth','Kathy','Lars']);
print(names is List<String>); // true
```

> But sometimes the items in the List are not really all of the String type, so it is best to judge the type of each item when using it.

# Constraint parameter type
Sometimes you can use type constraints when the type you expect in a generic implementation similar to a generic interface is some specific type

```dart
// T type must be SomeClass type or its subclass
class Foo<T extends SomeBaseClass> {
  // ···
}

class Extender extends SomeBaseClass {
  // ···
}

void main() {
  // You can use SomeBaseClass type
  var someBaseClassFoo = new Foo<SomeBaseClass>();
  var extenderFoo = new Foo<Extender>();
}

```

# Use generic functions
At the beginning, Dart's generics were limited to the realization of the class level, but later it also supported generic functions.

```dart
T first<T>(List<T> ts) {
  // Do some initial work or error checking, then...
  T tmp = ts[0];
  // Do some additional checking or processing...
  return tmp;
}
```

# The End
There are actually many ways to play generics, the above introduction still focuses on the level of code reuse and type checking. More usage can not only absorb experience from other languages ​​such as C++, but also refer to some discussions in the community.