---
title: Understand rust lifetime in a compiler way
subtitle: All you need to start with rust lifetimes
date: 2021-05-01 11:06:32
draft: true
---

tl;dr,

1. skim [lifetimes][the-book] from the rust book.
2. read [lifetimes][the-nomincon] from the Rustonomicon to learn how compiler chooses lifetime for reference.
3. evaluate what you've learn with [lifetime misconceptions][misconception].
4. get hands dirty with [too many linked list][too-many-linked-list], the best rust practice book ever.

Exercises for self-evaluation:

1. Lifetime is same with scope?
2. is lifetime of a reference the same with the lifetime of the referent (non-reference type)?
3. why lifetime annotations are needed? Can't compiler infer the lifetime for functions?
4. `&'a T` is `T: 'a`?

---

## Quick start

> Lifetimes are named regions of code that a reference must be valid for.
> -- [the nomincon][the-nomincon].

* reference always has lifetime annotation, explicitly or implicitly.

* lifetime of a **reference** is **named scope or region** of code that the reference must be valid for. It's up to compiler to choose appropriate region for the reference, which is **upper bounded** by the lifetime of the referent.

* lifetime of a **variable** is named scope or region of code where the variable is constructed and destroyed.

* lifetime annotations show the **relationship** between input and output parameters.

lifetime is the key feature that distinguishes rust from C/C++ and it's the most import part of the `ownership` concept.

## How compiler annotate lifetime

I found corresponding chapters from [the book][the-book] is a good introduction about lifetime, but it doesn't tell you how compiler chooses the lifetime for references.

For a simple example based on the fact below:

> One particularly interesting piece of sugar is that each let statement implicitly introduces a scope.
> -- [the nomincon][the-nomincon]

```rust
{
    let x = 5;            // ---------+-- 'a
                            //          |
    {                     //-'b       |
        let r = &x;       //  |       |
    }                     // -+       |
}                         // ---------+
```

`'a` and `'b` denotes the lifetime for variable `x` and `r` respectively.

If we were allowed to label the lifetime for `let` clause, what's the lifetime label for `&x`? Should it be `&'b x` or `&'a x`?

If you choose `&'b x`, you can say because the lifetime of variable `r` is `'b`, you can't use `r` outside named region `'b`. Fair.

If you go with `&'a x`, you can say the lifetime of the referent `x` is `'a`, so the reference should have the same lifetime with referent `x`, thus it can be used anywhere in region `'a`. It's `r` can't be used outside `'b`, not `&x`.

To find the answer, [the nomincon][the-nomincon] is the place:

> The borrow checker always tries to **minimize** the extent of a lifetime.

It's also mentioned by [too many linked list][too-many-linked-list]:

> The entire lifetime system is in turn just a **constraint-solving** system that tries to **minimize** the region of every reference.

The right answer to this is `'b` because `&x` is **only used** in region `'b`.  Besides, `'b` is enclosed by `'a`, therefore it's safe to minimize the region of `&x` as `&'b x` by the compiler.

The lifetime of the reference is **not** always same with the lifetime of the variable that holds it.

```rust
{ // 'a
    let x = 6;
    let y;
    { // 'b
        let r = &x;
        y = r;
    }
}
```

In this case, `&x` can be annotated as `&'a x`, even though `r` has lifetime `'b`, the reference that it holds is used outside `'b` by variable `y`. Therefore, the minimal region of `&x` being used is `'b`.

For more examples and explanations, see [the nomincon][the-nomincon].

## Why bother to annotate lifetime for references

### external context

Take a look at this code,

```rust
pub fn larger(x: &i32, y: &i32) -> &i32 {
    if *x > *y {
        x
    } else {
        y
    }
}
```

The lifetime of the output reference is dependent on 2 inputs. Its lifetime can either from `x` and `y`. It's ambiguous.

Compiler could know it by

1. running the actual code at **runtime**. Then it can't detect errors at compile time.
2. recursively analyzing enclosing scope, i.e., trace back to root caller. This would be hugely inefficient when code gets nested and complicated.

