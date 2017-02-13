---
title:  "Custom Stream Buffers"
categories: [C++, C++11, C++17, streams]
tags: [C++, C++11, C++17, streams, iostream]
---
A stream buffer is the vegetated land adjacent to a streambank.

![img](../../assets/stream2.jpg)

> Stream buffers are strips of trees and other vegetation that
improve water quality by filtering pollutants from stormwater runoff such as oil, fertilizers, pesticides; reduce flooding and erosion by stabilizing stream banks; moderate stream temperature and sunlight, keeping fish and other aquatic life healthy; provide nesting and foraging habitat for many species of birds and animals.

Yeah, OK - not [those stream buffers](https://en.wikipedia.org/wiki/Riparian_buffer). I'd like to talk a bit about the beloved, intuitive, C++ standard library kind, the ones everybody loves (to hate) and understands (not).    

When I was a wee C++ apprentice (pre-C++98), I was taught that when designing your API don't take a file-name or assume a standard input/output stream. Instead, take a `std::istream&` or `std::ostream&` (or `std::iostream`) and thus support *any* such (derived) stream.  

Recently, while working on integrating two libraries, I came across an interesting challenge.   

One library, `Encoder`, *the encoding library*, expected an `std::ostream&` to write data too (it thus naturally supported file serialization given a `std::ofstream`): `Encoder::encode(std::ostream& os);` 
  
The other library, call it *the device library*, provided a C-style write function such as `int device_write_callback(const char* buf, int sz)`.  

I needed to get the encoding library to encode my data and write the encoded data into the device using this API. The easiest solution is of course:

```cppEncoder encoder; 
// fill encoder with data...std::ostringstream ostr;encoder.encode(ostr);
auto buf_str = ostr.str(); // makes another COPY!device_write_callback(buf_str.c_str(), buf_str.size());
``` 
    
This works, but has a major drawback: the encoder will first encode all the data into an internal *in-memory* buffer. Only after this is allocated and done, will the entire thing be written to the device. If our encoded data is very large, this can consume copious amounts of memory.  

To make matters worse, `ostr.str()` returns an ***additional copy*** of that giant buffer!

What we'd like is to create a `std::ostream` that instead of writing to a memory buffer, directly calls the device-writer function.  

One option is to use [Boost.Iostreams](http://www.boost.org/doc/libs/1_63_0/libs/iostreams/doc/index.html). If you already have Boost in your project that is a valid way to go, but if you don't, or don't want the extra power and complexity that it brings, it turns out we can get the same benefit by rolling our own little `streambuf`.


## Custom Stream Buffers

The standard provides two ready made stream buffers [`std::basic_filebuf`](http://en.cppreference.com/w/cpp/io/basic_filebuf) and [`std::basic_stringbuf`](http://en.cppreference.com/w/cpp/io/basic_stringbuf) which are the basis of [`std::basic_fstream`](http://en.cppreference.com/w/cpp/io/basic_fstream) and [`std:: basic_stringstream`](http://en.cppreference.com/w/cpp/io/basic_stringstream) respectively.  

We will create our own class derived from [`std::streambuf`](http://en.cppreference.com/w/cpp/io/basic_streambuf) and pass that to a generic [`std::ostream`](http://en.cppreference.com/w/cpp/io/basic_ostream). 

A thorough review of iostreams and stream-buffers is *way* beyond the scope of this post (or my knowledge for that matter). So here's how to create a simple little `streambuf` to avoid the extraneous memory allocations (at least on your side) and just call the callback with whatever data is ready to be written:  

```cpp
template <typename Callback>struct callback_ostreambuf : public std::streambuf{    using callback_t = Callback;    callback_ostreambuf(Callback cb, void* user_data = nullptr): 
       callback_(cb), user_data_(user_data) {}protected:    std::streamsize xsputn(const char_type* s, std::streamsize n) override     {         return callback_(s, n, user_data_); // returns the number of characters successfully written.    };    int_type overflow(int_type ch) override     {         return callback_(&ch, 1, user_data_); // returns the number of characters successfully written.    }private:    Callback callback_;    void* user_data_;};
```
Basically, we derive our class from `std::streambuf` and override the two virtual functions `xsputn()` and `overflow()`. The default (base) implementations of the rest of the streambuf methods will eventually reach these two functions: one writes a single character and the other several (in fact, the *default* implementation of `std::streambuf::xsputn()` calls `std::streambuf::overflow()` `n`-times). 

A few things to note:

1. The class is templated on a `Callback` type. Although we know the desired signature of the write callback (since we have to call it in the methods and they must return the number of characters successfully written), we use a template type parameter as the callback type so we can also pass capturing lambdas as our callbacks.
1. In the code above, I made the write function be a callback that also accepts a `void* user_data` argument. This is useful with C-style APIs but not always necessary (and not really relevant for the discussion here).

We can also add a little helper `make` function:

```cpp
template <typename Callback>auto make_callback_ostreambuf(Callback cb, void* user_data = nullptr){    return callback_ostreambuf<Callback>(cb, user_data);}
```

We can now use our class like this:

```cpp
auto cbsbuf = make_callback_ostreambuf([](const void* buf, std::streamsize sz, void* user_data){    std::cout.write(reinterpret_cast<const char*>(buf), sz);    return sz; // return the numbers of characters written.});std::ostream ostr(&cbsbuf);ostr << "TEST " << 42; // Write string and integer
```
Although somewhat contrived, this will print `TEST 42` to the console. In fact in such a lambda we can call any other device writing API regardless of the actual signature of that API.

### Sweet 17 and beyond
The code above should work with C++11 and C++14 (up to some minor fixes). In fact, without the lambdas it'll should also work with pre-C++11 compilers (the iostream library is pre-C++98!).   
However, there are a few issues with the code above that make it less modern and type safe than it could be.

For starters, `make_callback_ostreambuf()` is an ugly API wart. The only reason we need it is so that we don't need to write `callback_ostreambuf<decltype(callback_fun)> my_streambuf(callback_fun);` where we must specify the type to the template class (i.e. `<decltype(callback_fun)>`).

C++17 will bring us [*class template deduction*](http://en.cppreference.com/w/cpp/language/class_template_deduction) so with a C++17 conforming compiler we will be able to declare `callback_ostreambuf my_streambuf(callback_fun);` and the callback type will be deduced automatically just like with `make_callback_ostreambuf()`.

A more subtle annoyance is that there is actually no static type enforcement on the callback type. It is called `Callable` but it is only *assumed* that:

1. it is a callable;
2. it has the correct arity (number of args);
3. it has the correct return type;
4. all the args have the correct types.

Duck typing galore!  
Any deviation from these will be rewarded with a compilation error or various wanrnings.   However, to make matters worse, these errors and warnings will be shown at the point of duck-type usage, i.e. *inside* the method implementations of our class and not at the point of call at the class ctor.

Ironically, had we made our class non-template and decided to support only function-pointer callbacks, the compiler would have warned us of a type mismatch at the ctor call. 

Using [SFINAE](http://en.cppreference.com/w/cpp/language/sfinae) it is, in fact, possible to decompose the `Callable` type into various types of callables (e.g. function pointers, class member `operator()`, etc.) and `static_assert` correct conversions. While demonstrating some very clever template meta programming (TMP) techniques is appealing, the resulting code would be several times larger and many more times more obscure than our current 20 line class.  

There is however a glimmer of hope. Part of the *raison-d'être* of C++ concepts, which may be available as soon as C++20, is to allow more powerful and expressive static type checking and allow the compiler to warn at the point of usage. Concepts would allow us to precisely describe the callback type for proper type checking. I quiver in anticipation.


> **TL;DR**   
> To create a custom `streambuf`:
> 
> 1. Derive your class from `std::streambuf`;
> 2. Override the two virtual functions `xsputn()` and `overflow()`.  
>    (You can even get away with overriding only `overflow()` and the default `xsputn()` will call it `n` times).
> 
> (see what I did? put the TL;DR at the end... mwahahaha)

<p style="text-align: center;">🦆</p>


## Custom Stream Buffer View

So we have a buffer with our encoded data. Maybe we saved it, maybe we sent it over the network. At some point we'd like to decode from such a buffer. We have a large contiguous memory buffer and we'd like to read from it. Our decoding library, again, sports a `std::istream` interface for decoding. How can we provide it with such a stream? 

As before, the obvious option is to create a `std::istringstream` initialized with our buffer. However, inspecting the docs for [`std::istringstream`](http://en.cppreference.com/w/cpp/io/basic_istringstream) shows that both the [ctor](http://en.cppreference.com/w/cpp/io/basic_istringstream/basic_istringstream) and the [`std::istringstream::str(<string>)`](http://en.cppreference.com/w/cpp/io/basic_istringstream/basic_istringstream) methods create *copies* of the data, something we'd like to avoid for very large buffers.

What we want is an "`istring_viewstream`" that will stream a *view* of our buffer just like C++17 [`std::string_view`](http://en.cppreference.com/w/cpp/string/basic_string_view) is to `std::string`.

Fortunately, that's pretty short work: 

```cpp
template <typename Byte = char>class istreambuf_view : public std::streambuf{public:    using byte = Byte;    static_assert(1 == sizeof(byte), "sizeof buffer element type 1.");    istreambuf_view(const byte* data, size_t len) :     // ptr + size        begin_(data), end_(data + len), current_(data)    {}    istreambuf_view(const byte* beg, const byte* end) : // begin + end        begin_(beg), end_(end), current_(beg)    {}protected:    int_type underflow() override    {        return (current_ == end_ ? traits_type::eof() : traits_type::to_int_type(*current_));    }    int_type uflow() override    {        return (current_ == end_ ? traits_type::eof() : traits_type::to_int_type(*current_++));    }    int_type pbackfail(int_type ch) override    {        if (current_ == begin_ || (ch != traits_type::eof() && ch != current_[-1]))            return traits_type::eof();        return traits_type::to_int_type(*--current_);    }    std::streamsize showmanyc() override    {        return end_ - current_;    }    const byte* const begin_;    const byte* const end_;    const byte* current_;};
```
Again, we derive our class `istreambuf_view` from `std::streambuf` and this time override a different set of virtual functions. All this class really does is manage 3 pointers. 

The usage is quite straight forward:

```cpp
auto buffer = "TEST 42"s;auto view_buf = istreambuf_view<>(buffer.data(), buffer.size());std::istream istr(&view_buf);std::string str;int v = 0;istr >> str >> v; // Read string and then integerassert("TEST" == str && 42 == v);```

A few things to note:

1. The class is templated on a `Byte` type. Any 1-byte type should work and this is `static_assert`ed.
2. The default character type is `char`. Using that we get the aptly named *"diamond operator"* `<>`.
2. For convenience I added two constructors, one for pointer+size and the other for a begin+end.

## Sweet 17
In this case too, C++17 will come to our aid and slightly simplify our code. With either [*class template deduction* or *user-defined deduction guides*](http://en.cppreference.com/w/cpp/language/class_template_deduction) we will be able to drop the diamond operator in the default case and remain with the cleaner `auto view_buf = istreambuf_view(buffer.data(), buffer.size());` 

A natural extension to this class would be to also accept a [`std::string_view`](http://en.cppreference.com/w/cpp/string/basic_string_view) or GSL's various span flavors. 

<p style="text-align: center;">💎</p>

## Summary
Iostreams are much maligned, but they go back a long way and for better or worse they shall remain with us for a while yet. Many idioms build upon them and occasionally one needs to dip a toe into these frigid waters. This post is not intended as a comprehensive tutorial but mostly as a public pasteboard where I can come and remember how to do these things in a few years instead of rummaging and collecting this info again elsewhere. I hope you find it useful too. 


Acknowledgments:  

- I'd like to thank [Simon Brand](https://twitter.com/tartanllama), [Jonathan Müller](https://twitter.com/foonathan), [Anthony Williams](https://twitter.com/a_williams) and [Lynden Shields](https://twitter.com/LyndenShields) for the great discussion about SFINAE on lambdas on the lovely [C++ Slack channel](https://cpplang.slack.com/). 
- Miro Knejp taught me the term *diamond operator* and cool uses for *user-defined template constructor deduction guides*.
- [This SO answer](http://stackoverflow.com/a/13704122/135862) and [this post](https://artofcode.wordpress.com/2010/12/12/deriving-from-stdstreambuf/) for pointing me in the right directions.

*If you found this post helpful, or you have more thoughts on this subject, please leave a message in the comments, Twitter or Reddit. You can also follow me on [Twitter](https://twitter.com/adishavit).*

*Credit:
[banner](https://www.pexels.com/photo/nature-water-rocks-stream-128184/)*

