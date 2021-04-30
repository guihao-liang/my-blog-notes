---
title: "Understand rust Vec indexing"
subtitle: "declaration vs definition time compiler check"
date: 2020-06-14 14:49:50
author: Guihao Liang
tags: ['rust']
categories: ['rust']
draft: true
---


As I read the documentation for rust `Vec`, I want to understand how index operator `[]` works. For example, `v[0]` is a syntax sugar for `*v.index(0)` and how `index` is implemented? This leads me to `Index` trait [implementation](https://doc.rust-lang.org/src/alloc/vec.rs.html#1940-1947).

```rust
impl<T, I: SliceIndex<[T]>> Index<I> for Vec<T> {
    type Output = I::Output;

    #[inline]
    fn index(&self, index: I) -> &Self::Output {
        Index::index(&**self, index)
    }
}
```

There are many things happening in above code snippet. Let me elaborate.

## Index Trait

```rust
pub trait Index<Idx>
where
    Idx: ?Sized,
{
    type Output: ?Sized;
    fn index(&self, index: Idx) -> &Self::Output;
}
```

`Idx` can potentially be an `unsized` type and being passed by value in above __declaration__. After [discussion][forum-discuss] in rust forum, Rust compiler __won't check the declaration__ only until it sees __definition__.

* __declaration__ time: declaring generics and trait.
* __definition__ time: instantiating generics, define a function with `body`, non-generic struct/enum and trait/method implementation.
* __both__: declaring and defining variables

Even though we know `Idx` type must be `sized` in order to be passed by value, the compiler doesn't care it at generic declaration time. That's reasonable, why force the compiler to think that far? The `Idx` type might not be any unsized type forever.

During a definition or implementation, if a `trait-typed` **value** is passed in, compiler will error out since `unsized` type can't be passed by value. Check the comparison between trait type, trait object in my [previous post](/2020/06/20/trait-over-trait).

The reason to declare this as `?Sized` instead of strictly `Sized` is that it serves as a __forward-compatible__ for __unsized local__. You can turn this feature on by adding `#![feature(unsized_locals)]`. See an example in [playground](https://play.rust-lang.org/?version=nightly&mode=debug&edition=2018&gist=2ddbdf71ea67ba17391d1c98d08a18cb).

Under this context, `Idx: ?Sized` is interpreted as __any type__ is acceptable for this generic but no guarantee it can be instantiated without any errors.

## Index::index: Universal Function Call Syntax

Universal Function Call Syntax, [UFCS][call-expression], is a **desugared** form of function call. The **sugared** form is `(&**self).index(index)`, which is less readable since it doesn't tell that this `index` is from `Index<T>` trait.

```rust
// *self -> Vec<T>
// **self -> [T]
// &**self -> &[T]
```

Thus, this `index` method is also from `Index` trait and it is defined in `[T]`:

```rust
impl<T, I> ops::Index<I> for [T]
where
    I: SliceIndex<[T]>,
{
    type Output = I::Output;

    #[inline]
    fn index(&self, index: I) -> &I::Output {
        index.index(self)
    }
}
```

Therefore, syntax `Index::index` means that it uses the `index` method from [Index trait implementation](https://doc.rust-lang.org/src/core/slice/mod.rs.html#2724-2734) for `[T]`. In this way, there may be other `index` methods for `[T]` but this unambiguously designates that the `index` method is from the implementation for `Index` trait and makes it more readable.

In the function body, it uses sugared form `index.index(self)` to refer to the method of an instance that implements `SliceIndex<[T]>`.

## SliceIndex Trait

Let's take vector indexing with `usize` index for example.

```rust
impl<T> SliceIndex<[T]> for usize {
    type Output = T;
    #[inline]
    fn index(self, slice: &[T]) -> &T {
        // N.B., use intrinsic indexing
        &(*slice)[self]
    }
```

As we can see, `vec[8]` is equivalent to `*8usize.index(&vec)`.

```rust
#![feature(slice_index_methods)]

use std::slice::SliceIndex;
pub fn main() {
    let v = vec![1;10];
    println!("{}", *8usize.index(&v));
}
```

Run above code in [playground](https://play.rust-lang.org/?version=nightly&mode=release&edition=2018&gist=e15028844024851a972e3c7fe6a2de89).

## Summary

`Vec<T>` indexing will redirect into indexing on `[T]`, which also implements `Index` trait. Then `[T]` will delegate the indexing job to an instance that implements the `SliceIndex<T>` trait in order to index on slice type.

The chain is quite long and involves a lot of function calls. Fortunately, the whole chain is __inlined__ (`#[inline]`) so that the overhead of extra storage for fat pointer `&[T]` can be negligible. Check more from my previous [fat pointer blog]({% post_url 20-06-06-fat-pointer %}).

## Reference

* [universal function call syntax][call-expression]
* [rust forum discussion][forum-discuss]

[forum-discuss]: https://users.rust-lang.org/t/need-explanation-for-passing-sized-as-value-in-generic-declaration/44235?u=jarvi-izana
[call-expression]: https://doc.rust-lang.org/reference/expressions/call-expr.html#disambiguating-function-calls
