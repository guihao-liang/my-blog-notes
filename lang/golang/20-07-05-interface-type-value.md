---
title: "Golang interface value: fat pointer again?"
subtitle: "A brief tour to golang interface value"
author: Guihao Liang
published: true
bigimg: "/img/golang.jpg"
tags: ['golang', 'rust', 'assembly']
---

Recently, I have a project that requires me to learn golang in a short time. IMO, all language with GC is easy to learn and so does golang.

## Comparing golang with Rust

I started my journey with a book, [the go programming language][book], which is well written. I skimmed the book and found golang is really similar to `rust`, in terms of __syntax__, which makes my learning curve super flat. These 2 languages aim to be an alternative for `C` in the domain of system programming.

IMO, they all want to address 2 problems that `C` has with different solutions:

### memory safety

memory leak and memory access safety (dangling pointers, bad access, and etc).

Golang uses GC and thus it has freeze-sweep time as Java does (maybe there's some always-on real-time GC too). Thus golang has a heavy runtime.

Rust uses lifetime compile time analysis to avoid memory safety issues. Therefore, rust has a supper lightweight runtime so that it could be used for embedding system programming, which is dominated by `C`.

All of them aim to free programmers from managing bare pointer and memory on their own hand in order to achieve better efficiency on finishing their tasks.

### safe concurrent programming

Golang follows [CSP](https://en.wikipedia.org/wiki/Communicating_sequential_processes) model, which advocates __no-shared-state__, functional programming style, concurrency programming paradigm, which is tailored for micro-service development.

Rust relies on its memory access model to ensure there's no rase condition during the multi-threading, which is still __shared-state__ concurrent programming model, adopted by most sequential programming languages.

## go interface value

Other than above 2 big differences, there are other obviously differences, such as fantastic pattern matching and generic programming that rust provides, but go doesn't. That's why golang is much simpler. If there's no sense of pointer, golang would be something like Java.

One thing to notice is that Golang and rust are almost the same in advocating __interface-oriented__ programming, unlike `Java`'s or `C++`'s classical class-inheritance style __OOP__. Rust uses `trait` and golang uses `interface`.

cited from the [golang book][book]:

> __Conceptually__, a value of an interface type, or interface value, has 2 components: a concrete type (type descriptor) and a value of that type.

> In general, we cannot know at compile time what the dynamic type of an interface value will be, so a call through an interface must use dynamic dispatch. Instread of direct call, the compiler must generate code to obtain the address of the method from the type descriptor, then make an indirect call to that address.

Sounds familiar? The author only depicts a conceptual model but not the actual implementation. From the above description, it's the same as the [rust fat-pointer][blog-fat-pointer]. Feel free to check my [explanation][blog-fat-pointer] on rust's fat pointer. The __descriptor__ is a pointer to virtual table and the __interface value__ is the pointer to the instance of the concrete type that implements the `interface`. The visualization should be similar to below:

```c
// see more from below asm
// go uses __method to name the vtable
// __object to denote object
[*__method, *__object]
```

Sometimes, people use interface value to denote `__object`, the pointer to the object. To avoid any confusion, I will use `interface object` to denote the fat pointer as a whole.

Let's go through an simple [example][code-example] to verify my observation.

```go
package main

type Foo interface {
    method(x int) int
}

type Bar int

func (b Bar) method(x int) int {
    return x + int(b)
}

type Baz int

func (p *Baz) method(x int) int {
    if p == nil {
        return x
    }

    return x + int(*p)
}

func main() {
    var x Foo
    var y Bar = 10

    // pass by value
    x = y
    x.method(4)

    // pass by pointer
    var z Baz = 10
    x = &z
    x.method(4)
}
```

After it's compiled to assembly code,

```c
// init the interface value to nil
// var x Foo
mov     QWORD PTR [rbp-400], 0
mov     QWORD PTR [rbp-392], 0

// var y Bar = 10
// x = y
mov     edi, OFFSET FLAT:main.Bar..d
call    runtime.newobject         # ptr_tmp = new(10); rax = ptr_tmp
mov     QWORD PTR [rbp-24], rax   # store the value of ptr_tmp to stack
mov     rax, QWORD PTR [rbp-24]   # reload ptr_tmp from stack
mov     rdx, QWORD PTR [rbp-8]    # tmp92, y (y = 10)
mov     QWORD PTR [rax], rdx      # *prt_tmp, tmp92
mov     rax, QWORD PTR [rbp-24]   # reload ptr_tmp from stack
mov     QWORD PTR [rbp-48], OFFSET FLAT:imt..interface.4.main.method.0func.8int.9.8int.9.5..main.Bar      # x.__methods, the vtable
mov     QWORD PTR [rbp-40], rax   # x.__object = ptr_tmp
```

As we can see, golang first allocates a new temp object by calling `runtime.newobject` and copy the value of `y` to it, because interface value (self or `__object`) needs to be a pointer, which has a __fixed__ size on stack.

Then it loads the vtable to `x.__methods`:

```c
// x.method(4)
mov     rax, QWORD PTR [rbp-48]     # _4, x.__methods
mov     rdx, QWORD PTR [rax+8]      # _5, _4->method; offset to method. see vtable below
mov     rax, QWORD PTR [rbp-40]     # _6, x.__object
mov     esi, 4                      # argument value: 4
mov     rdi, rax                    # load object address from _6
call    rdx                         # call method address _5
```

Let's check the vtable for `y` of type `Bar`:

```c
// for normal type: Bar
imt..interface.4.main.method.0func.8int.9.8int.9.5..main.Bar:
        .quad   main.Bar..d
        .quad   main.Bar.method
// for pointer type: &Bar
pimt..interface.4.main.method.0func.8int.9.8int.9.5..main.Bar:
        .quad   type...1main.Bar
        .quad   main.Bar.method
```

As we can see, `golang` automatically generates vtable for `*Bar` type too, which means you can also do `x = &y`.

I bought this [golang book][book] 3 years ago. And it's based on golang __1.5__. By the time I wrote this blog, the latest golang is __1.14__. In early golang versions, passing a reference `&y` to `x` is not allowed since you need to provide implementation for pointer type __explicitly__ or manually. If you don't do so, the code won't pass the compilation because type `*Bar` doesn't __satisfy__ the requirement of the interface `Foo`, where type `Bar` satisfies. How time flies and `golang` is more ergonomic by auto-generating implementation for corresponding pointer and non-pointer type __implicitly__.

From above, we can see that golang explicitly creates a copy of `y` on __heap__, instead of taking its address on stack. The reasoning is that when the receiver is passing by value, it's not a __mutator__ method and therefore it's okay to create a __disposable temporary__ object in heap with a copy of the original value. Therefore, at compile time, the interface object comes with clear size, 2 pointer size. Otherwise, the `__object` part can be with arbitrary size. As professor [Edwards][stephen-A], who taught me PLT in Columbia University, said, the philosophy of __another level of indirection__ is pervasive and most problems can be solved by that in computer engineering, for example, the renowned LLVM.

However, that's __forbidden__ in rust since it doesn't have GC. In rust, you must explicitly create the trait object (fat pointer) from a reference (pointer).

```rust
let y = 10;
// passing by reference explicitly
let x: &Foo = &y;
```

When you pass a pointer to interface object, things are quite simple.

```c
// var z Baz = 10
mov     edi, OFFSET FLAT:main.Baz..d
call    runtime.newobject
mov     QWORD PTR [rax], 10         # rax is the pointer to newly allocated memory
mov     QWORD PTR [rbp-16], rax     # store rax, i.e to z, a pointer to int(10)

// x = &z
mov     QWORD PTR [rbp-48], OFFSET FLAT:pimt..interface.4.main.method.0func.8int.9.8int.9.5..main.Baz     # x.__methods,
mov     rax, QWORD PTR [rbp-16]     # tmp93, z
mov     QWORD PTR [rbp-40], rax     # x.__object, tmp93

// x.method(4)
mov     rax, QWORD PTR [rbp-48]     # _7, x.__methods
mov     rdx, QWORD PTR [rax+8]      # _8, _7->method
mov     rax, QWORD PTR [rbp-40]     # _9, x.__object
mov     esi, 4
mov     rdi, rax                    #, _9
call    rdx                         # _8
```

When passing a pointer to interface object, it passes the address of the object to `x.__object` directly without creating a reference to a temp object. Well, the object is created directly on heap, and that's why there's no extra temp object.

[book]: https://www.gopl.io/
[code-example]: https://godbolt.org/z/Snn_Yt
[stephen-A]: http://www.cs.columbia.edu/~sedwards/
[blog-fat-pointer]: /2020/06/06/fat-pointer