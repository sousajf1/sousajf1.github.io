---
layout: post
title: "Smart-pointers: std::unique_ptr"
---
A smart pointer is a class that wraps a raw pointer, acts pretty much  as raw pointer, but it can manage the lifetime of the object being pointed to.

As of C++11 there are four smart pointers in the standard library: **```std::unique_ptr```**, **```std::shared_ptr```** , **```std::weak_ptr```** and **```std::auto_ptr```**.

**```std::auto_ptr```** was a leftover of a C++98 attempt to create something like **```std::unique_ptr```** but, mostly due to the lack of move semantics, had some huge limitations.

It was deprecated in C++11 and removed in C++17.

On this post we want to explore and get to know more about  **```std::unique_ptr```**.

As Scott Meyers pointed out on his *Effective Modern C++*, Item 18:
> Use std::unique_ptr for exclusive-ownership resource management. 

In this context means manage a (heap) resource that otherwise we would have to manage manually.

Let's see an example using raw pointers:
```cpp 
{
    /* 
    ...
    */
    T* my_ptr = new T;
    // ...
    delete my_ptr;
    /* 
    ...
    */
}
```
So in this case, we have a variable of type T pointer on the stack, and it's 
being initialized to point to the address of a heap allocated type T object.


So like this, it's our responsibility to delete the allocated resource(**```delete ptr;```**) 
or otherwise we will get a memory leak.

A raw pointer is also copyable:

```cpp 
{
    /* 
    ...
    */
    T* my_ptr = new T;
    T* my_copy_ptr = my_ptr;
    /* 
    Which of them has the clean up responsibility ?
    ...
    */
}
```

That also makes very unclear who has the ownership of the heap allocated object. Who has to clean-up?

Taking a look inside an example unique_ptr class template, which can look something like this:
```cpp 
{
    template<typename T>
    class unique_ptr {
    private:
        T* ptr;
    public:
        // ... 
        ~unique_ptr() {
           delete ptr;
        }
   

        unique_ptr(unique_ptr<T>&& move_ptr) noexcept
        {
           std::swap(ptr, move_ptr.ptr);
        }

        unique_ptr<T>& operator=(unique_ptr<T>&& move_ptr) 
        noexcept
        {
            std::swap(ptr, move_ptr.ptr);
            return *this;
        }

        unique_ptr(const unique_ptr<T>& ptr) = delete;
        unique_ptr<T>& operator=(const unique_ptr<T>& ptr)
        = delete;
    }
```


Let's see what unique_ptr offers us:

- A non-null **```std::unique_ptr```**  always  owns  what  it  points  to.
- Moving  a  **```std::unique_ptr```**  transfers ownership from the source pointer to the destination pointer.
- **```std::unique_ptr```** is a move-only type (This preserves unique ownership).
- Upon   destruction, a non-null **```std::unique_ptr```**  destroys  its  resource, and by default does that by applying **```delete```** to the
pointer inside **```std::unique_ptr```** . 
- **```std::unique_ptr```** also has a specialization for array types. If I have a **```std::unique_ptr```** to an array of ```T``` 
at the end of the lifetime it will call the array form of **```delete```**.

But actually **```std::unique_ptr```** has one more template parameter: a deleter !

It's defaulted to [std::default_delete](https://en.cppreference.com/w/cpp/memory/default_delete), but one can pass its own deleter type.

```cpp 
{
    template<class T, class Deleter = std::default_delete<T>>
    class unique_ptr {
    private:
        T* ptr;
        Deleter d;
    public:
        // ... 
        ~unique_ptr() {
           if(ptr) d(ptr);
        }
        //...
        unique_ptr(const unique_ptr<T>& ptr) = delete;
        unique_ptr<T>& operator=(const unique_ptr<T>& ptr) 
        = delete;
    }
```

As stated in the example of Scott Meyers book, this is actually pretty cool, if you need, for example, to add a log entry before deletion,
you can just:

```cpp
auto logAndDel = [](logAndDelType* pLogAndDel) {  
    makeLogEntry(pLogAndDel);  
    delete pLogAndDel;
};
```

This can then be passed to the ptr:

```cpp
std::unique_ptr<logAndDelType, decltype(logAndDel)> 
pDel(nullptr, logAndDel);
};
```

This is pretty useful to encapsulate the responsibility of handling resources as file closing, logging, handling with some API specific types
by wrapping those on unique_ptr class interfaces with custom deleters.
