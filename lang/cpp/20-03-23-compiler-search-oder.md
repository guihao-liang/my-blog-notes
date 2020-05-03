---
title: Compiler header search order
date: 2020-03-26 20:10:49
author: Guihao Liang
published: true
bigimg: /img/rain_drop.jpg
tags: ['python', 'cpp', 'clang']
categories: ['python', 'cpp']
---

## Distinguish C and C++ header search path

Recently, I was using `setuptools` to build `Cython` source files for turicreate library, which is a better way of building and linking Cython extensions.

```python

setup(
    ext_modules=cythonize(
        [
            Extension(
                "*",
                ["turicreate/_cython/*.pyx"],
                include_dirs=include_dirs,
                libraries=["TuriCore"],
                library_dirs=library_dirs,
                extra_compile_args=["-O3", "-DNDEBUG", "-std=c++11", "-w"],
            ),
        ]
    ),
)
```

The approach is very straightforward. Let `setuptools` know where to collect all the Cython source. It will call `Cythonize` to automatically generate corresponding C/C++ code, and compile them with `extra_compile_args`. At final stage, it will link with `libraries` that are specified as keyword arguments.

But something really weird happens to me, when I compile the code that refers to `cmath`, shown below,

```bash
clang -fno-strict-aliasing -I/Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX10.14.sdk/usr/include -DNDEBUG -g -fwrapv -O3 -Wall -Wstrict-prototypes -I/Users/guihaoliang/.pyenv/versions/2.7.16/include/python2.7 -c turicreate/_cython/cy_dataframe.cpp -o build/temp.macosx-10.14-x86_64-2.7/turicreate/_cython/cy_dataframe.o -O3 -DNDEBUG -std=c++11 -w
...
/Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/include/c++/v1/cmath:317:9: error: no
      member named 'isnan' in the global namespace
using ::isnan;
      ~~^
```

It's fairly easy to tell this error is caused by `::isnan` is defined as a macro not a function, so it cannot be referred as a function with namespace resolution operator `::`.

```bash
...

In file included from turicreate/_cython/cy_dataframe.cpp:651:
In file included from /Users/guihaoliang/Work/gui-1-36/targets/include/core/data/flexible_type/flexible_type.hpp:1448:
/Users/guihaoliang/Work/gui-1-36/targets/include/core/data/flexible_type/flexible_type_detail.hpp:906:26: error: expected
      unqualified-id
    flex_float t2 = std::isnan(t) ? NAN : t;
                         ^
/Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX10.14.sdk/usr/include/math.h:165:5: note:
      expanded from macro 'isnan'
    ( sizeof(x) == sizeof(float)  ? __inline_isnanf((float)(x))          \
    ^
18 errors generated.
error: command 'clang' failed with exit status 1
```

For this one, it expands macro `isnan` __inline__.

However, I've been using `cmath` many times on my local machine, never ever had this issue before. So I write a simple C++ program, `dummy.cc`, <a name=dummy-cc></a>

```cpp
#include <cmath>

int main(int argc, char** argv) {
    return !std::isnan(10);
}
```

and compile it with `clang -std=c++11 dummy.cc`, it works without a problem. Then I use the full compile flags used above, and it's reproduced. The very suspicious part is the include path,

```c
clang -I/Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX10.14.sdk/usr/include
```

If I remove it, the program compiles again without error. Something must be related to `clang` include order: <a name="good-search"></a>

```bash
$ clang -Wp,-v -E -
clang -cc1 version 10.0.1 (clang-1001.0.46.4) default target x86_64-apple-darwin18.7.0
#include "..." search starts here:
#include <...> search starts here:
 /usr/local/include
 /Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/lib/clang/10.0.1/include
 /Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/include
 /usr/include
 /System/Library/Frameworks (framework directory)
 /Library/Frameworks (framework directory)
End of search list.
```

I don't see any `Platform` related headers are searched. Then I added the suspicious search path: <a name="bad-search"></a>

