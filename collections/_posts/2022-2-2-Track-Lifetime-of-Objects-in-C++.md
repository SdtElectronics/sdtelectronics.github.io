---
layout: article
title: Track Lifetime of Objects in C++
categories: Programming
tags: C++ design-pattern
eyeCatcher: https://sdtelectronics.github.io/assets/gallery/2022-2-2-Track-Lifetime-of-Objects-in-C++.png
---

## Motivation
Managing the lifetime of objects is one of the biggest hassles in C++, and it became even more complicated after the introduction of move semantics in C++11. Occasionally we found ourselves being puzzled by some of the following questions:
* When the object was copied? (Did we forget to pass by reference or the compiler failed to elide copies?)
* When the object was moved? (Did move constructor/assignment operator worked?)
* When the object was destructed? (Is the reference still valid at some point?)

For class defined by ourselves, an invasive way to inspect these is to modify the copy/move constructor and destructor. However, for external dependencies, the invasive approach can be tough or impossible to adopt. A non-invasive alternative is preferred to amend the behavior of ctors and dtors for arbitrary classes. This article attempts to devise such a method.

## A Generic Wrapper Implemented as Static Decorator
When we need to add behavior to existed classes, the decorator pattern comes to our mind. One of the powerful features in C++ is that the inherited class can be templated, which allows us to implement static decorators. For example, if we want to write a decorator which make an object print a message before destruction, it can be implement as below:
```C++
template <typename T>
struct Decorator: public T{
    ~Decorator(){
        std::cout << "destructed" << std::endl;
    }
};
```
Now when a `Decorator<T>` goes out of scope, "destructed" is printed. However, this doesn't help much, as there is no way to initialize `T` inherited by `Decorator<T>`. We can write a constructor accepting an instance of `T`, or even better, we let `Decorator<T>` expose same constructors as `T`, which makes `Decorator<T>` become a drop-in replacement of `T`. In C++11, we can achieve this with a single line, thanks to the feature called "Inheriting constructor":
```C++
template <typename T>
struct Decorator: public T{
    using T::T; // inheriting constructors declaration

    ~Decorator(){
        std::cout << "destructed" << std::endl;
    }
};
```
However, we shall be aware of that the copy/move constructor are not inherited in this way, thus we have to write copy/move constructor for it manually:
> For each non-template constructor in the candidate set of inherited constructors **other than a constructor having no parameters or a copy/move constructor having a single parameter**, a constructor is implicitly declared with the same constructor characteristics unless there is a user-declared constructor with the same signature in the complete class where the using-declaration appears or the constructor would be a default, copy, or move constructor for that class. [class.inhctor]/p3

