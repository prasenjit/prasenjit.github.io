---
title:  "Istream Idiosyncrasies"
categories: [C++, C++11, streams]
tags: [C++, C++11, streams]
---
Idiosyncrasies with istreams.

![img](../../assets/stream.jpg)

A lot has been written and the vice and virtues of the C++ iostreams library.  
While working on [Argh], my little C++11 argument parsing library, I ran across some somewhat surprising idiosyncrasies and a few interesting lessons. 

Argh uses `std::istringstream`s to allow the user to convert or lexical-cast command line argument strings into the desired target types (thus supporting all primitive types and any other types with `operator >>`). In doing so, some of its `std::istringstream` uses are less-than typical. I will not delve too deeply about Argh and its philosophy here. You are welcome to read more about it [here][Argh]. 

### C++11 Breaking Changes
It is well known how much effort the standardization committee puts into keeping C++ backward compatible and to avoid breaking changes. So I was a bit surprised at what happens when extraction fails i.e., streaming an incompatible string into a type, e.g. `hello` into an `int`.  

In pre-C++11 the streamed to value is *left unmodified*.  
However, [since C++11](http://en.cppreference.com/w/cpp/io/basic_istream/operator_gtgt) if extraction fails, ***zero** is written to value*. 

**TL;DR** If you need the original value in case of failure, store it on the side.

### Bad Streams
To indicate that an argument is not present, `std::istringstream` can also be used as an "optional" (or Maybe). A bad stream supports [`bool operator!()`](http://en.cppreference.com/w/cpp/io/basic_ios/operator!) for checking validity. So if an argument does not exist, the stream will be bad and should not be streamed from (or if you do, the streaming will fail).

Unfortunately, there is no way to construct a bad stream, so it is impossible to do this:

```cpp
std::istringstream parser::operator()(std::string const& name)
{
  if (name.empty())
     return std::istringstream( ... set failbit ...); // ERROR: No such ctor
  // ...
}
```

The only way to create a bad stream is to create a variable and set it to bad:

```cpp
std::istringstream parser::bad_stream() const
{
   std::istringstream bad; 
   bad.setstate(std::ios_base::failbit);
   return bad; 
}
``` 
This is both verbose and prevents the [mandated](http://en.cppreference.com/w/cpp/language/copy_elision) RVO and instead relies on NRVO which isn't.

**TL;DR** It would be nice to have a constructor that can construct a bad stream.

### Move Semantics
Note that the return type of both `parser::operator()` and `bad_stream()` above is `std::istringstream`. However, `std::istringstream` does not have a copy constructor, it only has a move constructor.

When returning an object of type `std::istringstream` it is in fact either moved or copy-elision will take place. The fact that copy elision can happen here also means that `return std::move(istr);` is a pessimization since it will force a call to the move ctor even when complete elision could have taken place.

**TL;DR** Beware of `return std::move(xxx);` it is very often (almost always?) a pessimization.

Interestingly, when a GitHub user wanted Argh to support GCC 4.9, it [turned out](http://stackoverflow.com/questions/40110466/workaround-for-returning-uncopyable-object-without-a-move-ctor) that [the move ctor is missing](https://gcc.gnu.org/bugzilla/show_bug.cgi?id=54316) which caused the code to not build.

The workaround, [contributed](https://github.com/girishnayak12/argh/pull/6) by the diligent [@ChaosCabbage](https://github.com/ChaosCabbage) was to create a proxy class around `std::istringstream` which *does* have a copy ctor:

```cpp
stringstream_proxy(const stringstream_proxy& other) :
   stream_(other.stream_.str())                // copy string
{
   stream_.setstate(other.stream_.rdstate());  // copy state
}
```  
This workaround only kicks in for GCC < 5.   
Conveniently, this actually works *because* we did not explicitly specify that we want to *move* the stream, but we allow to compiler to either: elide, move or copy the stream. On gcc <5 it will choose the copy-ctor of our proxy, on more conformant library implementations and compilers it will move the stream or completely elide the call.

### Summary
Beware of little stream surprises, don't let them bug you.  

<p style="text-align: center;"><img src="../../assets/archer-opt.gif" width="440px"/></p>

Learn to embrace the little surprises in life. Keep notes of what you did and maybe blog about it.

If you want to learn more about Argh, find it [here][Argh]. If you find it useful, do drop me a note - I assure you, it'll make my day. Pull requests will be gladly accepted.

*If you found this post helpful, or you have more thoughts on this subject, please leave a message in the comments, Twitter or Reddit. You can also follow me on [Twitter](https://twitter.com/girishnayak12).*


*Credit:
[banner](https://www.pexels.com/photo/time-lapse-photography-of-falls-surrounded-by-trees-211471/) :: [giphy](http://gph.is/295idIW)*



[Argh]: https://github.com/girishnayak12/argh