---
title: "Build OpenMP with Apple Clang"
subtitle: "In-tree and out-of-tree build in practice with CMake"
author: Guihao Liang
date: 2019-12-18
published: true
bigimg: /img/cute_owl.jpg
tags: ["cmake", "cpp", "openmp"]
---

## Summary

TL;DR, clang in XCode toolchain (I call it __Apple Clang__, run `xcrun -f clang` to locate) doesn't support `-fopenmp` option. In order to use OpenMP, follow this [post][2] and this [PR][3].

```cmake
set(ON_X86
    ON
    CACHE BOOL "build on x86 architecture")

set(LIBOMP_COPY_EXPORTS
    FALSE
    CACHE STRING "build in place")
...

# build openmp from openmp source tree
add_subdirectory(${_openmp_dir})

set(TURI_OMP_H_PATH
    ${CMAKE_CURRENT_BINARY_DIR}/${_openmp_dir}/runtime/src
    CACHE STRING "build time generated omp.h path")

target_include_directories(omp INTERFACE ${TURI_OMP_H_PATH})
```

In short, Apple Clang doesn't support __ARM__ architecture, which is discussed in my [previous posts](19-12-17-build-omp-for-ios.md), and `ON_X86` is set explicitly here. `LIBOMP_COPY_EXPORTS` is set to `FALSE` to prevent OpenMP from moving the architecture dependent and on-the-fly generated `omp.h` to somewhere else but to put it in place. The last line is to explicitly add an include path to the CMake target `omp` set in OpenMP source, in order to provide OpenMP __progma__ directive location to the compiler to use `-fopenmp`, as it is mentioned by [post][2].

To compile with `-fopenmp` with Apple clang,

```cmake
# equivalent to -fopenmp -I/path/to/openmp/include ...
target_compile_options(dummy PRIVATE -fopenmp)
target_link_libraries(dummy PRIVATE omp)
```

## Build structure and process

Recently, I've been working on integrating [OpenMP][0] to [Turicreate](https://github.com/apple/turicreate) codebase. Let me elaborate through the approaches I have tried.

Before digging into details, there's one point I want to clarify. TuriCreate has only one src tree, rooted at `src`, and 2 build trees, rooted at `debug` and `release` respectively. The layout is quite simple:

```bash
turicreate
├── CMakeLists.txt
├── debug
├── deps
├── release
├── src
```

## External Project, out-of-tree build

`deps` contains the source of libraries that are built out-of-source-tree, and those libraries are called as external dependencies.

The overall build process is that we build the external dependencies, and install them into a known location outside of build tree, located at `turicreate/deps/local/{lib,lib64,include}`, where `turicreate` refers to the root of the project and I will use this convention from now on.

After all external dependencies are built, those external libraries are known at compile and link time when we compile the source tree and put all build into the build tree. [ExternalProject_add](https://cmake.org/cmake/help/latest/module/ExternalProject.html) is a great fit for this task!

With everything set up, I get an unexpected error:

```bash
clang: error: unsupported option '-fopenmp'
```

I haven't realized this is a the problem from the clang I'm using. So I proceed to use in-tree build because I suspect there are some extra hidden flags set by __Apple Clang__ used by `xcodebuild`.

## In-source build with xcodebuild

This is the approach actually that gets adopted. For details, check this [PR][3].

The in-tree build is to put the source file under `turicreate/src/external/`. During the build, the OpenMP source file will be treated as part of TuriCreate source and `xcodebuild` will pick up these source to compile with flags that are potentially missing in out-tree build.

```bash
external/openmp
├── CMakeLists.txt
├── crt_externs.h
└── openmp-src
    ├── CMakeLists.txt
    ├── CREDITS.txt
    ├── LICENSE.txt
    ├── README.rst
    ├── cmake
    ├── libomptarget
    ├── runtime
    └── www
```

I copy the source of OpenMP to `turicreate/src/external/openmp/openmp-src` and put a `CMakeLists.txt` file, with customized build configurations, outside of the `openmp-src` at `openmp` directory.

```cmake
set(_openmp_dir openmp-src)

# custom settings
...
set(LIBOMP_COPY_EXPORTS
    FALSE
    CACHE STRING "build in place")


# if <crt_extern.h> is missing
# add <crt_extern.h> manually
include_directories(${CMAKE_CURRENT_SOURCE_DIR})

# build openmp
add_subdirectory(${_openmp_dir})
...
```

The full `CMakeLists.txt` can be found [here](https://github.com/apple/turicreate/blob/be858d5ff30c72a46e3ec635fb5948b921a18681/src/external/openmp/CMakeLists.txt). To pick up target `openmp` into build process, one-liner `add_subdirectory(openmp)` is added into `turicreate/src/external/CMakeLists.txt`.

And, I get the same error in the end.

```bash
clang: error: unsupported option '-fopenmp'
```

Both in- and out-tree build doesn't work. The `-fopenmp` option probably is not supported by the clang I'm using.

## final call of faith

Let's get to the [final call](#summary).

[0]: https://github.com/llvm-mirror/openmp
[2]: https://iscinumpy.gitlab.io/post/omp-on-high-sierra/
[3]: https://github.com/apple/turicreate/pull/3001