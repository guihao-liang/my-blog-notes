---
layout: post
title: Install-name-id and install-name-value on Mac
subtitle: "Runtime linking with install-name-id and value"
published: true
bigimg: /img/color_path.jpg
tag: ["cpp"]
categories: ["cpp"]
date: 2020-04-22 18:49:12
---

When you `man install_name_tool`, it doesn't really tell you what's the difference between install-name-id and install-name-value. `id` is used at link time and `value` is used at runtime. They are all information provided for the linker to locate the dylib. The example used below is inspired by this [tutorial][0]. Note that `id` is only present for __dynamic libraries__. As for __executables__, only install name `value` is available.

Let's go through an example to get a concrete understanding.

```bash
$ cat a.cc
#include <iostream>
void a() { std::cout << "a()" << std::endl; }
$ clang++ -c a.cc
$ clang++ -o liba.dylib -dynamiclib a.o
$ otool -L liba.dylib
liba.dylib:
        liba.dylib (compatibility version 0.0.0, current version 0.0.0)
        /usr/lib/libc++.1.dylib (compatibility version 1.0.0, current version 400.9.4)
        /usr/lib/libSystem.B.dylib (compatibility version 1.0.0, current version 1252.250.1)
```

As you can see, the __first__ line is the `id`. Let's link with `libb.dylib`,

```bash
$ cat b.cc
#include <iostream>
void a();
void b() { std::cout << "b()" << std::endl; a(); }
$ clang++ -c b.cc
$ clang++ -o libb.dylib -dynamiclib b.o -L. -la
$ otool -L libb.dylib
libb.dylib:
        libb.dylib (compatibility version 0.0.0, current version 0.0.0)
        liba.dylib (compatibility version 0.0.0, current version 0.0.0)
        /usr/lib/libc++.1.dylib (compatibility version 1.0.0, current version 400.9.4)
        /usr/lib/libSystem.B.dylib (compatibility version 1.0.0, current version 1252.250.1)
```

Just notice the __second__ line, the `id` of `liba.dylib` is used during linking. In fact, lines below the second line are all install-name-values.

Let's change the `id` to `foo/liba.dylib` and link them again. Notice `-id` is used to change the `id` (first line) and `-change` is for `value`.

```shell
$ install_name_tool -id foo/liba.dylib liba.dylib
$ otool -D liba.dylib
liba.dylib:
foo/liba.dylib
liba.dylib:
        foo/liba.dylib (compatibility version 0.0.0, current version 0.0.0)
        /usr/lib/libc++.1.dylib (compatibility version 1.0.0, current version 400.9.4)
        /usr/lib/libSystem.B.dylib (compatibility version 1.0.0, current version 1252.250.1)
```

As you can see, the `-D` and `-L` all outputs the current `id` as __foo/liba.dylib__.

Let's link again with `liba.dylib`,

```bash
$ clang++ -o libb.dylib -dynamiclib b.o -L. -la
$ otool -L libb.dylib
libb.dylib:
        libb.dylib (compatibility version 0.0.0, current version 0.0.0)
        foo/liba.dylib (compatibility version 0.0.0, current version 0.0.0)
        /usr/lib/libc++.1.dylib (compatibility version 1.0.0, current version 400.9.4)
        /usr/lib/libSystem.B.dylib (compatibility version 1.0.0, current version 1252.250.1)
```

See the difference? The run time location to find `liba.dylib` is changed to `foo/liba.dylib` at the __second__ line. This means that `libb.dylib` uses `id` from `liba.dylib` that it links against.

It tells `libb.dylib` where to find `liba.dylib`. `liba.dylib` is located at directory `foo` under the starting directory where the executable is invoked (some people call it working or current directory). Whereas [@loader_path][0] means the directory that contains the library or executable.

This answer is also posted in [SO][1].

[0]: http://matthew-brett.github.io/docosx/mac_runtime_link.html
[1]: https://stackoverflow.com/a/61296548/5335565