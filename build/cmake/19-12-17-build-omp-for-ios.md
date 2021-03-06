---
title: "Cross Compile OpenMP for IOS"
author: Guihao Liang
date: 2019-12-17 14:15:32
tags: ['cpp', 'clang']
categories: ['compile', 'cpp', 'ios']
---

## Generate XCode project

In order to use `xcodebuild` to build targets for IOS device, we need to provide `.xcodeproj`, which can be generated through `configure` at [Turicreate][0] root directory. Under the hood, the `.xcodeproject` is generated by `CMake` which uses `XCode` as the underlying generator.

```bash
cd <turicreate_root>
./configure --no-visualization --no-python --no-remotefs --target=iphoneos --arch=arm64 --with-capi --builder=xcode
```

Try to build [TuriCreate](0) with debug mode for ios12, with command

```bash
cd debug
xcodebuild -j 8 -project Turi.xcodeproj/ -target Recommender -arch arm64 only_active_arch=no -quiet
```

Not a long time waiting, the error message says:

```bash
cd /Users/guihaoliang/Work/guicreate-2/deps/build/libomp/src/ex_libomp && make
[  5%] Built target libomp-needed-headers
[  8%] Building CXX object runtime/src/CMakeFiles/omp.dir/kmp_alloc.cpp.o
[ 11%] Building CXX object runtime/src/CMakeFiles/omp.dir/kmp_atomic.cpp.o
[ 14%] Building CXX object runtime/src/CMakeFiles/omp.dir/kmp_csupport.cpp.o
[ 17%] Building CXX object runtime/src/CMakeFiles/omp.dir/kmp_debug.cpp.o
[ 20%] Building CXX object runtime/src/CMakeFiles/omp.dir/kmp_itt.cpp.o
[ 23%] Building CXX object runtime/src/CMakeFiles/omp.dir/kmp_environment.cpp.o
/Users/guihaoliang/Work/guicreate-2/deps/build/libomp/src/ex_libomp/runtime/src/kmp_environment.cpp:62:10: fatal error: 'crt_externs.h' file not found
#include <crt_externs.h>
         ^~~~~~~~~~~~~~~
1 error generated.
make[3]: *** [runtime/src/CMakeFiles/omp.dir/kmp_environment.cpp.o] Error 1
make[2]: *** [runtime/src/CMakeFiles/omp.dir/all] Error
```

Having a glance at  the related [OpenMP source code](https://github.com/llvm-mirror/openmp/blob/56d941a8cede7c0d6aa4dc19e8f0b95de6f97e1b/runtime/src/kmp_platform.h#L34),

```cpp
#if KMP_OS_DARWIN
#include <crt_externs.h>
#define environ (*_NSGetEnviron())
#else
extern char **environ;
#endif
```

It needs `<crt_externs.h>` when `KMP_OS_DARWIN` is defined, which means it compiles against `darwin` OS (`uname -s` to get codename for OS).

```cpp
#if (defined __APPLE__ && defined __MACH__)
#undef KMP_OS_DARWIN
#define KMP_OS_DARWIN 1
#endif
```

After googling, the fix should be fairly simple. Either copy source file of [crt_externs.h][cert-h] or inline expanding `#include <crt_externs.h>` to actual implementation code, which is about 5 lines.

But wait for a minute, I'm building binary for ios, and why there's a [compiler pre-defined macro](https://sourceforge.net/p/predef/wiki/OperatingSystems/) `__MACH__`? Shouldn't `__MACH__` refers to the laptop [Machintosh](https://en.wikipedia.org/wiki/Macintosh)? After a little bit research, `__MACH__` stands for the kernel called `machintosh`, which is the kernel that the successor kernel `darwin` builds on top of.

## Build with ARM architecture specified

The issue is that it runs the code to determine the compiler architecture before setting up the conditional compiler flags, in order to later pass preprocessor flags to source code. What wse need to do is to __explicitly__ add compile flags `-arch arm64` to tell compiler (`clang`) which architecture we want to compile for. `arm64` or `aarch64` is used here due to the fact that IOS is using ARM architecture.

For more details, feel free to check my another post about [compiler architecture flags](19-12-17-clang-compiler-arch-flag.md).

After the `arm64` is set, let's compile the project again,

```bash
[ 97%] Building C object runtime/src/CMakeFiles/omp.dir/z_Linux_asm.S.o
/Users/guihaoliang/Work/guicreate-2/deps/build/libomp/src/ex_libomp/runtime/src/z_Linux_asm.S:1546:5: error: unknown directive
error: unknown directive
    .size __kmp_unnamed_critical_addr,8
```

Tracing the error message, I found `asm` code causing trouble below.

```c
#if KMP_ARCH_ARM || KMP_ARCH_MIPS
    .data
    .comm .gomp_critical_user_,32,8
    .data
    .align 4
    .global __kmp_unnamed_critical_addr
__kmp_unnamed_critical_addr:
    .4byte .gomp_critical_user_
    .size __kmp_unnamed_critical_addr,4
#endif /* KMP_ARCH_ARM */

#if KMP_ARCH_PPC64 || KMP_ARCH_AARCH64 || KMP_ARCH_MIPS64
    .data
    .comm .gomp_critical_user_,32,8
    .data
    .align 8
    .global __kmp_unnamed_critical_addr
__kmp_unnamed_critical_addr:
    .8byte .gomp_critical_user_
    .size __kmp_unnamed_critical_addr,8 /** line 1546 **/
#endif /* KMP_ARCH_PPC64 || KMP_ARCH_AARCH64 */
```

Looking at line __1546__, clang already knows we intend to cross compile for __ARM__ but the clang we used here is specifically tailored for `XCode`,

```bash
CC=/Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/bin/clang
```

which doesn't support `.size` instruction.

## Summary

At this stage, I can do nothing but give up the attempt to integrate OpenMP into IOS devices.

IMO, it is pretty reasonable for XCode team to make this decision because IOS computing resources are quite limited and much much less powerful than laptops, and enabling OpenMP probably won't provide much benefit. What's more, OpenMP makes multi-threading much easier and convenient to use, developers may end up abusing multi-threading in their Apps, causing overall IOS slowdowns.

[0]: https://github.com/apple/turicreate
[cert-h]: https://opensource.apple.com/source/Libc/Libc-319/include/crt_externs.h.auto.html
