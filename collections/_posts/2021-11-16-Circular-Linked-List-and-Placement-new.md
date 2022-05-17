---
layout: article
title: Circular Linked List and Placement new
categories: Programming
tags: C++ data-structure library
eyeCatcher: https://sdtelectronics.github.io/assets/gallery/2021-11-16-head-Circular-Linked-List-and-Placement-new.jpg
abstract: Placement new is a rarely used feature in C++. This article gives a brief introduction to this feature, and shows how it can be helpful in the implementation of circular linked lists.
---

A circular linked list is a linked list in which the last node points to the first node. It comes handy when dealing with data with a ring structure, but the focus of this article is not the application of circular linked lists. The main problem to be addressed is how the first node can be inserted in the same way as other nodes, and this is explained below:

## `push_back` and The Issue of Empty Lists
Appending an element is one of the most common operations to a container, which is called `push_back` in C++ convention. The implementation of `push_back` for circular linked list should be quite straightforward: We construct a new node, then link the last node in the list to it, and link it to the first node. It becomes interesting when the list is empty - There is no first or last node to link to. This is not a big issue though, as there are 2 simple way to solve it:
- Add a conditional branch to take care of the empty list. If the first node in the list is NULL, we link the new node to itself.
- Forbid construction of empty list. Every list should have at least one node.


The first solution seems quite simple and effective, but that additional branch adds a little overhead to each append operation. The second solution eliminates this overhead by reserving a node in the list, but we have to construct an instance of the underlying data type during the initialization of the list, which is troublesome or even impossible in some conditions. 

## Introducing Placement `new`
Take a look at our second approach again, and it can be figured out that what we need is reserving the space for the node but not initializing it. The crux is that the allocation and initialization of an object are typically done with a single operation in C++, i.e. `new`. We do know how to allocate a region of raw memory with `malloc` in C, but how about the initialization of an object at the allocated region? That is exactly what the placement `new` operator does. 

The syntax of placement `new` is:
``` c++
Foo* p = new(raw) Foo();
```
Where `Foo` is the underlying type and `raw` is the pointer to the allocated memory. For more about placement new, you can refer to [this page on isocpp.org](https://isocpp.org/wiki/faq/dtors#memory-pools).

Now we can allocate space for a node at the initialization of the new list and initialize it when a new element is appended. A demonstrative code fragment of implementation would be like this:

``` c++
template<typename T>
class ccNode{
  public:
    ccNode(const T& data, ccNode<T>* next);

  private:
    template<typename T>
    friend class ccList;
    // pointer to the next node
    ccNode<T>* _next;
    T _data;
};

template<typename T>
class ccList{
  private:
    // pointer to the head node
    ccNode<T>* head;
    // pointer to the tail node
    ccNode<T>* tail;
    // pointer to the memory reserved for the next appended node
    ccNode<T>* next;

  public:
    ccList(): 
        head(allocateMemory<T>()), 
        tail(head), 
        next(head){
    }

    void push_back(const T& data){
        tail->_next = new(next) ccNode<T>(data, head);
        tail = next;
        next = allocateMemory<T>();
    }

    // ... destructor, functions for other operations, etc. 
}
```

## Allocation and Destruction
You may have noted the `allocateMemory<T>()` function in code above. I didn't use `malloc` but wrote this non-existent function on purpose. In C++, we have the C++ way for memory allocation, that is the `allocator<>`. More details about allocators are beyond the scope of this article and I will cover the usage of standard allocator only. After the initialization of an allocator for the designated type `T`, a region of memory for `n` instances of `T` can be allocated by its member function `allocate`:
``` c++
std::allocator<T> alloc;
T* p = alloc.allocate(n);
```
The memory allocated by `allocator<>` can only be freed with the `deallocate` function:
``` c++
alloc.deallocate(p, 1);
```
But this won't destroy the object in the memory. The destructor of that object must be called explicitly in advance to avoid resource leakage:
``` c++
p->~T();
```
Destructors are typically called automatically, thus this may seems quite weird, but this operation is valid, and necessary here.

## The Complete Implementation
Things become complex quickly when considering actual implementation. In addition to more member function to support more operations, some features should be added to meet the convention of containers in C++. One important facility is the iterator, and template arguments should allow passing in user-defined allocators. The complete code would be too lengthy to be shown here, and a minimal working example by me can be find at [this repository](https://github.com/SdtElectronics/libSCALE/blob/master/src/container/circList.h).

## References
- [isocpp.org/wiki/faq/dtors#memory-pools](https://isocpp.org/wiki/faq/dtors#memory-pools)
- [isocpp.org/wiki/faq/dtors#placement-new](https://isocpp.org/wiki/faq/dtors#placement-new)
