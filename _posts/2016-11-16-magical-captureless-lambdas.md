---
title:  "Lambda Magic ✨"
categories: [C++]
tags: [C++, C++11, lambdas]
---
C++ lambdas are *magical*. They're totally [*splendiferous*][Splendiferous].  

![Magic](../../assets/magic.jpg)  
But, at the end of the day, they are nothing but syntactic-sugar for creating function objects (you *could* even write them yourself (and pre-C++11 we *did* (... or more likely *didn't*))).

Well, almost -- because *captureless* lambdas *are* magical.  

> *Like faeries, captureless lambdas are pure and magical creatures.  
Unlike faeries, captureless lambdas can be converted to function pointers.*  

Automtic function pointer conversion is very useful when interacting with C APIs that require function pointer arguments. One notorious example is [`qsort()`][qsort] (you should always prefer [`std::sort`][sort] where possible). Similarly, many C APIs require a function pointer callback.  

<p style="text-align: center;">
  <img src="http://i.giphy.com/M13G8Iq8OHOZG.gif"/>
</p>

A further magic trick that captureless lambdas can perform is to convert to a function pointer of *any* desired function [*calling convention*](https://en.wikipedia.org/wiki/Calling_convention). This may be of more interest to 32-bit Windows developers, though calling conventions are more intimiately related to the processor architecture than the OS.

Calling conventions behave as part of the function signature. Given a function or function pointer of one calling convention `static_cast<>`ing it to another will not compile and using `reinterpret_cast<>` would most likely result in a crash or **Undefined Beahaviour**.

This code does not compile (on 32-bit x86):

```cpp
int f(int) {return 0;}          // __cdecl is the default
int (           *fn1)(int) = f; // OK
int (__cdecl    *fn2)(int) = f; // OK
int (__stdcall  *fn3)(int) = f; // Nope!
int (__fastcall *fn4)(int) = f; // Nope!
```

However, with lambdas *Voilà!*

```cpp
auto f = [](int)->int{return 0;};
int (           *fn1)(int) = f;
int (__cdecl    *fn2)(int) = f;
int (__stdcall  *fn3)(int) = f;
int (__fastcall *fn4)(int) = f;
```

Visual Studio happily converts the lambda to the various function pointers even on x86 builds. This is type safe and should work as expected.

In the next post, I will investigate how "pure" captureless lambdas *really* are. I'll also dive deeper into using lambdas as callbacks.  
Stay tuned.

<p style="text-align: center;">🎩</p>


*Acknowledgments:
[banner](http://www.baltana.com/fantasy/magic-desktop-wallpaper-04592.html)*


[Splendiferous]:      http://www.oed.com/view/Entry/187134
[qsort]: http://en.cppreference.com/w/c/algorithm/qsort
[sort]: [http://en.cppreference.com/w/cpp/algorithm/sort]