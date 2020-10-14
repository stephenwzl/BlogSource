---
title: Dart function
date: 2018-04-23 19:09:46
tags:
---

Function is the basic unit of Dart program operation. In this article, we will systematically understand Dart's functions.

<!--more-->
Dart is a true object-oriented programming language, so a function is also an instance of an object, and it also has a type called `Function`. This means that functions can also be assigned to variables or passed as function parameters.
A simple function is as follows:

```dart
bool isZero(int number) {
  return number == 0;
}
```

Although the official Dart tutorial wants you to add a type to the return of the function, you can still ignore the return type and let the compiler infer it by itself:

```dart
isZero(int number) {
  return number == 0;
}
```

If there is only one line of expression like the above code, you can use arrow functions to express:

```dart
bool isZero(int number) => number == 0;
```

# Optional parameters
Optional parameters actually have two meanings

## Optional named parameters
What is named? See code

```dart
// defines a function like this
void enableFlags(bool bold, bool hidden) {
  // ...
}
// If there are many parameters and their types are similar, you don't know which parameter corresponds to which position when you use it
enableFlags(true, true);

// Hope it is the following usage with parameter names
enableFlags(bold: true, hidden: true);
```

To achieve the usage of the named function, add `{}` to the parameter when defining

```dart
void enableFlags({bool bold, bool hidden}) {
  // ...
}
```

## Optional positional parameters
The difference from JavaScript is that Dart **certain positions** ignorable parameters must be specified with `[]` symbols in the function definition:

```dart
// The device parameter can be ignored when calling
String say(String from, String msg, [String device]) {
  var result ='$from says $msg';
  if (device != null) {
    result ='$result with a $device';
  }
  return result;
}
```

# Parameter default value
Like other languages, you can add a default value to the function form. If the parameter is not passed, the default value is `null`

```dart
void enableFlags({bool bold = false, bool hidden = false}) {
  // ...
}

// bold = true, hidden = false
enableFlags(bold: true);
```

# main function
The main function is the entry point of a dart application. The return type of the main function is `void`, and optional parameters are passed in as a `List<String>` type.

```dart
// Call method:
// dart demo.dart 1 test
void main(List<String> arguments) {
  print(arguments);

  assert(arguments.length == 2);
  assert(int.parse(arguments[0]) == 1);
  assert(arguments[1] =='test');
}
```

If you want to use Dart to write a command line tool, you can use [args library](https://pub.dartlang.org/packages/args) to parse command line parameters.

# Anonymous function
The functions seen above are all named. Of course, Dart can also use anonymous functions.
For example, use an anonymous function to iterate the List

```dart
void main() {
  var list = ['apples','bananas','oranges'];
  list.forEach((item) {
    print('${list.indexOf(item)}: $item');
  });
}
```

# Semantic scope
The content of this part is relatively simple and has already been recognized in [Dart Introduction](/2018/04/20/dart-getting-start/), so I won't repeat it.

# return value
All functions in Dart have return values. If the return value of an expression or function is not specified, the default is `null`.

# Function cascade
The cascade of functions is also called chain call. Dart's native syntax supports chained calls, just use `..`

```dart
querySelector('#confirm') // Get an object.
  ..text ='Confirm' // Use its members.
  ..classes.add('important')
  ..onClick.listen((e) => window.alert('Confirmed!'));
```

# The End