That's why, you need to provide extra type annotation in order to allow compiler finish **static** analysis at compile time.

With annotation, it compiles.

```rust
pub fn larger<'a>(x: &'a i32, y: &'a i32) -> &'a i32 {
    if *x > *y {
        x
    } else {
        y
    }
}
```

Notice we don't need lifetime annotation inside of the function body. Because within function body, it's **local** context,

> The compiler has full information and can infer all the constraints to find the minimum lifetimes. However at the type and API-level, the compiler doesn't have all the information. It requires you to tell it about the **relationship** between different lifetimes so it can figure out what you're doing. -- [too many linked list][too-many-linked-list]

That being said, the function body can't be arbitrarily long and it should be light workloads for compiler.  Similar to **memoization** of dynamic programming, functions called within the enclosing function body are annotated with lifetimes. When analyze enclosing function body, compiler doesn't need to step into their function bodies to repeat the process.

### relationships

> One lifetime annotation by itself doesn’t have much meaning, because the annotations are meant to tell Rust how generic lifetime parameters of multiple references relate to each other. -- [the book][the-book]

```rust
pub fn larger<'a>(x: &'a i32, y: &'a i32) -> &'a i32 {
```

The above signature **doesn't** mean `x` and `y` should have the **exactly same** lifetime. It means `x` and `y` should live **as least long as** `'a`, i.e., the smallest valid region of two references.

At compile time, a concrete scope will be substituted for generic parameter `'a`. Compiler needs to work out the smallest region for every reference. As it's mentioned earlier,

> The entire lifetime system is in turn just a constraint-solving system that tries to minimize the region of every reference.

For example, [playground][code-larger],

```rust
pub fn main() {
    {//'b
        let x = 12;
        let rr = &x;

        { // 'c
            let y = 10;
            let zz = &y;
            println!("{}", larger(rr, zz));
        }
    }
}
```

When `larger` is called, it can be desugared as,

```rust
            println!("{}", larger<'a = 'c>(&'b x, &&'c y));
```

The compiler will subtitle generic parameter `'a` with the smaller concrete scope `'c` since it minimize the region for input and output parameters.

Code below will be rejected by compiler.

```rust
pub fn main() {
    {//'b
        let hh;

        let x = 12;
        let rr = &x;

        { // 'c
            let zz = &y;
            let y = 10;

            hh = larger(rr, zz);
        }

        println!("{}", hh);
    }
}
```

Let's wrap everything together. Can code below compile?

```rust
pub fn main() {
    {//'b
        let hh;

        let y = 10;
        let x = 12;

        let rr = &x;
        { // 'c
            let zz = &y;

            // 'a = 'b
            hh = larger(rr, zz);
        }
        println!("{}", hh);
    }
}
```

Yes. The minimal region for `'a` to get everything safely compiled is `'b` at this time.

The lifetime of `hh` is `'b`, so the output should at least be `&'b i32`, otherwise, `hh` can be a dangling pointer. It constrains `zz` to be `&'b i32`, instead of `'c i32`. This constraint can be resolved since the referent `y` has lifetime `'b`.

## lifetime as bounds

`&'a T` deduces `T: 'a`, which means the underlying memory, aliased or referenced by variable of type `T` should be valid for at least at region `'a`.

```rust
let x = 10; // aliased memory, owned type
let y = &x; // referenced memory, reference type
```

However, the reverse is not true since `T` can be owned type. Check [rust misconception][misconception] for further readings.


[too-many-linked-list]: https://rust-unofficial.github.io/too-many-lists/second-iter.html
[the-book]: https://doc.rust-lang.org/book/ch10-03-lifetime-syntax.html#validating-references-with-lifetimes
[the-nomincon]: https://doc.rust-lang.org/nomicon/lifetimes.html
[misconception]: https://github.com/pretzelhammer/rust-blog/blob/master/posts/common-rust-lifetime-misconceptions.md#common-rust-lifetime-misconceptions
[code-larger]: https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=f1b5862b75e15716204e98ad85ed7d1f