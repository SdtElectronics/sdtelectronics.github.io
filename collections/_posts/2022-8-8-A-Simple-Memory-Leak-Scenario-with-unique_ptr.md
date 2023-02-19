---
layout: article
title: A Simple Memory Leak Scenario with unique_ptr
categories: Programming
tags: C++
eyeCatcher: https://sdtelectronics.github.io/assets/gallery/2022-8-8-A-Simple-Memory-Leak-Scenario-with-unique_ptr.png
abstract: It's well known that inappropriate use of std::shared_ptr can lead to memory leak due to circular reference. In this article, a simple case where circular reference can happen on std::unique_ptr is demonstrated, and the method to avoid it is suggested.
---

## Introduction
It's well known that inappropriate use of `std::shared_ptr` can lead to memory leak due to circular reference. This may give people an impression that `std::unique_ptr` doesn't have such pitfall. However, this impression is not true, and it can be shown that circular reference can emerge naturally in trivial use case of `std::unique_ptr`. Fortunately, it's rather easy to solve such problem, but you must be aware of it first.

## A simple example
It's quite common to have recursive data structures which own a pointer to a child with the same type of itself:
``` c++
struct Foo{
    Foo(): child(){}
    Foo(std::unique_ptr< Foo > c): child(std::move(c)){}
    std::unique_ptr< Foo > child;
    ~Foo(){
        printf("destroyed\n");
    }
};
```
We may want to make a child take the place of its parent, so that it ascends a level in the hierarchy. The simplest method is to use `std::unique_ptr::swap`:
``` c++
auto parentPtr = std::make_unique<Foo>(std::make_unique<Foo>());
parentPtr.swap(parentPtr->child);
```
And then bad things happened. When `parentPtr` goes out of the scope, only the original child got destroyed, and the resource of the parent is leaked. This occurred because after the pointer swap, the parent holds a pointer to itself. Only when the parent is destroyed, the object pointed by its `child` pointer can be destroyed. But in this case, this object is itself, so it never got destroyed. This is a typical circular reference scenario which causes memory (and possibly other resources) leak.

## The simple solution
The correct way to achieve what we want is just one line more:
``` c++
auto parentPtr = std::make_unique<Foo>(std::make_unique<Foo>());
auto tmp = std::move(parentPtr->child);
parentPtr.swap(tmp);
```
We create a temporary pointer to store the parent, and when it out of the scope, the parent got destroyed.
