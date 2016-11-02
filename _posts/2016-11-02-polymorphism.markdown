---
title:  "Polymorphism Polymorphism"
date:   2016-11-02 
categories: [C++]
tags: [C++,C++17,"std::variant"]
---
C++17 gives us `std::variant<>` which allows for a new form of runtime polymorphism.

#### Polymorphism
[Polymorphism] allows manipulating different types identically when they implement the same interface.  

C++ supports both static and dynamic polymorphism.  
Static polymorphism, via templates, is akin to duck-typing and will not be discussed in this post.  

Dynamic run-time polymorphism in C++ is achieved using sub-typing with inheritance and virtual functions.  

Let look at a simple class hierarchy:

```cpp
class Animal
{
public:
    virtual ~Animal(){}
    virtual std::string talk() const = 0;
};

class Cat: public Animal
{
public:
    std::string talk() const override { return "Meow!"; }
};

class Dog: public Animal
{
public:
    std::string talk() const override { return "Woof!"; }
};
```

We can create concrete instances of the derived classes `Cat` and `Dog` and call the virtual method `talk()` without distinguishing explicitely which is which:

```cpp
//...
Dog bud;
Cat pst;

auto speak = [](Animal const& pet)
{
    std::cout << pet.talk() << '\n';
};

speak(bud);
speak(pst);
```
Note that `bud` and `pst` are local variables declared on the stack. There is no free-store or heap allocations for these objects (though there may be for the returned strings).  

In this case we knew in advance the concrete types we needed. However, in most use cases the point of a polymorphic hierarchy is to handle runtime objects created without knowing in advance their concrete types.  
It is common to have some kind of factory that creates the concrete instance. For example:

```cpp
std::unique_ptr<Animal> get_pet()
{
    // ... determine which pet...
    return std::make_unique<Dog>(); // automatic cast to unique_ptr<Animal>
    // .. on another branch...
    return std::make_unique<Cat>(); // automatic cast to unique_ptr<Animal>
}

// use factory:
auto the_pet = get_pet();
std::array<std::unique_ptr<Animal>,2> pets = { get_pet(), get_pet() };

```
In this case the concrete instance objects of the derived class types are allocated on the *heap* whenever `std::make_unique<>` is called.  

This is the way it has always been in C++: *Polymorphism ⇒ Heap-allocated objects.*

A major advantage of this approach is that this list derived types that can be handled is open and can be extended without updating the code.

#### A New Way
C++17 gives us [`std::variant<>`][variant] which allows for a new form of stack-based runtime polymorphism.

```cpp
#include<variant>

//...

using Pets = std::variant<Dog, Cat>;
Pets get_pet2()
{
    // ... determine which pet...
    return Dog();
    // .. on another branch...
    return Cat();
}
```
Our new factory fuction `get_pet2()` returns a `variant` of `Dog` and `Cat` and does not perform any heap allocations. The resulting object is returned on the stack - *by value*.

The result type is a variant which is less conveniet to use than manipulating the base type. It is also not *an* `Animal` although the actual `variant` value is.  
We can write a helper function to convert it to a base reference:

```cpp
template <typename BaseType, typename ... Types>
BaseType& cast_to_base(std::variant<Types ...>& v)
{
    return std::visit([](BaseType& arg) -> BaseType& { return arg; }, v);
}
```
`cast_to_base<>` is a wrapper for `std::visit` which returns a reference to whichever type is actually stored inside the `variant`.  
If the variant mistakingly holds a type *not* derived from `Base`, the code will not compile - how cool is that!

Now we have all the pieces:

```cpp
auto a_pet1 = get_pet2();
auto a_pet2 = get_pet2();

Animal& pet1 = cast_to_base<Animal>(a_pet1);
Animal& pet2 = cast_to_base<Animal>(a_pet2);

speak(pet1);
speak(pet2);

std::array<Animal*,2> pets = {&pet1, &pet2};
for (auto pet: pets)
    speak(*pet);
    
// direct storage    
std::array<Pets,2> more_pets = {get_pet2(), get_pet2()};
for (auto& pet: more_pets)
    speak(cast_to_base<Animal>(pet));      
```
[Working demo](http://melpon.org/wandbox/permlink/HrAypsYnnIspUDfD).

The advantage of this appraoch is that there is no additional heap allocation (beyond what the ctors may do) - everything remains on the stack.  
The objects have value semantics so that copying is a full object copy.

The drawback is that the size of each item is as large as the largest type in the variant and the factory return type, which must be know at compile time, is closed in the sense that to add new types (e.g. `Hamster`) requires extending the variant and recompiling.

#### An Indirect Value-Type for C++
In fact, there is [a proposal][indirect] for a yet another type of runtime polymorphism:  

> Add a class template, `indirect<T>`, to the standard library to support free-store allocated objects with value-like semantics.
> The class template, `indirect`, confers value-like semantics on a free-store allocated	object. An `indirect<T>` may hold a an object of a class publicly derived
from `T`, and copying the indirect will copy the object of the derived type.



So, what do we have:

| **Where vs.<br>Semantics** | **Stack**                | **Free-Store / Heap**                            |
|---------------------|----------------------|------------------------------|
| **Reference**           |                      | Classic<br> - `std::unique_ptr<>`<br> - `std::shared_ptr<>` |
| **Value**               | `std::variant<>` | `std::indirect` |
  
or in a feature table form:

| **Polymorphism**  | **Where** | **Semantics** | **Type List** | **Size** |
|:--------------|:------|:----------|:----------|:-----------------|
| classic       | Heap  | Reference | Open      | As concrete type |
| `std::indirect` | Heap  | Value     | Open      | As concrete type |
| `std::variant<>`       | Stack | Value     | Closed    | As largest type  |

This idea originally posted on Twitter:

- [Stack-based run-time polymorphism with std::variant. Abomination?](https://twitter.com/AdiShavit/status/789053849843691520)
- [Type-safe stack-based run-time polymorphism with std::variant.](https://twitter.com/AdiShavit/status/789348353725300736)
- [The Polymorphism of Runtime Polymorphism](https://twitter.com/AdiShavit/status/790816806919401472)



*If you found this post helpful, or you have more thought on this subject, please leave a message in the comments.*


[Polymorphism]: https://en.wikipedia.org/wiki/Polymorphism_(computer_science)
[variant]: http://en.cppreference.com/w/cpp/utility/variant
[indirect]: http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2016/p0201r1.pdf		