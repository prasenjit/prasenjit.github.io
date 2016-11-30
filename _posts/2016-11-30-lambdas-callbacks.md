---
title:  "Lambda Callbacks 📞"
categories: [C++]
tags: [C++, C++11, lambdas]
---
Lambdas can be made to play nicely with C callbacks.

![img](../../assets/telephone.jpg)

Many C APIs have callback arguments in the form of function pointers.  
Consider:  

```cpp
void nifty_thing_doer(void (*callback)(int));
```

In the [first post in this series](https://adishavit.github.io/2016/magical-captureless-lambdas/) we saw that *captureless* lambdas can automatically convert to function pointers. Thus, we can easily pass `nifty_thing_doer()` a captureless lambda like so: 

```cpp
::nifty_thing_doer([](int i){ /* admire i */ });
```

However, lambdas with *non-empty* capture lists are *not* convertible to function pointers and thus cannot be inserted as such.

```cpp
  ::nifty_thing_doer([&](int i){ /* admire i */ });  // :-(
  //> error: cannot convert '<lambda(int)>' to 'void (*)(int)'
```
So what is one to do if said callback needs to capture state?  
(I am assuming here that we cannot change the API that we are using itself).

<p style="text-align: center;">📞</p>

### User Data
Fortunately, C-style callback APIs typically accept an additional callback argument of type `void* user_data`:

```cpp
void nifty_thing_doer2(void (*callback)(int, void* user_data), void* user_data)
{
   //... 	
   callback(nifty_i, user_data);	
}	
```

If we want to pass extra variables to the callback, we cannot do it via the capture list. Instead, we can do it via the opaque `user_data` pointer.  
Here's one C++11-ish way of doing it:

```cpp
void f()
{
  // local variables
  int x = 0;
  float y=1;
  
  // locally defined uncopyable, unmovable type
  struct MtEverest
  {
    MtEverest() = default;
    MtEverest(const MtEverest& that)  = delete; // no copy
    MtEverest(const MtEverest&& that) = delete; // no move
  } mt_everest;
  
  // create "user-data" payload  
  auto payload = std::tie(x,y,mt_everest);
  
  ::nifty_thing_doer2([](int i, void* userdata)
  { 
    auto& payload_tup = *reinterpret_cast<decltype(payload)*>(userdata);
    auto& xx = std::get<0>(payload_tup);
    auto& yy = std::get<1>(payload_tup);
    auto& me = std::get<2>(payload_tup);
    /* admire i */   	
  }, &payload); // <<= Pass the payload
}
```
Here, `std::tie` is used to create a `tuple` of *references* to the local variables. This also means that we can bind uncopyable, unmovable types like `MtEverest`.  

In the [second post in the series](https://adishavit.github.io/2016/leaky-closures-captureless-lambdas/), we saw that there are multiple entities that *can* be accessed from a captureless lambda, and that these are governed by the so called *ODR-use* rule.

Inside our lambda we need to cast the type-erased `user_data` to our payload type. Although we do not capture `payload` (this is a captureless lambda), we can still use it within a *non*-ODR-use such as in `decltype(payload)` to get the correct type of our tuple. From there all that is left to do is create handy aliases for the tuple elements.  

> With C++17 Structured Bindings (a.k.a. destructuring) we might be able to write something like this: 
>
```cpp
auto& payload_tup = *reinterpret_cast<decltype(payload)*>(userdata);
auto& [xx,yy,me] = payload_tup;
```
>or maybe even:  
>
>```cpp
auto& [xx,yy,me] = *reinterpret_cast<decltype(payload)*>(userdata);
```
> I have not tried this, so if not, illuminate me.

**CAVEAT:** In this example, we are passing references to local variables to the callback. This is only valid if the callback will always be called within the lifetime of the referenced variables. If the callback might be called after the termination of our function or e.g. asynchronously do not bind references. This is a general rule with references.

This cool [Bannalia blog post][Bannalia] shows another way of passing capturing lambdas to a callback by passing the capturing lambda *itself* as the `user_data` and another captureless lambda [*thunk*](https://en.wikipedia.org/wiki/Thunk) for the conversion. 

<p style="text-align: center;">📞</p>

### Statics and Globals
But what are you going to do when your callback API does *not* support `void* user_data`, and you still need to pass state variables to your callback?

In this case we are left with globals and their `static` cousins.
Any function, including captureless lambdas can access and use global variables without capturing them. The same goes for static members of class types.

We could use globals to store pointers to our internal state but that messes up the code, polluting the global namespace, increasing the chance for collisions, increases coupling and all the rest of the reasons not to use globals.  
This leaves static members.

What we'd like to do is to somehow create localized types with static members that can be set to allow us to inject state into a captureless lambda.

```cpp
namespace // anonymous namespace, everything stays in the translation unit
{
  template <typename T>
  struct payload_injector
  {
    static T val;  
  };

  template <typename T>
  T payload_injector<T>::val; // static variable definition for linker
}

int main()
{
  // local variables as before...
  
  // create payload as before 
  auto payload = std::tie(x,y,mt_everest);

  // declare/instantiate new payload injector type  
  using payload_ptr_t = payload_injector<decltype(payload)*>;  // ptr to payload type #1
  payload_ptr_t::val = &payload; // set the injector val to ref the local payload     #2

  ::nifty_thing_doer([](int i) // << captureless lambda
  {
    auto& payload_tup = *payload_ptr_t::val; // << type not even erased               #3
    auto& xx = std::get<0>(payload_tup);
    auto& yy = std::get<1>(payload_tup);
    auto& me = std::get<2>(payload_tup);

    /* admire i */      
  });
}
```
The `payload_injector<>` template has a static member of type `T`, it along with its static member `val` are defined inside an anonymous namespace so that there are no ODR-violations with other translation units for this static variable when the template is instantiated.
 
When we need to inject some state through a `static` we instantiate the template type (#1), and set the static value to our desired value (#2). In our case we are taking a pointer to the payload type.  
Essentially, `payload_ptr_t::val` is just like `user_data` above, except:

1. It does not have to be passed as a function parameter;
2. It is not type-erased

Inside our captureless lambda, we can directly retrieve the payload without even needing the `reinterpret_cast<>` (#3)!

**CAVEAT:** If `payload_injector<>` is *ever* instantiated with the same type for `T`, e.g. from different calling sites, there will be only one `payload_injector<>::val` defined. This might lead to collisions in certain situations. More on why this isn't necessarily an issue in a moment.  

### Coming Full Circle 
Now that we have the `payload_injector<>` facility, we can do even better!  
Why go to all the trouble with creating `payload` with `std::tie` at all and pass that around?

```cpp 
  auto my_callback = [&](int i) // the capturing callback we wanted all along
  {
    auto z = x+y;
    // use x, y, and mt_everest as desired
    // continue to admire i  ...
  };
  
  // declare/instantiate new payload injector type  
  using payload_ptr_t = payload_injector<decltype(my_callback)*>; // ptr to my_callback
  payload_ptr_t::val = &my_callback;

  ::nifty_thing_doer([](int i) 
  { // trivial thunk just calls lambda
    (*payload_ptr_t::val)(i); 
  });
```    
Now isn't that cool?  

Instead of injecting the variable bundle into the captureless lambda, something that requires updating as the lambda changes, we can just create a capturing lambda and inject that lambda object *itself* into a simple captureless "thunk" lambda that simply calls it!

This approach is also superior to the above one since lambdas *always* have unique types, so there can *never* be collisions for the static variable `payload_injector<>::val` as each instantiated template has its own type. 

### Loose Ends
Even if we prefer to use the first approach and inject a tuple, we could extend `payload_injector` with a tag type template parameter or a non-type template parameter and pass that the current line or some other local unique tag to avoid collisions.

However, even that does not necessarily solve the collision problem since multithreaded code might have the same lambda running asynchronously on multiple threads. All these runs will try to modify and use the same static variable and may cause severe havoc. This is the usual caveat with using `static` variable and members in multithreaded code.  

But, C++11 gives us [`thread_local`](http://en.cppreference.com/w/cpp/language/storage_duration) so that each thread will actually use its own copy of the static variable:  

```cpp
namespace // annonymous namespace
{
  template <typename T>
  struct payload_injector
  {
    thread_local static T val;  
  };

  template <typename T>
  thread_local T payload_injector<T>::val;
}
```

To summarize, how do we create per-lambda "static" state?

1. Use anonymous namespace to prevent link-level ODR-violations across translation units  
   Each `.cpp` will get its own copy of the `static` variable.
2. Use `thread_local` instead of `static` so that each *thread* gets its own copy of the variable.
3. By passing the lambdas themselves, each lambda gets its own template type instantiation and thus its own variable.

As usual, comments are most welcomed!

*Acknowledgments:
[banner](https://www.pexels.com/photo/black-rotary-telephone-at-top-of-gray-surface-163008/) :: 
[Bannalia][Bannalia]*

 
[Bannalia]: http://bannalia.blogspot.com/2016/07/passing-capturing-c-lambda-functions-as.html