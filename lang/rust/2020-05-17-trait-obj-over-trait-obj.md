---
title: "Trait over ?Sized object"
subtitle: "Trait Object dynamic cast to Trait Object"
author: Guihao Liang
# published: true
tags: ['rust', 'cpp', 'c']
categories: ['rust', 'cpp', 'c']
---

TL;DR

trait object over fat pointers are not allowed in rust, i.e., no trait object over trait object and unsized struct, e.g., slices.

Following the [simplified model][1] of this question answered by [trentcl][2]. Let me explain in another perspective.

After reading the blog [explaining the implementation of trait object][3], the problem is that how could a vtable know the data pointer it has is a [fat pointer][4]. It always assumes the data pointer is a pure data pointer.

## dynamic cast

In Cpp, `dynamic_cast` is quite common for polymorphism and can be used to switch `vtable`s. Can Rust do the same thing?

```rust
trait Bar {
    fn bar_method(&self) {
        println!("this is bar");
    }
}
trait Foo: Bar {
    fn foo_method(&self) {
        println!("this is foo");
    }
}

impl Bar for u8 {}
impl Foo for u8 {}

fn main() {
    let x: u8 = 35;
    let foo: &dyn Foo = &x;
    // cast &dyn Bar to
    let bar: &dyn Bar = foo;
}
```

[playground][play-trait-cast]

```rust
error[E0308]: mismatched types
  --> src/main.rs:19:25
   |
19 |     let bar: &dyn Bar = foo;
   |              --------   ^^^ expected trait `Bar`, found trait `Foo
```

Initially, I thought `trait Foo: Bar` means `Foo` inherits from `Bar`. That's not precise. In Rust, there's no hierarchical inheritance, that is to say, `Bar` is the parent of `Foo`. Here, it simply means any type implements `Foo` __must also__ implement `Bar`. In other words, `Foo` and `Bar` are __siblings__.

But why they cannot be converted? Let's peek at [trait object][trati-impl]

Let's first have a peek at [trait object][5],

```rust
pub struct TraitObject {
    pub data: *mut (),
    pub vtable: *mut (),
}
```

This built-in struct is guaranteed to match abi layouts

In other words, let's say the pseudo code,

```rust
// let foo: &dyn Foo = &x;
let foo = TraitObject {
    data: &x,
    vtable: &foo_vtable,
}
// let bar: &dyn Bar = foo;
bar = TraitObject {
    data: foo.data,
    vtable: foo.vtable,
};
```

[play-trait-cast]: https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=52c669d40aeda9719e968181abbc2784
[trait-impl]: https://doc.rust-lang.org/nightly/std/raw/struct.TraitObject.html
[1]: https://stackoverflow.com/a/57432042/5335565
[2]: https://stackoverflow.com/users/3650362/trentcl
[3]: https://brson.github.io/rust-anthology/1/all-about-trait-objects.html
[4]: https://iandouglasscott.com/2018/05/28/exploring-rust-fat-pointers/
[5]: http://doc.rust-lang.org/nightly/std/raw/struct.TraitObject.html