---
title: "Build OpenMP for IOS12"
author: Guihao Liang
published: true
---

In order to build with xcodebuild, we need to have `Turi.xcodeproj`, which can be generated throught `configure` at [Turicreate](0) root directory:

```bash
cd <turicreate_root>
./configure --no-visualization --no-python --no-remotefs --target=iphoneos --arch=arm64 --with-capi --builder=xcode
```

Try to build [TuriCreate](0) with debug mode for ios12, with command

```bash
cd debug
xcodebuild -j 8 -project Turi.xcodeproj/ -target Recommender -arch arm64 only_active_arch=no -quiet
```

Not a long time waiting, we get error message says:

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

Just check the error related [OpenMP source code](https://github.com/llvm-mirror/openmp/blob/56d941a8cede7c0d6aa4dc19e8f0b95de6f97e1b/runtime/src/kmp_platform.h#L34), where the header `<crt_externs.h>` is referenced from.

```cxx
 61 #if KMP_OS_DARWIN
 62 #include <crt_externs.h>
 63 #define environ (*_NSGetEnviron())
 64 #else
 65 extern char **environ;
 66 #endif
```

and `KMP_OS_DARWIN` is defined through

```cxx
#if (defined __APPLE__ && defined __MACH__)
#undef KMP_OS_DARWIN
#define KMP_OS_DARWIN 1
#endif
```

After googling, the fix should be fairly simple. Either copy source file of [<crt_externs.h>](https://opensource.apple.com/source/Libc/Libc-320/include/crt_externs.h.auto.html) or copy the code into OpenMP source code by expanding `#include <crt_externs.h>` to its source code since the source is about 5 lines.

But wait for a minute, I'm building binary for ios, and why there's a [compiler pre-defined macro](https://sourceforge.net/p/predef/wiki/OperatingSystems/) `__MACH__`? Shouldn't `__MACH__` refers to the laptop [Machintosh](https://en.wikipedia.org/wiki/Macintosh)? After a little bit research, `__MACH__` stands for the kernel called `machintosh`, which is the kernel that the successor kernel `darwin` builds on top of.

The issue is that it runs the code to determine the compiler architecture before setting up the compiler flags.  What we need to do is set something like `-DCMAKE_C_FLAGS="${CMAKE_C_FLAGS} ${CMAKE_C_DEBUG_FLAGS} -arch CMAKE_OSX_ARCHITECTURE`

When cmake compiles the program to determine the architecture, it does so before setting the flags. For more details, please check my another post about [compiler architecutre flags](19-12-17-clang-compiler-arch-flag.md).

```bash
[ 97%] Building C object runtime/src/CMakeFiles/omp.dir/z_Linux_asm.S.o
/Users/guihaoliang/Work/guicreate-2/deps/build/libomp/src/ex_libomp/runtime/src/z_Linux_asm.S:1546:5: error: unknown directive
error: unknown directive
    .size __kmp_unnamed_critical_addr,8
```

```asm
1527 #if KMP_ARCH_ARM || KMP_ARCH_MIPS
1528     .data
1529     .comm .gomp_critical_user_,32,8
1530     .data
1531     .align 4
1532     .global __kmp_unnamed_critical_addr
1533 __kmp_unnamed_critical_addr:
1534     .4byte .gomp_critical_user_
1535     .size __kmp_unnamed_critical_addr,4
1536 #endif /* KMP_ARCH_ARM */
1537
1538 #if KMP_ARCH_PPC64 || KMP_ARCH_AARCH64 || KMP_ARCH_MIPS64
1539     .data
1540     .comm .gomp_critical_user_,32,8
1541     .data
1542     .align 8
1543     .global __kmp_unnamed_critical_addr
1544 __kmp_unnamed_critical_addr:
1545     .8byte .gomp_critical_user_
1546     .size __kmp_unnamed_critical_addr,8
1547 #endif /* KMP_ARCH_PPC64 || KMP_ARCH_AARCH64 */
```

CC=/Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/bin/clang

```bash
/Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/bin/clang   -fPIC  -arch armv7 -isysroot /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/Developer/SDKs/iPhoneOS12.4.sdk   -o CMakeFiles/cmTC_066c2.dir/testCCompiler.c.o   -c /Users/guihaoliang/Work/guicreate-2/deps/build/libomp/src/ex_libomp/CMakeFiles/CMakeTmp/testCCompiler.c
    clang: error: invalid iOS deployment version 'IPHONEOS_DEPLOYMENT_TARGET=12', iOS 10 is the maximum deployment target for 32-bit targets [-Winvalid-ios-deployment-target]
    make[2]: *** [CMakeFiles/cmTC_066c2.dir/testCCompiler.c.o] Error 1
    make[1]: *** [cmTC_066c2/fast] Error 2
```

[0]: https://github.com/apple/turicreate