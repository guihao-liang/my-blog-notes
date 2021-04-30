---
title: "Learn rust mut from a Cpp programmer's perspective"
subtitle: "Rust mut vs Cpp const"
author: Guihao Liang
date: 2020-05-30 18:04:37
tags: ['rust', 'cpp', 'c']
categories: ['rust', 'cpp', 'c']
---

At beginning of learning rust, I mistakenly thought `mut` is a type decorator,

```rust
fn foo(v: mut i32) {
    // blah
}
```

This won't compile since there's no such type called `mut i32`. but in C,

```c
void foo(int const bar, const int baz) {
    // blah
}
```

It seems like there's a type called `const int` or `int const`.

However, I realize I had misunderstood the `const` decorator.

`const` decorates the __variable__ `v` instead of __type__ `int`, denoting the variable (the memory bound to this variable name) is immutable. The same thing applies to `mut`, whose usage is counterpart to `const` in C/C++. Everything in C/C++ is mutable by default and you need to use `const` to notify the compiler that you need special treatment. Rust does this in a reverse manner.

```c
// C/C++ variable declaration
int const v;

// pseudo code for rust, and const is for demo
const v: int;
```

What rust does is to flip the order of the variable declaration from `type decorator var` into `decorator var : type`, with a `:` to visually separate type and variable declaration.

What about [pointers and references][ref-pointer]?

> RawPointerType : * ( mut | const ) TypeNoBounds
> ReferenceType : `&` Lifetime? `mut`? TypeNoBounds

In terms of the syntax definition, it's the same as C/C++ that decorator `mut` and `const` is part of the pointer and reference `type`.

For C/C++, the reference and pointer type are like below.

```c
// virtually separate & * with type and variable name
int * ptr;
int * const ptr;
const int * ptr;
int & ref; // ref variable is always `const`
const int & ref;
```

Let's convert them to rust's declaration.

* add `:` to visually separate type declaration and variable declaration.

```c
int * : ptr;
int * : const ptr;
const int * : ptr;
int & : ref; // ref variable is always `const`
const int & : ref;
```

* swap variable and type declaration over `:` and reversely output tokens of the type declaration

```rust
ptr : * int;
ptr : * const int;
ref : & int;
ref : & const int;
```

* remove `const` and add `mut` to mutable variable and pointer type

```rust
let mut ptr : * mut int;
let mut ptr : * int;
let ref : & mut int;
let ref : & int;
```

In rust, reference variable (memory that stores the address) can be __mutable__, whereas in C/C++, reference variable are immutable and value can be only set at initialization.

All in all, I'm in favor of C/C++ syntax and not into Rust's variable declaration or definition syntax. However, Rust provides a visual separator, which is good for readability in a long run.

[ref-pointer]: https://doc.rust-lang.org/reference/types/pointer.html?highlight=ReferenceType#references--and-mut
