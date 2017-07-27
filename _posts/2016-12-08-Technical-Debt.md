---
title:  "Technical Debt"
tags: [C++, lambdas, technical-debt]
categories: [C++]
---
My series on captureless lambdas generated some interesting comments (some via Twitter and Reddit). Here are some followups.  

![](../../assets/issue_list.jpg)

### Easy Cast
We can convert captureless lambdas into function pointers without explicitly specifying the cast-to type by prepending them with a `+`.

Consider this code (from [here](http://stackoverflow.com/questions/17822131/resolving-ambiguous-overload-on-function-pointer-and-stdfunction-for-a-lambda)):

```cpp
#include <functional>

void foo(std::function<void()> f) { f(); }
void foo(void (*f)()) { f(); }

int main ()
{
    foo(  [](){} ); // COMPILATION ERROR: ambiguous
    foo( +[](){} ); // not ambiguous (calls the function pointer overload)
}
```
As explained in the [answer](http://stackoverflow.com/a/17822241/135862):  
The first unadorned call is ambiguous since it matches both `foo` overloads.  
The second is unambiguous since we have an exact match.  
See that [answer](http://stackoverflow.com/a/17822241/135862) for more details why and how this works.


### Calling Conventions
In my [Lambda Magic ✨ post](https://girishnayak12.github.io/2016/magical-captureless-lambdas/), I mentioned that on 32-bit *Windows* captureless lambdas not only implicitly convert to function pointers, but in fact they can assume any calling convention desired.  

However, consider the following code:

```cpp
template< typename T >
void foo(T* f) { f(); }

int main()
{
   foo(+[]{}); // cast lambda to (unspecified) function pointer
}
```
On 32-bit MSVC, this code fails:

```
error C2593: 'operator +' is ambiguous
note: could be 'built-in C++ operator+(void (__cdecl *)(void))'
note: or       'built-in C++ operator+(void (__stdcall *)(void))'
note: or       'built-in C++ operator+(void (__fastcall *)(void))'
note: or       'built-in C++ operator+(void (__vectorcall *)(void))'
note: while trying to match the argument list '(main::<lambda_...>)'
``` 
Visual C++ performs automatic conversion to any calling convention, but in this example (and the previous one too), the target function pointer calling convention is unspecified, so the compiler fails on ambiguity.  
This is, in fact, un-comformant as this code is perfectly valid C++ and is gladly accepted by 32-bit Clang.  

Had Visual C++ chosen a default calling convention (which it does for any function where it isn't specified anyway) it would have become conformant while still providing this practical extension for working with legacy libraries.

Interestingly, since the calling convention is *not* an official part of the type system, Clang balks at these conversions and will not allow a lambda to be cast into a function pointer with a different calling convention. 
 
Thus, this code does not work on clang:

```cpp
auto f = [](int) { return 0; };
int (__stdcall  *fn1)(int) = f;
int (__fastcall *fn2)(int) = f;
```
There is no viable conversion:

```
error: no viable conversion from '(lambda...)' to 'int (*)(int) __attribute__((stdcall))'
error: no viable conversion from '(lambda...)' to 'int (*)(int) __attribute__((fastcall))'
```
So, if you're using Clang and have to use a library with a different calling conventions, automatic lambda to function pointer conversions do not work.   
However, such systems are probably not as common on non-Windows environments, and these issues have become less relevant with pervasive x64 OSs.

Here's a [⚡godbolt⚡](https://godbolt.org/g/FBgT98) demonstrating these two compilation errors on the two compilers with a 32-bit build.

### Callbackize
Following my [Lambda Callbacks📞](https://girishnayak12.github.io/2016/lambdas-callbacks/) post, [Vaughn Cato](https://twitter.com/girishnayak12/status/804246649006686208) came up with an even cleaner implementation of a function to convert any capturing lambda into a function-pointer. Here it is with some tweaks:

```cpp
template <typename Lambda>
static auto callback(Lambda &&l)
{
  thread_local auto* p = &l; // initial assignment, allows using auto
  p = &l;
  return [](int arg){ return (*p)(arg); };
}
```
The template *function* `callback` will take a lambda (rvalue, lvalue, mutable or not, const or not) and return a captureless lambda that can be cast to a function pointer.  
It works the same way as `payload_injector` in the previous post does, but provides a nicer interface:

- `callback()` is a function and not a template. This allows the use of automatic type deduction.
- It is `static` to create a file-local function, this equivalent to the anonymous namespace.  
- It creates the *thread local static* variable *inside* the function, so that it does not need define it externally as a standalone `static` member of a class.  
- Like all static variables, it is only initialized once upon the first call to the function (this is guaranteed by the compiler). If the function is called twice with the same lambda, the initialization will be skipped on the second call. Hence, the second initialization `p = &l;` is essential.   
The *first* static initialization assignment is done to allow us to use `auto*` with the correct type deduction, as `Lambda` may or may not be `const`, `mutable` etc. (If we really wanted to get rid of this single pointer assignment we could have declared `p` of type `std::add_pointer<decltype(l)>::type` or something like that.)
 
It is used like this:

```cpp
// a function expecting a function pointer
void f(void (*fun)(int), int y)
{
  fun(y);
}

int main()
{
  auto x = 42;
  auto lamb = [&](int y) // capturing lambda
  {
      std::cerr << "Baaaa... " << "x = " << x << ", y = " << y << '\n';
  };
  
  // call f() converting lamb to a function pointer
  f(callback(lamb), 0);  

  x = 24;
  
  f(+callback(lamb), 1);    
}
``` 
The [output](http://coliru.stacked-crooked.com/a/0b14542f14e58fe6) is: 

```
Baaaa... x = 42, y = 0
Baaaa... x = 24, y = 1
```

As usual, I ❤ feedback. Please use the comments below, Twitter or Reddit.


*Extra credits: [banner](https://www.flickr.com/photos/tidymind/4327328037)*
