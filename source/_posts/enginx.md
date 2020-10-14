---
title: Client-side routing rewrite engine — enginx
date: 2017-08-29 19:50:48
tags:
---
<img src="/images/TB1MVRfLVXXXXaKaXXXXXXXXXXX-800-545-768x523.jpg" style="max-width: 350px;"/>
(The above picture is from Xindong.com)
Many mobile apps have tens of millions of users at every turn, and the dynamics of the client is a very hot topic, and I am also ready to catch up on it.

<!--more-->

Previously, referring to the practice of the Tmall team, a routing layer was also made, which is mainly used for intermodulation between business modules and modules, between remote notifications and modules, and between Web and client modules, etc. through URL protocol. But it is clear that each part has its own URL protocol format, such as:

```javascript
//Web
https://h5.ele.me/restaurants?id=9527
 
//native client
eleme://restaurant?restaurant_id=9527

```

With the growth of business modules on the client side, the isomerization of the entire protocol will be very serious, which leads to many problems. Here are a few simple scenarios:

1. Web front-end developers need to determine whether the current page is running in the Native App through a series of hacks, and decide whether to call the Native page through scheme or continue to jump to the Web page according to the bool value.
2. A bug in the calling function module of a URL scheme that is hard-coded in the App has a bug, which can only be solved by changing the URL after publishing the version.
3. Few people know how to write the complete URL for calling another module, it is likely to be misspelled, making it impossible to jump online.


Wait, there are some other minor issues, but the above is big enough. Therefore, in response to this situation, in addition to learning from the Rewrite engine of the Tmall team, I developed an SDK for URL protocol conversion, such as converting the two URLs mentioned above, but it is not that difficult to understand. Expression is a semantic DSL. I call this SDK enginx.

# enginx Introduction

In terms of name, it is to imitate related modules of nginx. The usage is similar. For a configuration file or string, input the URL string and output the target string, it's that simple.

But conceptually, enginx divides a URL into the same server and the same location domains for matching operations, which is conceptually very similar to nginx.
![](/images/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7-2017-08-29-%E4%B8%8B%E5%8D%884.04 .34-768x610.png)


I won’t explain more about the specific code references. Let’s take a look at how to rewrite a URL through a configuration file:


```javascript
server {
  domain: "ele.me";
  scheme: "https";
  port: "443";
 
  location "/" {
    if (equal $scheme "https") {
      return "https://help.ele.me";
    }
    return "https://h5.ele.me";
  }
}
```

In this way, we can solve the problems mentioned above:

1. You can rewrite the URL of the h5 domain to the Native route. When the web page is redirected, there is no need to judge by code, and the native page can also be evoked by directly redirecting to the web URL
2. If a module has a bug, you can rewrite the URL by issuing a new configuration to make it impossible to jump over and buy time for remedy.
3. By rewriting the SDK, the patchwork logic of the URL can completely fall into the configuration file code, and the UI can be realized later, and the fool-like configuration can be realized.

## URL matching

URL matching is a very important process, because you want to rewrite different URLs into different effects, and their corresponding operations are also different. The top-level domain of enginx is "server". A server corresponds to all URLs under the same domain.

But obviously not all URLs of the same domain will be rewritten in the same way, so there will be a difference in location. Location is a regular match against the URL path, and the location scope operation can only be entered after the match is completed.

So the content written in the above configuration file can match the URL https://ele.me, and the final rewrite result is https://help.ele.me. It should be understood that the domain of the server field can be matched. The "/" of the location means the root path, and students who have used nginx should know it.

## Instructions

In addition to being able to match the URL that you want to rewrite, you also need a series of instructions to operate on the URL to complete the rewriting. Therefore, when I wrote the SDK, I defined the capabilities of these commands in the configuration file, such as:

1. encode/decode, URL encode/decode a string or variable
2. match, use regular expressions to extract variables from a string or variable
3. parse, parse the string form similar to k=v to extract variables
4. var, define variables
5. greater/equal/smaller, used for comparison operations of if code blocks
6. return, return the rewritten string

## Variable
The identifier used to refer to some string literals when the instruction operates, including built-in variables and defined variables.

Built-in variables are generated when a URL is matched. For example, $scheme, $host, $query_string, etc. are all information about the URL itself.

The definition variable may be defined by you, or it may be generated by some instruction operations.

## String template
In fact, the URL itself is still a string operation, so any variables, definitions, and return values ​​are replaced by the variables in the string template. In this way, the purpose of concatenating strings can be achieved.

## Introducing enginx

enginx has implemented 2 versions in total. The first version is implemented in C++, and the configuration is delivered in JSON format, which is a little different from the above. The second version uses pure C implementation, completely new DSL format configuration files.

enginx is completely cross-platform, currently encapsulating iOS pod, Android module, and Node.js npm management library.

The first version has been used in the Ele.me App for more than half a year, without any crash or memory leak. However, a large number of regulars are used and STL libraries are introduced. The performance is not very satisfactory. The peak value is only about 5000 times/s, and the code package volume is relatively large. As a result, the second version is implemented, the second version is implemented in pure C, which can support more advanced syntax formats, the code package size is reduced by 90%, and the peak speed is increased by 6-7 times.

Currently using enginx is relatively simple, just follow the README directly: [https://github.com/stephenwzl/enginx](https://github.com/stephenwzl/enginx)

# enginx implementation
The implementation of the first version is very simple, you can switch 1.0 branch to view.

The implementation of the second edition is slightly more complicated and is divided into several parts: configuration file parser, syntax tree generation, memory management, and runtime.

## Configuration file analysis
This part looks scary, but it is actually relatively simple, because it is not a real handwritten recursive descent parser. The enginx parser uses Bison tools (Yacc and lex) to describe specific grammars through grammatical deduction, token definition, and state transition control.

The overall syntax is relatively simple, and I don't want to call this part a "compiler". You can check it in Lexer/enginx.l and Lexer/enginx.y in the repo.

## Syntax tree generation
This part is more scary, in fact it is relatively simple, mainly the use of linked lists, briefly explain the ideas:

In includes/enginx_dev.h you can see the node definition of the enginx syntax tree, including several basic types: value, location, expression, statement, server, they are essentially structures.

When parsing the syntax tree of Bison's code, you can directly call the function I wrote to mount the syntax tree node. These functions can be chain server/location and expression. So when the configuration file is parsed, the syntax tree is also generated.

## Memory Management
Because it was not written specifically for the iOS platform, I did not use NSAutoReleasePool to manage memory, but simply imitated one myself. The main function is to use the interface to allocate a section of marked memory, and call the check method at the end of the runtime to check for meaningless memory. The essence is not difficult, you can see the specific implementation in the memory folder.

## Runtime
The essence of runtime is to get the URL and the root node of the syntax tree, perform a match, and then execute the expression logic in the syntax tree. This part is also relatively simple, so I won't introduce more, you can find it directly in implements/enginx.c.

## The End

enginx is a cross-platform practice for me. I may not consider that much in terms of documentation and versatility, but at least it solves the problem of semantic configuration.

Stop rewriting various URLs with regular expressions