---
layout: post
title: "Smart-pointers: std::shared_ptr"
---
In this second post about smart pointers we are going to talk about **```std::shared_ptr```** .

As in the last post about smart pointers, we are going to quote Scott Meyers here,
on his *Effective Modern C++*, Item 19:

> Use std::shared_ptr for shared-ownership resource management.

So it expresses shared ownership, and in this context that means ***reference counting***.

An object accessed via **```std::shared_ptr```** has its lifetime managed by these, and no specific
**```std::shared_ptr```** owns the object.
When the last **```std::shared_ptr```** pointing to an object stops pointing to it, it will destroy the object
it points to.

How can a **```std::shared_ptr```** know whether it is the last one pointing to a resource?

By consulting the resource's ***reference-count***.

In fact there's a lot more going on in terms of memory allocation when creating a  **```std::shared_ptr```** than a 
**```std::unique_ptr```**.

In fact a **```std::shared_ptr```** contain a ptr to the object(resource) and also a pointer to something like 
a control-block-resource, block which contains the reference-count, but also other information like: weak-reference count (we'll talk
about this in the next post about smart_pointers), possibly a custom deleter, and also a pointer to the controlled
object(mostly due to the fact that we can actually have a pointer to a subclass of an object we actually pointing to).

Also due to the fact that the deletion block is also in this control-block, the **```std::shared_ptr```** plays effectively
a role in the ownership of this block, but this block actually controls the deletion(quite different from **```std::unique_ptr```**).

All of this can of course impact performance, mostly because(as Scott Meyers pointed out in *Effective Modern C++*, Item 19):

- **```std::shared_ptr```** is twice the size of a raw ptr. Due to the fact that they actually hold a ptr
to the resource object but also a ptr to the control block.

- Memory for the reference count must be dynamically allocated. Using **```std::make_shared```** can 
reduce this allocation to just one allocation. It can just **```malloc```** a single chunk of memory
to contain the resource obj and the control-block.

- Obviously the reference-count increments and decrements must be atomic, because there can be readers
and writers in different threads.
So in one thread we can have one **```std::shared_ptr```** calling its destructor -> ***decrementing the reference count***,
while on another thread we can have in a different thread a **```std::shared_ptr```** pointing to the same obj
being copied -> ***incrementing the reference count***.
These kind of atomic operations are typically slower than non-atomics ones.

- Move-constructing and move assignment actually stop the need for the manipulation of the reference-count,
so these operations are faster.

It's good to keep in mind:

- An object's control block is set up by the first creation of a **```std::shared_ptr```** pointing to that object.

There are also some rules that apply for the control-block creation:

- **```std::make_shared```** always creates a control block.

- A control block is created when a **```std::shared_ptr```** is constructed from a unique-ownership ptr.

- A control block is created when a **```std::shared_ptr```** has its constructor called with a raw ptr.

This last point actually means that if we construct more than one **```std::shared_ptr```** from a raw ptr,
we'll have multiple control blocks for a single pointed-to resource.

Which then means that we can have a single object being destructed multiple times (***UB, UB, UB, UB, UB!!!***).

While they are not a perfect solution, and have some performance caveats(some which can be contained by the use of **```std::make_shared```**), they are perfectly usable and actually fine
for the functionality they provide(something like garbage collection
 functionality for the shared lifetime management of arbitrary resources).
