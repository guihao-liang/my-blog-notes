---
title: "How to detect compiler architecture when compiling?"
date: 2019-12-17 09:52:25
author: Guihao Liang
tags: ['cmake', 'cpp', 'openmp']
categories: ['cmake', 'cpp']
---

## Conditional compilation for Hardware/OS architectures

Recently, I work on introducing OpenMP to our [TuriCreate](https://github.com/apple/turicreate) codebase for performance experiments.

When I read the [OpenMP][1] source code, I feel really confused about how it can detect the architecture of the host machine. It turns out that special compiler flags need to be set in order to tell the source code what's the architecture to build for by using preprocessor flags (conditional compilation, in short). When compiler compiles source, the source can check these preprocessor flags to determine which part should be compiled for a certain architecture.

For example, in the preprocessor section of `runtime/src/kmp_platform.h` from [OpenMP][1]:

```c
#if KMP_OS_UNIX
#if defined __x86_64
#undef KMP_ARCH_X86_64
#define KMP_ARCH_X86_64 1
#elif defined __i386
#undef KMP_ARCH_X86
#define KMP_ARCH_X86 1
#elif defined __powerpc64__
#if defined __LITTLE_ENDIAN__
#undef KMP_ARCH_PPC64_LE
#define KMP_ARCH_PPC64_LE 1
#else
#undef KMP_ARCH_PPC64_BE
#define KMP_ARCH_PPC64_BE 1
#endif
#elif defined __aarch64__
#undef KMP_ARCH_AARCH64
#define KMP_ARCH_AARCH64 1
...
```

It first detects OS type, which is a UNIX type, and then it checks compiler architecture preprocessor flags, such as `__i386` or `__aarch64__`, to set up its __conditional compilation flags__ in this header file, in preparation for later conditional compilation on source files.

## Digression to KMP

Why do all of those conditional flags start with `KMP`, instead of `OMP`? Besides that, this implementation is contributed by Intel, and why those flags don't start with an I?

feel free to skip ;-)

I googled for a long time and got no good answer. Therefore, I asked a [question in StackOverflow](https://stackoverflow.com/questions/59333281/what-does-k-in-kmp-affinity-mean). It turns out that it doesn't stand for [Kernel](https://superuser.com/questions/1087714/why-do-process-names-start-with-the-letter-k-in-linux), which is very common in the Linux world. But __K__ stands for [Kuck](https://en.wikipedia.org/wiki/David_Kuck),

> who founded Kuck and Associates (KAI) in 1979 to build a line of industry-standard optimizing compilers especially focused upon exploiting parallelism. Later,

KAI is acquired by Intel that's why even though this is Intel's implementation but it doesn't start with an __I__ for Intel.

## Back to the point

This preprocessor detection trick can be also used by CMake build scripts. This [cmake snippet][0] that is used by [OpenMP][0] to detect compiler architecture:

```cmake
function(libomp_get_architecture return_arch)
  set(detect_arch_src_txt "
    #if defined(__KNC__)
      #error ARCHITECTURE=mic
    #elif defined(__amd64__) || defined(__amd64) || defined(__x86_64__) || defined(__x86_64) || defined(_M_X64) || defined(_M_AMD64)
      #error ARCHITECTURE=x86_64
    #elif defined(__i386) || defined(__i386__) || defined(__IA32__) || defined(_M_I86) || defined(_M_IX86) || defined(__X86__) || defined(_X86_)
      #error ARCHITECTURE=i386

    ...
    ")

     # Write out ${detect_arch_src_txt} to a file within the cmake/ subdirectory
    file(WRITE "${CMAKE_CURRENT_BINARY_DIR}/libomp_detect_arch.c" ${detect_arch_src_txt})

    # Try to compile using the C Compiler.  It will always error out with an #error directive, so store error output to ${local_architecture}
    try_run(run_dummy compile_dummy "${CMAKE_CURRENT_BINARY_DIR}" "${CMAKE_CURRENT_BINARY_DIR}/libomp_detect_arch.c" COMPILE_OUTPUT_VARIABLE local_architecture)

    # Match the important architecture line and store only that matching string in ${local_architecture}
    string(REGEX MATCH "ARCHITECTURE=([a-zA-Z0-9_]+)" local_architecture "${local_architecture}")

    # Get rid of the ARCHITECTURE= part of the string
    string(REPLACE "ARCHITECTURE=" "" local_architecture "${local_architecture}")

    # set the return value to the architecture detected (e.g., 32e, 32, arm, ppc64, etc.)
    set(${return_arch} "${local_architecture}" PARENT_SCOPE)

    # Remove ${detect_arch_src_txt} from cmake/ subdirectory
    file(REMOVE "${CMAKE_CURRENT_BINARY_DIR}/libomp_detect_arch.c")

endfunction()
```

It first defines preprocessor flags to detect the compiler architecture. It uses `#error` to print out results to the compile output (stderr) during compiling the code.

Then it uses [try_run](https://cmake.org/cmake/help/v3.2/command/try_run.html) to compile with the preprocessor code. When compiling, all previous defined `CMAKE_C_FLAGS` and `CMAKE_CXX_FLAGS` will be used. One can specify compiler architecture through these flags. We will cover that soon. After the code is compiled, it will do regex match over the build output to capture the keywords which `#error` shouts out. See, it's not that hard.

How do we compile the source code for an architecture that's different from our host machine for __cross compilation__? Let's check `clang` man page.

```bash
-arch <architecture>
          Specify the architecture to build for.
```

`-arch` is the right option for us!

In order to demo the [cmake code][2] mentioned above, I copy the preprocess code defined for CMake variable `detect_arch_src_txt` to a separate C source file, called `libomp_detect_arch.c`.

Let's try to compile it with different architectures, such as [arm, armv7 and arm64](https://stackoverflow.com/questions/21422447/what-iphone-devices-will-run-on-armv7s-and-arm64).

```bash
➜  omp git:(master) ✗ clang -arch arm libomp_detect_arch.c
libomp_detect_arch.c:14:2: error: ARCHITECTURE=arm
#error ARCHITECTURE=arm
 ^
1 error generated.
➜  omp git:(master) ✗ clang -arch armv7 libomp_detect_arch.c
libomp_detect_arch.c:8:2: error: ARCHITECTURE=arm
#error ARCHITECTURE=arm
 ^
1 error generated.
➜  omp git:(master) ✗ clang -arch arm64 libomp_detect_arch.c
libomp_detect_arch.c:22:2: error: ARCHITECTURE=aarch64
#error ARCHITECTURE=aarch64
 ^
1 error generated.
```

Hope this post is helpful ;-)

[0]: https://github.com/llvm/llvm-project/blob/0b969fa9ccf595abc31942e5d14be784707e960c/openmp/runtime/cmake/LibompGetArchitecture.cmake#L16
[1]: https://github.com/apple/turicreate
[2]: https://github.com/llvm/llvm-project/blob/0b969fa9ccf595abc31942e5d14be784707e960c/openmp/runtime/cmake/LibompGetArchitecture.cmake#L18-L52
