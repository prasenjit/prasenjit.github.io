---
title: "Default Constructed Return Value: return {}"
date: 2015-02-08 10:17
categories: [C++, C++11]
tags: [C++, C++11]
---
It is common for C/C++ functions to return default values, for example, if some internal condition fails.  
This is straightforward for native return types, as in:

```cpp
int  f1() { return 0;       }
int* f2() { return nullptr; }
```

However, if the return type is more complex, you would often need to default construct the type for returning it (I am assuming here that the return type does have a default ctor):

```cpp
std::shared_ptr<MyCustomType> f3() { return std::shared_ptr<MyCustomType>(); }
std::vector<MyCustomType>     f4() { return std::vector<MyCustomType>();     }
```

This pattern is often seen in *factory* methods or functions that return a polymorphic type via a base handle. Also with move semantics, returning containers like `std::vector<>` is a common case.  
If the function has multiple early return points, this default construction needs to be repeated at each such point and each time stating the explicit return type just for calling the default ctor. This is annoying because the compiler already knows the return type.

In the case of `f3()` above, `std::shared_ptr<>` can be constructed from `nullptr` so it can alternatively be written as:

```cpp
std::shared_ptr<MyCustomType> f3() { return nullptr; }
```

But that's just a convenient solution for `std::shared_ptr<>`. What's to be done for other types?

Help is at hand from C++11's us *list initialization*.  
List initialization allows us to [default] construct the return-type object upon returning (see case (8) [here](http://en.cppreference.com/w/cpp/language/list_initialization)).  
Thus, we can call the default ctor without explicitly specifying the types in all the cases above with the simple `return {}` statement:

```cpp
int                           f1() { return {}; }
int*                          f2() { return {}; }
std::shared_ptr<MyCustomType> f3() { return {}; }
std::vector<MyCustomType>     f4() { return {}; }
```

This will work with any (non-explicit) default constructible type.
