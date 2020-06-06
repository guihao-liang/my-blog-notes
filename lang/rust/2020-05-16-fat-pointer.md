---
title: "Learn rust fat pointer from a Cpp programmer's perspective"
subtitle: "A peek at TraitObject and Slice implementation"
date: 2020-06-06 16:25:02
author: Guihao Liang
published: true
bigimg: "/img/tiles.jpg"
tags: ['rust', 'c', 'cpp']
categories: ['rust', 'cpp', 'c']
---

The concept for `Sized` and `Unsized` is related to the [type layout][rust-type-layout] in Rust

> Types where all values have the same size and alignment known at compile time implement the Sized trait and can be checked with the size_of and align_of functions. Types that are not Sized are known as dynamically sized types. Since all values of a Sized type share the same size and alignment, we refer to those shared values as the size of the type and the alignment of the type respectively.

What are types that don't have known size in compile time? In Rust, there are two `Unsized` types: Slice and Trait. These `Unsized` types use fat pointers to reference the underlying object. The C-ABI compatible for fat pointer layout is like below:<a name="layout"></a>

```c
// poitner1, pointer2
struct fat_pointer {
    void* first;
    void* second;
};
```

For [TraitObject][trait-object-layout] definition, `#[repr(C)]` denotes that the layout is C-ABI (size and alignment) compatible:

```rust
#[repr(C)]
#[derive(Copy, Clone)]
#[allow(missing_debug_implementations)]
pub struct TraitObject {
    pub data: *mut (),
    pub vtable: *mut (),
}
```

Note that this is the __final__ fat pointer **physical** memory layout for slice object **but** not for trait object. It's just a **conceptual** memory layout that is convenient for us to visualize trait object. Check [trait object conceptual layout](#view-from-asm) for more details.

## inner vs external vpointer

As my previous [post](../cpp/20-05-31-what-is-vtable-in-cpp.md) discussed, in C++, object carries a vtable pointer, even though you don't use those virtual functions, or use any polymorphism, you still have to carry a vtable pointer and pay for the increased object size. Some people call this vtable pointer pattern as `inner vpointer` since it resides inside of each object.

For Rust, it's different. If you don't touch any polymorphism by using trait object, you don't pay for extra size to carry this vtable pointer. Every function call is static, even for those virtual functions. When you use trait object, you use this extra vtable pointer to perform virtual functions dispatch and pay for the overhead.

> In general, C++ implementations obey the zero-overhead principle: What you don’t use, you don’t pay for. And further: What you do use, you couldn’t hand code any better.  -- Bjarne Stroustrup

Well, in this case, Rust wins over C++.

## slice layout

In Rust [vec deref][rust-vec-deref] implementation:

```rust
impl<T> ops::Deref for Vec<T> {
    type Target = [T];

    fn deref(&self) -> &[T] {
        unsafe {
            let p = self.buf.ptr();
            assume(!p.is_null());
            slice::from_raw_parts(p, self.len)
        }
    }
}
```

The field `first` is a pointer to the underlying buffer of the vector or the start of the slice, and rust will cast its type from `void*` to `T*` at compile time since `[T]` defines type for each element in slice. The `second` is a pure `usize` to define the __length__ of the slice.

As convention, let's code an example:

```rust
pub fn demo() {
    let x = "string".to_owned();
    let slice = &x[..];
    println!("{}", slice);
}
```

