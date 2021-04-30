---
title: "Trait over Trait"
subtitle: "Trait type vs Trait object"
author: Guihao Liang
date: 2020-06-14 20:16:16
draft: true
tags: ['rust', 'c++']
categories: ['rust', 'c++']
---

It's very straightforward to understand a trait implementation over concrete type.

```rust
impl Trait for i32 {
    // ...
}
```

you can say for any `i32` __value__, it has properties from `Trait`.

When it comes to `impl TraitA for TraitB`, it's quite confusing because `TraitB` is a trait, which is a abstract type without a fixed size. How can you say `TraitB` __value__ can call methods from `TraitA`?

Let's look at one example:

```rust
trait A {
    fn method_a(&self) {
        println!("a");
    }
}

trait B {
    fn method_b(&self) {
        println!("b")
    }
}

// same as impl B for A {}
impl B for dyn A {}

impl A for i32 {}

fn main() {
    let a = 10i32;
    a.method_a();

    let b: &dyn A = &10;
    b.method_b();
}
```

[playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=313a4cb0e69ed53dcc2961530d515183)

## trait type vs trait object

```rust
    let a = 10i32;
    a.method_a();
```

It's easy to understand the first one since you can get `i32` type __by value__. But how about taking `dyn A` by value? If you can't get a `dyn A` type with value, how can you call `method_b`? As we know, trait type is __unsized__, thus you can't get it by value. Then what's the purpose for `impl B for dyn A`?

What happens to `x.method_b` first is automatic dereferencing, which is equivalent to `(*x).method_b()`. In OOP language, the receiver is `*x`, which is of type `dyn A`. However, this doesn't mean that we get __trait typed value__, for example.

```rust
// can't compile with stable
let y: dyn A = *x;
```

Rust uses the type info but **not** the value to verify trait `B` is implemented for type `A` (or `dyn A`), so that it can call `method_b`. Since the signature of `method_b` needs a reference, it will automatically reference from `*x` to `B::method_b(&*x)`. In the end, it gets a type `&dyn A` to `self`, which is a trait object.

`dyn A` value existence is __transient__, and it transits to trait object `&dyn A` in the end.

## type and storage

Type is not always associated with storage for Rust. From C/C++ that every type must comply with

1. not only defines the properties (from OOP)
2. but also defines the layout how the compiler should store it in memory, which is the size for the storage.

When you use type, it's a matter of whether you can use it to declare a value or not. Even though you are prohibited to declare abstract class by value, abstract class still has a clear memory layout from compiler perspective. The best example is C++ incomplete type,

```rust
struct Node {
     Node m_node; // recursive declare, infinite size
    // Node* m_node; // makes compiler happy. size_of Node is 8 bytes on 64bits
};
```

For Rust, A type can have 1, 2 at the same time (sized), or only 1 without 2 being satisfied (unsized).

dyn Foo is, as you said, a transient thing - it doesn't really exist by itself. It's a non- Sized value which stores part of itself in whatever its being stored behind.

In my mental model, I think of dyn Foo as the data half of the trait object + an obligation on whatever is storing it. The first part of dyn Foo is thing at the end of the *mut () in the code you posted. The second is an abstract concept of "whatever is storing me stores a vtable too".

Now I understand what you mentioned earlier. dyn Foo is an API obligation (protocol) not the storage schema.

dyn Foo can be viewed as *&dyn Foo, further [*data, *vtable], which refers to an object of Sized type that can be stored in memory (meet requirement 1 and 2) + the vtable object (also stored in memory). But you don't what's the sized type object. Therefore, it's unsized.

I think it's better for Rust to clarify the type can be sized (1, 2) or unsized (1). Probably DST already mentions it but that's confusing because people may think DST under the old C/C++ type model.

## References

* Question I posted: [what does it mean to implement Trait for Trait][forum-discuss]

[forum-discuss]: https://users.rust-lang.org/t/what-does-it-mean-to-implement-trait-for-trait/44031


[play-trait-cast]: https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=52c669d40aeda9719e968181abbc2784
[trait-impl]: https://doc.rust-lang.org/nightly/std/raw/struct.TraitObject.html
[1]: https://stackoverflow.com/a/57432042/5335565
[2]: https://stackoverflow.com/users/3650362/trentcl
[3]: https://brson.github.io/rust-anthology/1/all-about-trait-objects.html
[4]: https://iandouglasscott.com/2018/05/28/exploring-rust-fat-pointers/
[5]: http://doc.rust-lang.org/nightly/std/raw/struct.TraitObject.html
