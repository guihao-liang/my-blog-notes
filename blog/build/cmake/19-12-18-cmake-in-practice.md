---
title: "Learn CMake in a painful way"
author: Guihao Liang
published: true
---

Recently, I worked on integrating [OpenMP](0) to [Turicreate](https://github.com/apple/turicreate) codebase. Me and my collegue [Hoyt](https://github.com/hoytak) worked on several different ways to try to get the work done.

Let me elaborate through our approaches and explain why we do and what I've learned in a hard way.

## Build structure and process

Before digging into details, there's one point I want to clarify. TuriCreate has only one src tree, rooted at `src`, and 2 build trees, rooted at `debug` and `release` respectively. The layout is quite simple:

```bash
turicreate
├── CMakeLists.txt
├── debug
├── deps
├── release
├── src
```

`deps` contains the source of libraries that we build out-of-source-tree, and we call those libraries external dependencies.

The overall build process is that we first build the external dependencies, and install them into a deterministic location out side of build tree, located at `turicreate/deps/local/{lib,lib64,include}`, where `turicreate` refers to the root of the project and I will use this convention from now on when I mention it in a path.

After all external depencies are built, those external libraries are known at compile and link time when we compile the source tree and put all buildouts into build tree.

## CMake fundamentals

Initially, I thought the job should be fairly easy. Adding a `add_subdirectory()` to OpenMP should be sufficient and I don't have to pick up my long-time-ago CMake knowledge.

The reason not to invest much time on CMake is that wrting build scripts in most cases is one time thing. Once you get it working, you probably won't have a second chance to look at it again as long as it works (not even working well). Maybe after a long time, team decides to upgrade the package, you may come back and do the job again. When you look at the build script again, it looks a like a long distant relative visiting you during Chrismas. Well, It turns out that's the worst decision I have made through this project.

The concept I should've mastered before I read through OpenMP CmakeLists.txt are if-else, macros and variable scopes. I have another post called [CMake 101](19-12-19-cnake-101.md) to cover these topics.

---

## External Project, out-of-tree build

When I follow OpenMP installation instructions to build the library, it turns out the header `omp.h` is both architecture and build mode (debug or release) dependent. That means we don't have the `omp.h` available at compile time and only after OpenMP is built, we can know where the `omp.h` is located in the build tree.

The main concern is that if our source file relies on `omp.h`, the include path at least for now is unclear. What's more, [XGBoost](https://github.com/dmlc/xgboost) uses `#include <omp.h>`.

```cxx
#if defined(_OPENMP) && !defined(DISABLE_OPENMP)
#include <omp.h>
#else
```

If we want to enable XGBoost to use OpenMP, we need to at least add the include path to `omp.h` into cmake's `include_directories` clause into our XGBoost build script.

Since installing the OpenMP to our build tree is more straightforward, we decide to build OpenMP out of the source tree and install the generated header and library to a source-tree build time known location, which is `turicreate/<build_tree>deps/local/lib` and `turicreate/<build_tree>/deps/local/include`.

```CMake
if("${CMAKE_SOURCE_DIR}" STREQUAL "${CMAKE_CURRENT_SOURCE_DIR}")
  message(FATAL_ERROR "Direct configuration not supported, please use parent directory!")
endif()
```

---

## manually build inside of OpenMP source

with standalone and without

```bash
cmake -DCMAKE_C_COMPILER=clang -DCMAKE_CXX_COMPILER=clang++ -DLIBOMP_ARCH=arm -DCMAKE_BUILD_TYPE=Debug ..
```

it uses the architecture passed in (arm is 32-bits architecture):

```bash
[ 97%] Building C object runtime/src/CMakeFiles/omp.dir/z_Linux_asm.S.o
[100%] Linking C shared library libomp.dylib
Undefined symbols for architecture x86_64:
  "_NSGetEnviron()", referenced from:
      ___kmp_env_blk_init in kmp_environment.cpp.o
```

I set the architecture flag to `arm64` and build within the source of OpenMP. Surprisingly, it compiled without an error.

```bash
[ 87%] Building CXX object runtime/src/CMakeFiles/omp.dir/kmp_ftn_cdecl.cpp.o
[ 90%] Building CXX object runtime/src/CMakeFiles/omp.dir/kmp_ftn_extra.cpp.o
[ 93%] Building CXX object runtime/src/CMakeFiles/omp.dir/kmp_version.cpp.o
[ 96%] Building C object runtime/src/CMakeFiles/omp.dir/z_Linux_asm.S.o
[100%] Linking C static library libomp.a
```

But it turns out that it ignores the `LIBOMP_ARCH` flag. We can use libtool

```
```

---

## In-source build with xcodebuild

We suspect that xcode build flags are not passed into clang correctly, thus we want to try the in-tree build strategy. The in-tree build is to put the source file under `<turicreate_root>/src/external/` so that during the build, the OpenMP source file will be treat as part of TuriCreate source and `xcodebuild` will pick up these source to compile.

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

Now we copy the source of OpenMP to `<turicreate_root>/src/external/openmp/openmp-src` and put our own `CMakeLists.txt` file, with customized build configurations, outside of the `openmp-src` at `openmp` directory.

```cmake
set(_openmp_dir openmp-src)

# custom settings
set(OPENMP_STANDALONE_BUILD TRUE CACHE BOOL "not build within llvm source tree")
set(LIBOMP_ENABLE_SHARED OFF CACHE BOOL "open static/dynamic link")
set(LIBOMP_USE_ADAPTIVE_LOCKS ${ON_X86} CACHE BOOL "enable only for x86")
set(OPENMP_ENABLE_LIBOMPTARGET OFF CACHE BOOL "no libomptarget")
set(LIBOMP_COPY_EXPORTS FALSE CACHE STRING "build in place")

# for <crt_extern.h>
include_directories(${CMAKE_CURRENT_SOURCE_DIR})

# build openmp
add_subdirectory(${_openmp_dir})
```

To include `openmp` in our build, we just add one-liner `add_subdirectory(openmp)` into `src/external/CMakeLists.txt`, which is a standarded way of using CMake.

Then we can start the in-tree build. My other post [build omp for ios](19-12-17-build-omp-for-ios.md) covers this topic. Feel free to have a look at.

---

## final call of faith

turn off assembler to avoid compiling the `z_linux.S` by using compiler flags [-integrated-as, -no-integrated-as](https://stackoverflow.com/questions/11118887/how-to-switch-off-llvms-integrated-assembler)

I want to get the exported path to find the header:

retrieve variable from child scope (EXPORT_CMN_DIR)
[get_directory_property](https://stackoverflow.com/questions/34804107/retrieve-variable-from-child-scope#34815419)

---

## real final call

```bash
clang: error: unsupported option '-fopenmp'
```

[0]: https://github.com/llvm-mirror/openmp