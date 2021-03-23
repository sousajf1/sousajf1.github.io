---
layout: post
title: "Smart-pointers: std::weak_ptr"
---
In this third post about smart pointers we are going to talk about **```std::weak_ptr```** .

As far as the machine is concerned, it has the same *physical layout* as the **```std::shared_ptr```**, that
means a pointer to the resource, and also a pointer to the control block.

The difference is when you copy a **```std::weak_ptr```** you will increment the ***weak reference count*** instead.

As usual, we are going to quote Scott Meyers on this post,
on his *Effective Modern C++*, Item 20:

> Use std::weak_ptr for ***std::shared_ptr-like*** pointers that can dangle.

What does this mean?

So let's imagine this situation:

- We have a **```std::shared_ptr```** pointing to some resource.
- We also have a **```std::weak_ptr```** pointing to the same resource.
- They share the same control-block, which has these reference counts right now:
    * reference-count : 1
    * weak-reference-count : 1

Imagine we then destroy the **```std::shared_ptr```**. How does the whole situation looks right now?

- **```std::shared_ptr```** was destroyed so we no longer 
have a **```std::shared_ptr```** pointing to the resource.
- The reference-count is now zero so the resource is also destroyed.
- But the **```std::weak_ptr```** still points to an existing control-block, which wasn't destroyed,
because the weak-reference-count was 1, so it wasn't destroyed when **```std::shared_ptr```** was destroyed:
    * reference-count : 0
    * weak-reference-count : 1
- So now **```std::weak_ptr```** has a ptr to a control block which has a reference count of zero.
- **```std::weak_ptr```** knows it is **dangling**.
 
Another thing is that you cannot dereference a **```std::weak_ptr```**.
You can think like this:
- If one has a **```std::weak_ptr```** he is entitled to convert it to a **```std::shared_ptr```** if and when he
wants(if the pointed resource still exists, of course).

One can do that using the member function **```lock()```** (or through explicit type-conversion 
(this will throw if the **```std::weak_ptr```** is dangling)).

It's actually a good case to use a variable declaration inside an if statement:
```cpp 
...
std::weak_ptr<int> wk_ptr;
...
if(auto sh_ptr = wk_ptr.lock()) {
    // do something with it
}
```

Because **```lock()```** will return **```nullptr```** if the **```std::weak_ptr```** is dangling.