```bash
clang -cc1 version 10.0.1 (clang-1001.0.46.4) default target x86_64-apple-darwin18.7.0
#include "..." search starts here:
#include <...> search starts here:
 /Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX10.14.sdk/usr/include
 /usr/local/include
 /Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/lib/clang/10.0.1/include
 /Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/include
 /usr/include
 /System/Library/Frameworks (framework directory)
 /Library/Frameworks (framework directory)
End of search list.
```

It's added to the top of the path search list. No wonder it uses the `math.h` defined in that path which is used by `C` instead of `C++`.

`-H` in `clang` can also be used to trace the all headers include history. Let's compile the [dummy.cc](#dummy-cc) with clang without prepending the `C` header search path.

```bash
$ clang -std=c++11 -c dummy.cc -H
. /Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/include/c++/v1/cmath
.. /Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/include/c++/v1/math.h
```

Recall the [search path](#good-search), `cmath.h` requires `math.h` so the compiler search `math.h` based on its search order. In above example, `math.h` is found in `C++` header path.

Continue with the wrong header search order by prepending a `C` header search path, which contains the `math.h` for pure `C`.

```bash
$ clang -std=c++11 -c dummy.cc -H -I/Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX10.14.sdk/usr/include
. /Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/include/c++/v1/cmath
.. /Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX10.14.sdk/usr/include/math.h
```

Recall from the [search path](#bad-search) discussed above, it first searches `cmath` from:

```bash
/Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX10.14.sdk/usr/include
```

The reason is that above path only contains pure C code and doesn't have `cmath` so that it continues to search from:

```bash
/Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/include
```

Then it restarts the search order, and searches from the suspicious path and halts with errors.

## Distinguish C and C++ for Cython

Initially, I thought it's Cython that prepends the suspicious path, so I submitted an [issue](https://github.com/cython/cython/issues/3459) to Cython team. It turns out that it's the `setuptools` that add that path to compile the python executable on top of `distutils`.

> Another place where distutils pulls these variables is from the values of sysconfig.get_config_vars('CC', 'CXX', 'OPT', 'BASECFLAGS','CCSHARED', 'LDSHARED'), which are fixed when compiling python.

```python
 from distutils import sysconfig
      sysconfig.get_config_vars('CC', 'CXX', 'OPT', 'BASECFLAGS',
                                'CCSHARED', 'LDSHARED', 'SO')
```

After checking all the variables in `sysconfig.get_config_vars()`, it shows:

```bash
'CC': 'clang',
'CXX': 'c++', # compiler driver, which refers to clang++
'CFLAGS': '-fno-strict-aliasing -I/Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX10.14.sdk/usr/include  -DNDEBUG -g -fwrapv -O3 -Wall -Wstrict-prototypes',
'CPPFLAGS': '-I. -IInclude -I./Include -I/usr/local/opt/readline/include -I/usr/local/opt/readline/include -I/usr/local/opt/openssl/include -I/Users/guihaoliang/.pyenv/versions/2.7.16/include',
```

By default, it treats all Cythonized files as __pure C files__ and `CFLAGS` is used, with pure C header path prepended. You may ask how could C++ source be compiled with `CC`? `-std=c++11` can be used with `CC` or `clang` since `c++11` is a __dialect__ of `C` on __language__ level. That's why you can compile C++ files into C objects. The difference of `CC` and `CXX` is that `CC` only links with C standard libraries, whereas `CXX` links both with C and C++ standard libraries, such as `libc++` or `libstdc++`.

No wonder `math.h` in pure C header path doesn't support C++ syntax! What I need to do is to let `setuptools` treat Cythonized files as __C++ files__, and use `CXX` and `CPPFLAGS` to compile instead. In order to do that, I need to pass `language=c++` to [distutils.core.Extension](https://docs.python.org/3.6/distutils/apiref.html#distutils.core.Extension).