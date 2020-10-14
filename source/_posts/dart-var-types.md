---
title: Dart variables and types
date: 2018-04-21 12:11:01
tags:
---
In [previous article](/2018/04/20/dart-getting-start/), we briefly introduced and met the Dart language.

In this article, we will systematically understand the important components of the Dart programming language: variables and types.

<!--more-->

# Variable
As you know above, the syntax for defining variables is very simple, such as:

```dart
var name ='Bob';
int line = 0; // define int type
String foo ='Bar'; // Define String type
List counts = [1, 2, 3]; // Define List type
```

> In Dart, variables are all reference types, which means that all variables are objects, so Dart is a completely object-oriented language.

Dart is type safe, so when you use the `var` keyword to define a variable, the essence is actually a reference to a specific type. For example, the above code is actually a reference to a `String` type object, and the content of this object is'Bob'

## Uninitialized variables
The values ​​of uninitialized variables are all null, which is reassuring in Dart.

```dart
int lineCount;
assert(lineCount == null);
```

## Type optional
When you use `var` to define a variable, it means that the type is determined by the compiler. Of course, you can also use a static type to define a variable.
The advantage of using static types to define variables is that you can express your intentions more clearly to the compiler, so that the compiler and editor can use these types to provide you with warnings or code completion functions.

But in Dart2, types are no longer optional. This means that when you use `var` to define a name variable, this type will be constrained to the String type, and an error will occur if you assign other types. If you don't want to constrain the type of a variable, you can use the ʻObject` type definition or the `dynamic` keyword definition.

## final and const
If you want to define immutable variables, add the `final` or `const` keywords before the definition.

```dart
final nickname ='ccc';
const int count = 3;
const List list = const [];
```


`const` means that the value of the variable can be determined during compilation. If the variable is in `class`, it is marked with `static const`. When using `const` to mark a variable, you need to specify its value when you declare it.

`final` is somewhat different, the variables marked with it can be determined at runtime, for example:

```
var x = 100; // Runtime variable, can be any other value
var y = 30; // Runtime variable, can be any other value

final res = x / y;
```
`final` is actually declaring an immutable reference at runtime, once the reference is confirmed, it cannot be changed.


# Types of
Dart has built-in the following types:

* Number
* String
* Boolean
* List
* Map
* Rune
* Symbol

You can use these types to declare variables without introducing other libraries.

## Number
There are actually only two types of Dart's number: int and double
The range of int is -2^53 ~ 2^53
double is a 64-bit double-precision floating point number conforming to the IEEE 754 standard.

ʻInt` and `double` are both subtypes of `num`. In addition to the common operators, you can also use the built-in methods such as ʻabs()`, `ceil()`, and `floor()`. If you need other calculation methods, you can introduce the `dart:math` library to try it out.

## String
Dart's String is actually composed of UTF-16 code strings. Like JavaScript, you can use either single quotes or double quotes to write string literals

```dart
var s1 ='Single quotes work well for string literals.';
var s2 = "Double quotes work just as well.";
var s3 ='It\'s easy to escape the string delimiter.';
var s4 = "It's even easier to use the other delimiter.";
```

You can also embed variables in strings. In addition to String variables, Dart will call its `toString()` method by default for objects embedded in strings:

```dart
var s ='string interpolation';
var s1 ='this is uppercased string: ${s.toUpperCase()}'
```

The string concatenation directly uses the `+` operator, so I won't mention this.
It is worth mentioning that when declaring a multi-line string, you can use three single quotes or three double quotes, which is a bit like Python:

```dart
var s1 ='''
You can create
multi-line strings like this one.
''';

var s2 = """This is also a
multi-line string.""";
```

## Rune
When it comes to String, I have to mention Rune. Rune is actually a UTF-32 string in Dart.
Because Unicode defines a special numeric value for each word, character or symbol, it is more difficult to use String to represent 32-bit Unicode.
Usually, the form of a Unicode code is \uXXXX, this XXXX represents a 4-digit hexadecimal number. For example, the Unicode code of the character (♥) is `\u2665`, but if the code to be expressed exceeds or is less than 4 digits, it must be enclosed in curly braces. For example, 😆 is `\u{1f600}`.

Rune actually helps String to represent 32-bit Unicode. For example:

```dart
main() {
  var clapping ='\u{1f44f}';
  print(clapping);
  print(clapping.codeUnits);
  print(clapping.runes.toList());

  Runes input = new Runes(
      '\u2665 \u{1f605} \u{1f60e} \u{1f47b} \u{1f596} \u{1f44d}');
  print(new String.fromCharCodes(input));
}
```

The output is:

```
👏
[55357, 56399]
[128079]
♥ 😅 😎 👻 🖖 👍
```

It can be clearly seen that the `codeUnits` function of String can only return 16-bit codes, but `Rune` can represent 32 bits.

## Boolean
Dart's bool type has only two values: true and false
The difference from other languages ​​is that when the judgment is executed, only true in Dart will judge true, and any other value is false

```dart
var name ='Bob';
if (name) {
  // will not be executed here in Dart
  print('You have a name!');
}
```

## List
You can also call List Array, but in Dart, array is called List.

```dart
var list = [1, 2, 3];
```

Dart's List actually has types. For example, the type of the above code is `List<int>`, so the elements you add to it must be of type int.

Like other languages, the array index also starts from 0. Similarly, you can iterate arrays like JavaScript:

```dart
list.forEach((item) {
  print('index: ${list.indexOf(item)}, item:${item}');
});
```

In addition to some basic functions, Dart actually has many ways to manipulate collection types. For details, please refer to the document: [Collections in Dart](https://v1-dartlang-org.firebaseapp.com/guides/libraries/library- tour#collections)

## Map
The Map type is also called a dictionary in other languages, which is a collection of key-values. Declaring Map in Dart is also very simple:

```dart
var gifts = {
  // Key: Value
  'first':'partridge',
  'second':'turtledoves',
  'fifth':'golden rings'
};

// or

var gifts = new Map();
gifts['first'] ='partridge';
gifts['second'] ='turtledoves';
gifts['fifth'] ='golden rings';

```
Similarly, Map also has types, the above code is Map<String, String> type. But many times we do not use Map in accordance with the established generics. If you read the above carefully, you will find that the solution is also very simple.
For the operation method of Map, you can also refer to [Collections in Dart](https://v1-dartlang-org.firebaseapp.com/guides/libraries/library-tour#collections)

## Symbol

Symbo is a special type in Dart, you generally don't use it, but if you need to get the literal value of the identifier, this effect is incalculable.

```dart
var name ='a';
Symbol s = #name;
```

So under normal circumstances, it has no effect, but the advanced usage of this thing will be mentioned later.


# The End
Through the understanding of variables and types, we can realize:

* In Dart, all variables are objects, and all objects are instances of classes, and null is no exception. All classes inherit from ʻObject`
* Try not to be lazy, specify the type for the variable, so that the compiler and editor can better help you write the code.