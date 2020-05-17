---
title: "Learn rust fat pointer from a CPP programmer's perspective"
subtitle: "A peak at TraitObject coercion"
date: 2020-05-16 11:59:08
author: Guihao Liang
published: false
bigimg: "/img/tiles.jpg"
tags: ['rust', 'cpp', 'c']
categories: ['rust', 'cpp', 'c']
---

## CPP vpointer layout

## fat pointer layout

## inner vpointer vs fat pointer

## trait object layout

```rust
#[repr(C)]
#[derive(Copy, Clone)]
#[allow(missing_debug_implementations)]
pub struct TraitObject {
    pub data: *mut (),
    pub vtable: *mut (),
}
```

implicit coercion: `dyn`

```rust
// let the compiler make a trait object
let object: &dyn Foo = &value;

// look at the raw representation
let raw_object: raw::TraitObject = unsafe { mem::transmute(object) };
```

Possible implementation for vtable.

```rust
trait Foo {
    fn method(&self) -> String;
}

// and its vtable object
struct FooVtable {
    destructor: fn(*mut ()),
    size: usize,
    align: usize,
    // C: const void *
    method: fn(*const ()) -> String,
}
```