## Designating Behaviors Added by the Decorator
We have implemented a decorator to track the destruction of objects, in which the behavior added is rather trivial: a hard-coded message is printed. We'd like to make the added behavior customizable, however, the approaches are pretty limited. We surely don't want to mess up with the constructor, which may interfere with the constructor of underlying type `T` (Plus do this for each time the decorator is used could be quite tedious). Thus, the behavior has to be bound with the type of the decorator. One way is to use virtual method and inheritance to override the behavior defined in the parent decorator, but we have to declare inheriting constructors in each derived decorator. This also breaks the encapsulation, which is not quite elegant. Another way is making the decorator inherit behaviors from user-defined classes (I will use `class M` to refer to it henceforth), while `Decorator<T>` has already inherited `T`, which incurs notorious multiple inheritance. However, this is merely a debugging utility that won't make it way to production code and it's unlikely that the hierarchy will evolve much further, so who cares I guess. If you have better idea to accomplish this, please do share it with me! The segregated structure would be like this:
```C++
struct Simple{
    void destructed(){
        std::cout << "destructed" << std::endl;
    }
};

template <typename T, typename M = Simple>
struct Decorator: public T, public M{
    using T::T; // inheriting constructors declaration

    Decorator(const Decorator& rhs): T(rhs), M(rhs){}

    Decorator(Decorator&& rhs): T(std::move(rhs)), M(std::move(rhs)){}

    Decorator(const T& rhs): T(rhs){}

    Decorator(T&& rhs): T(std::move(rhs)){}

    ~Decorator(){
        M::destructed();
    }
};
```
## Extending the Ability of Decorator: Type Inspection and Object Identification
So far we have made a decorator adding customizable behaviors to class `T`, which can directly replace `T` itself. Not bad! Now let's see how this ability can be exploited to make the inspection more helpful. When the lifetime of multiple types of objects are tracked, we may wonder what type of object is destructed. We can embed the type to messages by writing `class M` for each type, but RTTI (another controversial feature, but again we only use it for debugging purpose) can do this for us automatically. The only problem is how to pass type to `class M`. Make `class M` itself a template would free us from making each member function requiring type info a template, but `M` is already a template parameter for `Decorator<T,M>`, thus we need a *template template (no duplication here) parameter* whose syntax is not very pleasant:
```C++
template <typename T, template <typename U = T> typename M = Simple>
struct Decorator: public T, public M<>{
    using T::T; // inheriting constructors declaration

    Decorator(const Decorator& rhs): T(rhs), M<>(rhs){}

    Decorator(Decorator&& rhs): T(std::move(rhs)), M<>(std::move(rhs)){}

    Decorator(const T& rhs): T(rhs){}

    Decorator(T&& rhs): T(std::move(rhs)){}

    ~Decorator(){
        M<>::destructed();
    }
};
```
And `class M` can be implemented with `typeid()` operator and demangling facility provided by the compiler(`__cxa_demangle()` for g++ and clang++ is used here)
```C++
template <typename T>
struct Stamped{
    char* type(){
        char* name = 0;
        int status;
        name = abi::__cxa_demangle(typeid(T).name(), 0, 0, &status);
        return name;
    }

    void destructed(){
        char* name = type();
        std::cout << name <<  " is destructed"  << std::endl;
        free(name);
    }
};
```
We may also want to track the chain that an object is copied, moved and destructed. The address of the object can be used to identify it, where another peculiarity of multiple inheritance arises: `this` in the base and derived class may not have the same value. Methods in `class M` have no way to know where `T` is copied or moved, unless `Decorator<T,M>` tells it by argument. In `class M`, `this` has the type `M*`, while `&rhs` in the following snippet has the type `T*` or `Decorator<T,M>*`, so they can differ even when they are pointing to the same object, which could be confusing. We can pass `T::this` with an additional argument to methods in `class M`, nevertheless being a bit redundant, but this argument comes handy when combining several decorator into a new one.
```C++
template <typename T, template <typename U = T> typename M = Simple>
struct Decorator: public T, public M<>{
    using T::T; // inheriting constructors declaration

    Decorator(const Decorator& rhs): T(rhs), M<>(rhs){
        // Multiple inheritance cause M::this, T::this and Decorator::this may differ
        // Force to T::this to consolidate
        M<>::copied(&rhs, &static_cast<const T&>(*this));
    }

    Decorator(Decorator&& rhs): T(std::move(rhs)), M<>(std::move(rhs)){
        M<>::moved(&rhs, &static_cast<const T&>(*this));
    }

    Decorator(const T& rhs): T(rhs){
        M<>::copied(&rhs, &static_cast<const T&>(*this));
    }

    Decorator(T&& rhs): T(std::move(rhs)){
        M<>::moved(&rhs, &static_cast<const T&>(*this));
    }

    ~Decorator(){
        M<>::destructed(&static_cast<const T&>(*this));
    }
};
```
The `class M` to pretty print the addresses and type:
```C++
template <typename T>
struct Stamped{
    char* type(){
        char* name = 0;
        int status;
        name = abi::__cxa_demangle(typeid(T).name(), 0, 0, &status);
        return name;
    }

    void copied(const T* lhs, const T* rhs){
        char* name = type();
        std::cout << name << ", at " << rhs << ", is copied to " << lhs << std::endl;
        free(name);
    }

    void moved(const T* lhs, const T* rhs){
        char* name = type();
        std::cout << name << ", at " << rhs << ", is moved to " << lhs << std::endl;
        free(name);
    }

    void destructed(const T* lhs){
        char* name = type();
        std::cout << name << ", at " << lhs << " is destructed" << std::endl;
        free(name);
    }
};
```
# Epilogue: Introducing Logoo, The Log of Object Inspector
We have gone pretty far for a wrapper accomplishing the simple task of tracking the lifetime of an object, but there are surely more features to be added to make the inspection more helpful e.g. a stack trace with source snippet indicating where the copy/move/destruction occurred, a fingerprint marking all objects copied/moved from the same ancestor, etc. The implementation of these would be trivial though, and I found nothing of them worth being discussed here. I have put all of these together to form a tiny header-only toolkit, and you can find the source and examples here: [ACI/logoo/](https://github.com/SdtElectronics/ACI/tree/master/Logoo), where "logoo" stands for "**log** **o**f **o**bject".

Except illustrating the idea of the inspector itself, I was also intended to make this article a showcase that how two features of C++, namely templated inheritance and inheriting constructors, can be combined to yield a powerful technique, the static decorator. The subtlety of such decorators is that, they can seamlessly replace the original class, which allows pretty convenient and broad adoption, and such convenience preserves even when multiple decorators are composed. I have also proposed a method to customize the behavior added by the decorator with multiple inheritance, which may not be optimal. I am looking for a better solution to make this pattern more flexible and elegant.