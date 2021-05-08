---
layout: post
title: "C++ vs Rust Memory Management: a (very) brief overview"
---
#### What this post will NOT be about?
- the subtlest details of each language.
- best practices.
- strategies and patterns.

#### What this post aims to be about?
- give a very high level view of how modern C++ and Rust handle memory.
- pointing out some safety issues.
- and we'll see what more will come up. 

This post is heavily based and follows/extends a presentation done by my colleague
[Luiz Jardim](https://www.linkedin.com/in/luiz-jardim-b434b0170/).

## C++

Basically, C++ allocates on the Stack by default, and on the Heap
when using ```new```. 
```cpp 
{
    int i; // stack
    int* i2 = new int; // pointer to heap
    int* i3 = &i; // pointer to stack
    A obj; // stack
    A* obj2 = new A; // pointer to heap
}
```

Memory in the Heap must be manually freed by
using ```delete```.
```cpp 
{
    (...)
    delete i2;
    delete obj2;
}
```

This idiosyncrasy can be the evil and the source of many bugs: memory leaks, dangling pointers, and double free.

##### RAII

C++ programmers use a technique called *Resource
Acquisition Is Initialization*.
When an object goes out of scope, it’s destructor is called. The
destructor frees the memory.

Let’s try to write a class that holds a pointer to a heap object:

*easier said than done...*

```cpp 
class Int {
public:
    explicit Int(int value) : i{new int{value}}
{}
~Int() { delete i; }
private:
    int* i;
};
```

Now let's use it:
```cpp 
{   
    (...)
    Int i{3}; // calls new
    {
        Int i2{i}; // copies the pointer
    } // calls delete
}
```

Upsss...now the pointer inside ```i``` is dangling, and when ```i``` goes out of scope
there will be a double free.

**RAII** only works for resources acquired and released by 
stack-allocated objects, with a well-defined static object lifetime.

Heap-allocated objects need to be implicitly 
or explicitly deleted (once) along all possible execution paths.

This can usually be achieved by the use of **smart pointers**.

##### Default Initialization

[Default Initialization rules](https://en.cppreference.com/w/cpp/language/default_initialization) in C++ are tricky (and extensive).

Looking at a small example:
```cpp 
struct Int {
    int value;
};

struct Int2 {
    int value{};
};
int main() {
    int i; // indeterminate
    Int i1; // i1.value is indeterminate
    Int i11{}; // i11.value is 0
    Int2 i2; // i2.value is 0
    return 0;
}
```

##### Argument Passing

In C++, arguments are passed by value.
To avoid expensive copies, you can pass a pointer or a reference.

```cpp 
//foobar.h

void foo(S);
void bar(S&);
```

```cpp 
//main.cpp

int main() {
    S s;
    foo(s); // passed by value (creates a copy)
    bar(s); //  reference
    return 0;
}
```

Also you can't tell if ```bar()``` is getting ```s``` passed by value or reference
without taking a peek into ```foobar.h```.

When passing by reference you can (sometimes unwillingly) alter the original object.

If you don't want to do it you can use the ```const``` keyword in the signature as a promise not to
change the referenced value.

```cpp 
//foobar.h

(...)
void bar(S const&);
```

```cpp 
//main.cpp

int main() {
    S s;
    (...)
    S const s1;
    bar(s1); // bar(S const&) is called
    return 0;
}
```

In C++11, move semantics were added.
Moves can steal the resources of the object, leaving it in an
unspecified state.

```cpp 
//foobar.h

(...)
void foo(S&&);
```

```cpp 
//main.cpp

int main() {
    S s;
    (...)
    foo(std::move(s)); // foo(S&&) is called
    return 0;
}
```

After this ```std::move(s)``` the programmer should not use ```s``` anymore!

## Rust

##### The Ownership Model

In Rust, each value has exactly one variable that owns it.
When that variable goes out of scope, the value is *dropped*.
Assignments pass the ownership of the value.
```rust 
struct S {}

fn main() {
    let s1 = S{}; // s1 is the owner of the value
    let s2 = s1; // s2 is now the owner
    let s3 = s1; 
}
```

Booom...

```rust 
error[E0382]: use of moved value: `s1`
 --> <source>:6:14
  |
4 |     let s1 = S{}; // s1 is the owner of the value
  |         -- move occurs because `s1` has type `S`, which does not implement the `Copy` trait
5 |     let s2 = s1; // s2 is now the owner
  |              -- value moved here
6 |     let s3 = s1; // compilation error
  |              ^^ value used here after move
```

##### Copies

The compiler can generate a ```Clone``` method for your type if you want
it to.

```rust 
#[derive(Clone)]
struct S {}
fn main() {
    let s1 = S{}; // s1 is the owner of the value
    let s2 = s1.clone(); // s2 is the owner of the cloned value
    let s3 = s1; // s3 is now the owner of the first value
}
```

If the type implements the ```Copy``` trait, the copy is implicit.

This is done for types that are cheap to copy, such as ```i32```, ```bool``` and
```char```, for example.

```rust 
#[derive(Clone, Copy)]
struct S {}
fn main() {
    let s1 = S{}; // s1 is the owner of the value
    let s2 = s1; // s2 is the owner of the cloned value
    let s3 = s1; // s3 is the owner of another value
}
```

##### Copies

References are variables that don’t own any value.
You can have as many references to a value as you want at the same
time.

```rust 
struct S {}
fn main() {
    let s1 = S{}; // s1 is the owner of the value
    let s2 = &s1; // borrow. s1 is still the owner
    let s3 = s1; // s3 is now the owner
}
```

The compiler keeps track of lifetimes so you don’t have dangling
references and you don’t have runtime performance issues. (Similar
to clang’s ```-Wlifetime```).

```rust 
struct S {}
fn main() {
    let s1 = S{}; // s1 is the owner of the value
    let s2 = &s1; // borrow. s1 is still the owner
    let s3 = s1; // s3 is now the owner
    let s4 = s2; // compilation error
}
```

##### Mutable References

All variables are immutable by default in Rust!

The keyword ```mut``` makes them mutable.

```rust 
struct S {}
fn main() {
    let mut s1 = S{}; // the value is mutable
    let s2 = &mut s1; // s2 can alter the value now
}
```

To prevent data races, if there’s a mutable reference, there cannot be
any other references.

```rust 
struct S {}
fn main() {
    let mut s1 = S{}; // the value is mutable
    let s2 = &mut s1; 
    let s3 = &mut s1;
    let s4 = s2;
}
```

Pufff...

```rust 
error[E0499]: cannot borrow `s1` as mutable more than once at a time
 --> <source>:5:14
  |
4 |     let s2 = &mut s1; 
  |              ------- first mutable borrow occurs here
5 |     let s3 = &mut s1;
  |              ^^^^^^^ second mutable borrow occurs here
6 |     let s4 = s2; // compilation error
  |              -- first borrow later used here
```

##### Initialization

You must explicitly initialize the values before using them.

```rust
struct S {}
fn mut_borrow_arg ...
fn main() {
    let mut s : S;
    mut_borrow_arg(&mut s);
}
```
```rust
error[E0381]: borrow of possibly-uninitialized variable: `s`
 --> <source>:5:20
  |
5 |     mut_borrow_arg(&mut s);
  |                    ^^^^^^ use of possibly-uninitialized `s
```

Also Rust generally makes it unnecessary to leave variables uninitialized.

For example if one has something like:

```rust
let mut s;
if ... {
    s = 1
} else {
    s = 0
}
```

One can do instead something like:
```rust
let mut s = if ... {
    1
} else {
    0
};
```

##### Argument Passing

It’s always clear what a function can/will do by reading the call.
```rust 
let mut s = S{};
mut_borrow_arg(&mut s); // can alter the value
borrow_arg(&s); // cannot alter the value
move_arg(s); // takes ownership of the value
```

##### Heap Allocation

For heap allocation, use a smart pointer such as Box.

```rust 
let b = Box::new(0);
let s = Box::new(S{});
```

The value is not created in-place. It is created on the Stack, then
moved into the function ```Box::new```, which then moves it to the Heap.

Check other smart pointers available in Rust [here](https://doc.rust-lang.org/book/ch15-00-smart-pointers.html).

##### RAII

Rust also follows RAII.
Resources are freed when their owner variable goes out of scope, by
calling the drop method.

Custom types can implement their own drop method, if necessary.

```rust 
struct S {}
impl Drop for S {
    fn drop(&mut self) {
        println!("during drop");
    }
}

fn main() {
    {
        let s = S {};
        println!("before drop");
    } //prints "during drop"
    println!("after drop");
}
```