From [assembly code](https://godbolt.org/z/pzWcDF):

```c
    lea     rdi, [rsp + 40]
    call    <alloc::string::String as core::ops::index::Index<core::ops::range::RangeFull>>::index
    // move function call fat pointer result, 2 qword, to stack (temp vars)
    mov     qword ptr [rsp + 32], rdx
    mov     qword ptr [rsp + 24], rax
    // ...
    // slice storage: 2 qword
    mov     rax, qword ptr [rsp + 24]
    mov     qword ptr [rsp + 64], rax
    mov     rcx, qword ptr [rsp + 32]
    mov     qword ptr [rsp + 72], rcx
```

For slice type, the **physical** layout aligns with the C-ABI compatible [layout](#layout). As the comparison with [trait object conceptual layout](#view-from-asm), the physical layout matches because `second` (length of slice) can be different for different slice instances, even though they share the same slice type.

## trait object layout

Let's first dig into an example in [playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=f2c5b89a46b8f2291f20268fb6149bd5):

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
    x.bar_method();
    foo.bar_method();
}
```

`dyn` is used to distinguish the `Unsized` type from `Sized`. In previous Rust versions, you can do `let foo: &Foo = &x` for both, which is not helpful for readability since you can't tell Foo is a trait or a normal type at the first glance.

As we can see, object implements `Foo` should also implement `foo_method` and `bar_method`. Keep in mind, in Rust, there's no inheritance.`Foo` is not a `Bar`. The syntax `trait Foo: Bar` simply means if type wants to implement `Foo`, it must also implement `Bar`.

```c
# C++ equivalent for `import Foo for u8`
class Baz : public Foo, public Bar { ... }
```

The `Baz` should carry a vpointer that has all information about the virtual functions, which are trait methods defined in `Foo` and `Bar` declaration. Since there's no multiple inheritance in Rust, its `vtable` can be as simple as what we've seen in [vtable for C++](../cpp/20-05-31-what-is-vtable-in-cpp.md). In other words, it only needs to list all methods inside of the vtable in a __deterministic order__. That is to say, if we implement `Foo` for other types, the `bar_method` should have the same position in the vtable. The story is complicated for vtable layout when multi-inheritance is involved.

```rust
// below is all pseudo code
pub struct TraitObjectFoo {
    data: *mut (),
    vtable_ptr: &VTableFoo,
}

pub struct VTableFoo {
    layout: Layout,
    // destructor
    drop_in_place: unsafe fn(*mut ()),
    // methods shown in deterministic order
    foo_method: fn(*mut ()),
    bar_method: fn(*mut ()),
}

// fields contains Foo and Bar method addresses for u8 implementation
static VTABLE_FOO_FOR_U8: VTableFoo = VTableFoo { ... };

// let foo: &dyn Foo = &x;
let foo = TraitObjectFoo {&x, &VTABLE_FOO_FOR_U8}
```

At runtime, `foo` has a vtable `VtableFoo` and compiler knows the offset of `bar_method` in that table.

```c
// foo.bar_method();
foo.vtable.offset(4)(foo.data);
```

<a name="view-from-asm"></a> We can also verify above in [assembly code](https://godbolt.org/z/9JpvM3):

```c
// let x: u8 = 35;
mov     byte ptr [rsp + 7], 35
// let foo: &dyn Foo = &x;
lea     rax, [rsp + 7]
// foo.bar_method();
mov     rdi, rax                           // move self, &x
call    qword ptr [rip + .L__unnamed_1+32] // bar_method(self)
// Foo vtable for u8
.L__unnamed_1:
        .quad   core::ptr::drop_in_place
        .quad   1
        .quad   1
        .quad   example::Foo::foo_method
        .quad   example::Bar::bar_method
```

As we can see in above assembly example, `let foo:&dyn Foo = &x` only takes the address of `x` (1 qword) instead of constructing this [C-ABI layout](#layout) for variable `foo`, which requires 2 qwords. Thus, the [C-ABI layout](#layout) is not the final memory layout for the compiled rust code. We can verify this by running [std::raw::transmute][trait-object-doc] example and reading the compiled [assembly code](https://godbolt.org/z/TJtm37):

```c
// load vtable pointer address
// rip contains address of current instruction being executed in CPU
        lea     rax, [rip + .L__unnamed_2]
// ...
// let raw_object: raw::TraitObject = unsafe { mem::transmute(object) };
        mov     qword ptr [rsp + 232], rcx // data*
        mov     qword ptr [rsp + 240], rax // vtable*
```

The reason __I guess__ is that the compiler knows the vtable address for `Foo` __type__, i.e., `.L__unnamed_2`. So there's no need to pay extra storage for all `Foo` instances since all of them share the same vtable from `Foo` __type__. This is an __optimization__ benefited from the fat pointer or external vtable layout.

## references

* [all about trait object][trait-object-impl]
* [any dynamic-cast for trait object?][dynamic-cast-rust]
* [exploring fat pointer](https://iandouglasscott.com/2018/05/28/exploring-rust-fat-pointers/)
* [std::raw::TraitObject][trait-object-doc]

[dynamic-cast-rust]: https://users.rust-lang.org/t/why-cant-i-do-dynamic-cast-in-rust-as-i-do-in-c/42793
[rust-type-layout]: https://doc.rust-lang.org/reference/type-layout.html
[rust-vec-deref]: https://github.com/rust-lang/rust/blob/master/src/liballoc/vec.rs
[trait-object-doc]: https://doc.rust-lang.org/std/raw/struct.TraitObject.html
[trait-object-impl]: https://brson.github.io/rust-anthology/1/all-about-trait-objects.html
[trait-object-layout]: https://doc.rust-lang.org/src/core/raw.rs.html#83-86