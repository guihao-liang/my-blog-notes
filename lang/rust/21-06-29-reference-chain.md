---
layout: post
title:  "Chained references in rust"
subtitle: "References of references"
categories: ["rust"]
date: 2021-06-29 16:00:11
published: false
---

goal:
data race happens when these 3 behaviors occur:

1. 2 or more pointers access same data at the same time.
2. at least one pointer is used to write (Therefore, no mutable and immutable at same time)
3. no synchronization is provided to protect the data

language spec:

one level deep;

> These point to memory owned by some other value. When a shared reference to a value is created it prevents direct mutation of the value

```rust
 let mut xx = vec![1, 2];
    let yy = &mut xx;
    let zz = &yy;
    let gg = zz;
    gg.push(1);
```

> This is only one of the reasons. Preventing two mutable references makes it possible to keep invariants on types easily and let the compiler enforce that the invariant are not violated.

[here][vlog-recommend]
> mutable reference is only one level deep

think of it as unique access. Only unique access can mutate the data, in whatever depth

- [ ] https://www.reddit.com/r/rust/comments/93owat/deep_mutability_design_rationale/
- [ ] https://www.reddit.com/r/rust/comments/7d9pkg/why_does_rust_not_allow_borrow_references_and_a/

[so-discuss]: https://stackoverflow.com/questions/58364807/why-rust-prevents-from-multiple-mutable-references
[spec]: https://doc.rust-lang.org/reference/types/pointer.html#shared-references-
[vlog-recommend]: https://www.youtube.com/watch?v=rAl-9HwD858&t=3288s
[unique-shared]: https://limpet.net/mbrubeck/2019/02/07/rust-a-unique-perspective.html
