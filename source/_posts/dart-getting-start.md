---
title: Know Dart language
date: 2018-04-20 15:06:19
tags:
---

<img src="/images/dart-logo.svg" style="max-width: 400px;"/>
<br/>
  
Dart is an open source programming language developed and maintained by the Google Chrome team.
After a period of simple trial and use, it can indeed make people feel that this is a programming language designed to improve the client development experience, bringing a glimmer of dawn to us who have been suffering from JavaScript for a long time.
This article introduces the basics of Dart language to use.

<!--more-->

# Install Dart
Since there is no Linux machine at hand, I will introduce the installation of macOS environment.
Although Dart has developed to **Dart2**, many new features have been added, but the essence has not changed much. In order to better understand the Dart standards, we directly install the default Dart SDK.

> Please make sure homebrew is installed

Open the terminal App and enter:

```shell
brew tap dart-lang/dart
brew install dart
```

> As of the date of posting, Dart's default SDK is still not Dart2, but don't worry, it hasn't changed much. Let's start with 1.

# Choose IDE
If you want to do your job well, you must first sharpen your tools. It is very important to choose a suitable IDE. Atom, Sublime Text, VSCode, Intellij IDEA all have Dart plugins. The author chose Intellij IDEA as the IDE for this article, and installed `Dart Plugin`.
Open IDEA's `Preference` -> `Languages ​​& Frameworks` -> `Dart`, configure the Dart SDK path:
![](/images/dart-intellij-idea.png)

After configuration, IDEA can prompt you correctly for standard library functions, and can help you do language analysis, so it is very comfortable to write.



# First line Dart

In order to experience Dart simply, we can directly create a new `demo.dart` file, which can be opened in IDEA. A basic hello world is as follows:

```dart
// demo.dart

void main() {
  print('hello world');
}
```

Then execute `dart demo.dart` on the command line, the command line will output:

```
dart demo.dart ↵
hello world
```

Dart, like most compiled languages, requires main as the entrypoint of language execution, rather than interpretation and execution like JavaScript. So obviously, Dart is a compiled language.
But we did not compile first when we ran demo.dart. This is because Dart runs through the VM. When the source code is loaded, it is compiled into bytecode and then executed by bytecode. The advantage of this is that the execution speed will be faster than JIT. The code is much faster. The disadvantage is that the speed of the code from loading to actual execution is relatively slow. Therefore, Dart also supports compilation into AOT code, so that the speed of code loading and running can be greatly improved.

# Dart grammar
After knowing how to simply run dart code, let's take a look at the syntax of Dart.
## Variables and types
In Dart, we can declare a variable with `var` or a specific type:

```dart
var name ='Bob';
int num = 9527;
```

By default, the values ​​of uninitialized variables are all `null`

```dart
int count;
print(count == null); // true
```
This way we don't have to worry about being unable to determine whether a passed variable is ʻundefined` or `null` in some cases. Because in Dart, all undefined cases are null.

