---
title:  "Leaky 🕳 Lambdas "
categories: [C++]
tags: [C++, C++11, lambdas]
---
There is a whole host of powerful closure critters that can squeeze into a captureless lambda.

![Magic](../../assets/colander.jpg)  

In my [last post](https://adishavit.github.io/2016/magical-captureless-lambdas/) I posited: 

> Like faeries, captureless lambdas are pure and magical creatures.  
Unlike faeries, captureless lambdas can be converted to function pointers.

Well, I lied.  

While they may be *magical*, they are not entirely *pure*.  
Alas, that is not their fault -- they are, after all, a type of C++ functions.

It turns out that even *captureless lambdas* (which are e.g. convertible to function pointers) can see, hear, sniff and *use* certain things outside their own scope in their enclosing closure!

These sneaky entities can deviously waltz right past the lambda's *empty* capture-list and appear within the lambda body.

<p style="text-align: center;">
  <img src="http://i.giphy.com/hqJVGyeJtRCmc.gif"/>
</p>

### Types and Namespaces

Obviously, external types and namespaces are visible through the closure:
  
```cpp
struct A {};

auto f1 = []()
{
  A a1;	// A is visible
  
  struct B {}; // local struct
  
  auto f2 = []()
  {
  	A a2;             // A is visible here too
  	B b1;             // The local type B is also visible and usable
  	std::vector<A> v; // std::vector<> from the std namespace
  };  
  f2();
};
```
No surprises here, just like any C/C++ function, but extended to see local types (because as opposed to lambdas, regular C/C++ functions cannot be defined locally (though function-object types can)).  

### Functions
Similarly, functions are also accessible:

```cpp
int foo() { return 1; } // a function

auto bar = []()
{
  return foo(); // accessible here
}();
```


### Globals
Another unsurprising case is the visibility of global objects.  
Although using them is frowned upon, globals are part of our C legacy, and thus supported:

```cpp
struct Foo 
{
  int foo(){ return 0; }
};

Foo globj; // global object

namespace Bar
{
  Foo barbj; // a global within a namespace 
}

auto lambda = []()
{
  return    ::globj.foo() + // use a global object with empty capture list
         Bar::barbj.foo();  // including from within accessible namespaces
};

int (*fun_ptr)() = lambda; // conversion works
```
  
### Static Members
If both types *and* globals are accessible, then it is only logical that `static` members are also accessible:

```cpp
struct Foo
{
  static int bar; // static class member
};

int Foo::bar = 42; // define the static member (to pacify the linker)

auto lambda = []()
{
    return Foo::bar; // static member accessible here
};
```

### Constants
Often we have constant values that are known at ***compile-time***. These *too* are visible:

```cpp
int main()
{       
    const     int k1 = 1;   // a local const value - known at compile time
    constexpr int k2 = 2;   // a local const value - known at compile time
    enum        { k3 = 3 }; // a local enum - a constant value known at compile time
    
    auto f = []()
    {
        return k1 + k2 + k3; // constant values can be used here!
    };
    
    return f();
}
```
We shall return to constants in a moment.

### Ordinary Variables?

Check out this valid, [working](http://melpon.org/wandbox/permlink/unlBh2ltOD6Mbyf6), code:

```cpp
int main()
{       
    int x = 42;
  
    auto f = []()
    {
        return sizeof(x); // local variable `x` from external closure is visible here!!!
    };
    
    return f();
}
```

> ***Say, what?!***  
> *This "leakage" thing seems too be getting out go control!  
Why have lambda-captures at all if this code works!*

## Say Cheese 🧀
By now, our once seemingly *pure* captureless lambdas might seem like veritable Swiss cheese.

In the code above notice that the lambda is actually returning `sizeof(x)` - a value that is known at compile time. The lambda *never* actually tries to look at the *value* of `x`. 
So this case is, in fact, like the case of the compile time constants above.

Furthermore, consider this example which seems similar to the two examples above:

```cpp
void f(int v)
{       
    const     int k1 = 1;   // a local const value - known at compile time
    constexpr int k2 = 2;   // a local const value - known at compile time
    const     int k3 = v;   // a local const value - known ONLY at run time
    int           x  = v;   // a local variable    - known ONLY at run time
  
    auto f1 = []() { return x;   }; // error: 'x' is not captured
    auto f2 = []() { return k3;  }; // error: 'k3' is not captured
    auto f3 = []() { return &k2; }; // error: lvalue required as unary '&' operand
    auto f4 = []() { return &k1; }; // error: lvalue required as unary '&' operand
}
```

Finally it is obvious that there's a limit to what can sneak into a captureless lambda.

- When `f1` tries to use the *run-time* value of `x`, an error is emitted that `x` is not captured.  
- Although `k3` is defined as `const`, its value is set at run-time and is not known at compile time, so when `f2` tries to use the *run-time* value of `k3`, an error is emitted that `k3` must be captured.  
- Even for constant variables with values known at compile time like `k2` and `k1`, if we try to reference an actual variable (i.e. address) for them (and not just the known compile-time value), we must bind and capture them in a dereferencable lvalue.

One might consider this a sufficient fix:

```cpp
    auto f1 = [&]() { return x;   }; // ok
    auto f2 = [&]() { return k3;  }; // ok
    auto f3 = [&]() { return &k2; }; // error: lvalue required as unary '&' operand
    auto f4 = [&]() { return &k1; }; // error: lvalue required as unary '&' operand
```

This is ok for `f1` and `f2` but default captures (`[&]` and `[=]`) will not bind the compile time constants `k2` and `k1` to a named (lvalue) variable automatically. To get that code to compile we must explicitly name the captured values:

```cpp
    auto f3 = [k2]() { return &k2; }; // compiles ok, but is it?
    auto f4 = [k1]() { return &k1; }; // compiles ok, but is it? 
```
or alternatively:

```cpp
    auto f3 = [&k2]() { return &k2; }; // compiles ok, but is it?
    auto f4 = [&k1]() { return &k1; }; // compiles ok, but is it?
```
This code compiles fine as the names are explicitly captured and thus valid within the lambda body.  

--- 

> ### Pop-Quiz!
>
>As pointed out by the astute [Jason Turner](https://twitter.com/lefticus) , there are some unrelated [subtle issues]((https://twitter.com/lefticus/status/801142776951959553)) (don't peek if you dare) with these particular *non*-captureless lambda examples:
>
1. What exactly are these lambdas returning? 
2. What is the lifetime of the returned values? 
3. What do these addresses point to? 
4. Are they even valid and if so until when?
5. Why are there no compiler warnings?
>
> This post is about *captureless* lambdas, and explaining the subtleties of these examples of *captureful-lambdas* is somewhat beyond its scope.  
> Go crazy in the comments.  
> If there's interest I'll write a followup post detailing this.

For completeness, here's a version that does not have the subtleties mentioned above:

```cpp
auto g(const int* p) { return p ? *p : 0; }; // a function taking an address
void f(int v)
{       
    const     int k1 = 1;   // a local const value - known at compile time
    constexpr int k2 = 2;   // a local const value - known at compile time

    auto f3  = [ k2]() { return g(&k2); }; 
    auto ff3 = [&k2]() { return g(&k2); }; 
    auto f4  = [ k1]() { return g(&k1); }; 
    auto ff4 = [&k1]() { return g(&k1); }; 
}
```

---

## The Mysterious ODR-Use
The rules governing what entities are visible within a lambda (any lambda not just a captureless one) are formally known as ***ODR-Use***.  

Colloquially, an entity that is mentioned or *used* but is not ***ODR-used*** within the lambda body, does not need to be captured in the capture list.  

> *Informally, an object is **odr-used** if its address is taken, or a reference is bound to it.  
> [--here](http://en.cppreference.com/w/cpp/language/definition)*

Aha! That sounds familiar, it's just what we just saw with our last example!  
Basically, as long as we don't access or require an entity's actual addressable, in-memory, run-time value we can avoid capturing it. This is very important when dealing with captureless lambdas.

Unfortunately, the standardese for ODR-Use is quite intensive and way beyond the scope of this post. For more details I found these links helpful:

1. From [cppreference.com](cppreference.com)
  - [ODR-Use](http://en.cppreference.com/w/cpp/language/definition#ODR-use)
  - [Lambda Capture](http://en.cppreference.com/w/cpp/language/lambda#Lambda_capture)
2. From the standard: 
  - §3.2 [[basic.def.odr]](http://eel.is/c++draft/basic.def.odr)
  - §5.1.5.13 [[expr.prim.lambda]](http://eel.is/c++draft/expr.prim.lambda#13)

Now that we know about ODR-use and how to covertly smuggle stuff into captureless lambdas, we will return to the issue of using them as C API callbacks in the next post. Stay tuned. 

*If you found errors in the post, found it helpful, or you have more thoughts on this subject, please leave a message in the comments. I am not an expert on this subject and am always happy to learn.*

*Acknowledgments:
[ISO C++ Standard-Future Proposals List](https://groups.google.com/a/isocpp.org/forum/#!topic/std-proposals/ER4bU9X2Uv0)* :: [banner](https://flic.kr/p/8TPsn8)





