---
title: "Learn Rust fat pointer from a Cpp programmer's perspective"
subtitle: "Understand TraitObject and Slice implementation"
date: 2020-06-06 16:25:02
author: Guihao Liang
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

Note that this is the fat pointer **physical** stack layout for slice object **but** not always true for trait object. For trait object, `second` is a constant and it may be optimized to use register not stack memory. Check laster part [trait object type erasure](#view-from-asm) for more details.

## inner vs external vpointer

As my previous [post]({% post_url /lang/cpp/20-05-31-what-is-vtable-in-cpp %}) discussed, in C++, object carries a vtable pointer, even though you don't use those virtual functions, or use any polymorphism, you still have to carry a vtable pointer and pay for the increased object size. Some people call this vtable pointer pattern as `inner vpointer` since it resides inside of each object.

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

For slice type, the **physical** layout aligns with the C-ABI compatible [layout](#layout). As the comparison with [trait object type erasure](#view-from-asm), the physical layout **always** matches because unlike the `second` (__length__ of slice), is not a __constant__ and can be different on instance-by-instance, even though they share the same slice type.

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
# C++ equivalent for `impl Foo for u8`
class Baz : public Foo, public Bar { ... }
```

The `Baz` should carry a vpointer that has all information about the virtual functions, which are trait methods defined in `Foo` and `Bar` declaration. Since there's no multiple inheritance in Rust, its `vtable` can be as simple as what we've seen in [vtable for C++]({% post_url /lang/cpp/20-05-31-what-is-vtable-in-cpp %}).

### different vtable layout for different Trait type

In other words, `Foo` and `Bar` are __siblings__ and there's no hierarchy between them, i.e., a `Foo` is not a child of `Bar`. At compile time, all other dependent traits are known for the base trait and the compiler can define vtable layout for each different traits because different traits can not be dynamically cast to others, i.e., no hierarchical inheritance in Rust. I will explain this later.

From this [example code](https://godbolt.org/z/Wf3Dow), we know

```rust
trait Bar: A + B { /* ... */ }
```

Since there's no multiple inheritance, the vtable for `Bar` can just __simply stacks__ methods from `A`, `B` and `Bar` itself.

```c
.L__unnamed_1:
        .quad   core::ptr::drop_in_place // destructor
        .quad   1 // size
        .quad   1 // align
        .quad   example::Bar::bar_method
        .quad   example::B::b_method
        .quad   example::A::a_method
```

Thus, the relative order of the methods is __deterministic order__. When you want to call `b_method` from a trait object `&dyn Bar`, the compiler knows to offset vtable by 32.

Also, vtable will __only__ be generated when Trait Object is requested. That is to say, if you don't use it, you don't pay the storage for the vtable. Feel free to follow the comment in above [example](https://godbolt.org/z/Wf3Dow) and check corresponding assembly code.

### different trait object for different Trait type

For different trait types, the different trait objects are synthesized by rust compiler. For example, `Foo` trait would have a corresponding trait object `TraitObjectFoo` when `&dyn Foo` is requested, whereas `Bar` has `TraitObjectBar`. That is to say, even though `Foo` extends `Bar`, it's __prohibited__ to __dynamically cast__ `&dyn Foo` to `&dyn Bar`, which is analogous to convert `TraitObjectFoo` to `TraitObjectBar`.

Let's go through an example to get better understanding of above observation.

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

### vpointer is not always stacked in stack

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

As we can see in above assembly example, `let foo:&dyn Foo = &x` only takes the address of `x` (1 qword) instead of constructing this [C-ABI layout](#layout) on stack for variable `foo`, which requires 2 qwords. Instead, vpointer is stored in register in a disposable manner. We can also verify this by running [std::raw::transmute][trait-object-doc] example and checking the [assembly code](https://godbolt.org/z/TJtm37):

```c
// load vtable pointer address to rax; hit and run
// rip contains address of current instruction being executed in CPU
    lea     rax, [rip + .L__unnamed_2]
// ...
// let raw_object: raw::TraitObject = unsafe { mem::transmute(object) };
    mov     qword ptr [rsp + 232], rcx // data*
    mov     qword ptr [rsp + 240], rax // vtable*
```

The reason is that the compiler knows variable `x` is a `Foo` type and the vtable address for `Foo`, i.e., a __const__ relative offset `.L__unnamed_2`. Thus, there's no need to pay extra storage in stack for `Foo` instance `x` and register can be used for faster access the vpointer for `Foo`. This is an __optimization__ benefited from the fat pointer or external vtable layout. It's like:

```c
// c++ pseudo code
constexpr vpointer_for_foo = 0x1234;
Foo foo;
foo.data = &x;
foo.vpointer = vpointer_for_foo;
foo.vpointer.offset(4)(foo.data);
```

The vpointer can be __inlined__ with optimization:

```c
// c++ pseudo code
constexpr vpointer_for_foo = 0x1234;
int* data = &x;
(vpointer_for_foo+4)(data);
```

Therefore, register is sufficient to handle inlined value and no need to bother stack because it's much more slower.

### vpointer is stacked during type erasure

In this __type erasure__ [example](https://godbolt.org/z/pZG-BF), we can __erase__ the `x`'s type info by

```rust
trait Base {
    fn base_method(&self) {
        println!("this is Base");
    }
}

// x type info is erased when returned
fn test_dyn_ret(x: &dyn Base) -> &dyn Base {
    x.base_method();
    x
}
```

In __outer__ scope of `test_dyn_ret` function, `x` is a `Foo` type that compiler __assures__. However, type information is __erased__ in inner scope of `test_dyn_ret`, and the compiler doesn't have any knowledge of the type of `x` anymore. Compiler only knows a `&dyn Base` is returned not assuring the underlying instance's type info inside that returned trait object. Therefore, it needs to store this vpointer to __stack not only in callee frame but also in caller frame__, i.e., the outer scope. The reason of this different behavior with previous [inlined vpointer](#view-from-asm) is that it doesn't know how to infer and load the vpointer value inline, otherwise it would use register.

```c
    // let foo = test_dyn_ret(&y);
    call    example::test_dyn_ret
    mov     qword ptr [rsp + 16], rax // vpointer in caller stack
    mov     qword ptr [rsp + 8], rdx  // self in caller stack
```

Above layout is the [C-ABI layout](#layout) we've seen. That's __type erasure__ for rust and the most common usage for `Trait`.

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