Dart is a type-safe language, and all types are object types. The top level of all types comes from ʻObject`. Of course, `null` is also an object. Therefore, the `var` keyword is only a reference to a type, and the object declared using it is only a reference to the object type. This process is completed by the compiler and does not need to be concerned.
Dart has relatively simple built-in types, such as Number, String, Boolean, List, Map, Runes, Symbols, so I won't introduce them one by one.

## Functions and closures
Function is the basic unit of Dart execution, expressions outside the function cannot be executed correctly.
A basic function is as follows:

```dart
void sayHello() {
  print('hello');
}
```

The format is the same as in most languages. It is worth mentioning that if the return value is `void`, then `void` can be omitted.
If your function body has only one line of return value, you can use **arrow function**:

```
bool isZero(int num) => num == 0;
```

A function is actually an object, you can pass or assign a function as a parameter:

```dart
var loudify = (msg) =>'!!! ${msg.toUpperCase()} !!!';
print(loudify('hello') =='!!! HELLO !!!');
```

Of course, you can also use **anonymous functions** to iterate the List:

```dart
var list = ['apples','bananas','oranges'];
list.forEach((item) {
  print('${list.indexOf(item)}: $item');
});
```

Next, I want to mention one of the more friendly scopes in Dart, which is the semantic scope, which is vividly reflected in the closure:

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

In the above code, the ʻadd2` variable gets a closure of `(num i) => 2 + i;`, and ʻadd4` gets a closure of `(num i) => 4 + i;`. This logic is completely consistent with the semantics of your code, if you are in other languages, you may not be so lucky.
The benefit of semantic scope is that you can code with confidence without worrying about pitfalls in the language.

## Operator
Dart's operators are the same as those of most languages, so you can code them in a familiar way. But Dart has a few more interesting operators.

**`?.` Operator**: Suppose that the Person class has a `sayHello()` function, and p is an instance of Person. Then `p?.sayHello()` means avoid throwing an exception when p is null
**ʻIs` operator**: If p is an instance of the Person class, then `p is Person` returns true
**`??` Operator**: ʻa ?? b` expression means that if a is not null, return the value of a, otherwise return b
**`??=`Operator**: `b ??= value` means that if b is null, assign a value to b, otherwise do not assign a value

## Process control
Dart maintains the standard flow control syntax: ʻif-else`, `for`, `while`, `do-while`, `break/continue`, `switch-case`, ʻassert`.
It is worth mentioning that ʻassert`, which can prevent your code execution flow (similar to assert in C), but it is only useful in Dart's check mode (similar to a development environment), and not in production mode.

## Class
In Dart, declaring and using a class is simple:

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

**Constructor**: Dart's constructor is very convenient, you can write it easily

```dart
class Point {
  num x, y;

  // Common standard constructor
  Point(num x, num y) {
    this.x = x;
    this.y = y;
  }
  
  // Syntactic sugar, the same as above
  Point(this.x, this.y);
  
  // You can name a special constructor
  Point.origin() {
  x = 0;
  y = 0;
  }
  
  // Initialize the member list, you cannot use this keyword to assign values ​​to x/y here
  Point()
    : x = 0,
      y = 0 {
    print('Initialize member list');
  }
}
```

It is worth mentioning that Dart does not have keywords such as `public` and `private`, and the method name can be used as a private method with `_` in front of it. The default is public.
Dart also supports abstract classes, so that we can agree on some agreements, and then put the concrete implementation in other places.

```dart
abstract class AbstractContainer {
  // Here you can define constructors, member variables, method waits
  void updateChildren(); // Abstract method.
}

class SpecializedContainer extends AbstractContainer {
  void updateChildren() {
    // The implementation of the method is provided here, so that the method is not abstract
  }
  // Unimplemented abstract methods will throw a warning, but it does not prevent object instantiation
}
```
Dart's classes are very powerful and limited in space. Later, I will open an article to introduce more powerful features.

## Error handling
Dart's error handling is also very Modern. Because Dart is type-safe, error handling can be based on type, which is more convenient:

```dart
// demo
try {
  breedMoreLlamas();
} on OutOfLlamasException {
  // specific type of exception
  buyMoreLlamas();
} on Exception catch (e) {
  // Not sure what type of exception
  print('Unknown exception: $e');
} catch (e) {
  // No type, unified processing
  print('Something really unknown: $e');
}
```

## Library
The ecology of a programming language depends on its library. Let's take a look at the syntax imported by the Dart library.

```dart
// Import html library
import'dart:html';

// Import only part
import'package:test/test.dart';

// give an alias
import'package:lib2/lib2.dart' as lib2;

// Only import foo in this library
import'package:lib1/lib1.dart' show foo;

// Import everything except foo
import'package:lib2/lib2.dart' hide foo;

// lazy loading
import'package:greetings/hello.dart' deferred as hello;
```

How to implement a library may be a bit longer, so I plan to put it in another article.

# The End
This article is mainly to explore the slightly different places in the Dart language, so that beginners will be easier to get started. Dart actually has a lot of advanced content. To avoid confusion, I plan to introduce all of them in separate articles.
In this way, we can have a clear and simple learning route for Dart.