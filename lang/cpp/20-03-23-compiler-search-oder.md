---
title:
subtitle: Compiler header search order
date: 2020-03-26 20:10:49
author: Guihao Liang
published: true
tags: ['python', 'cpp', 'clang']
categories: ['python', 'cpp']
---

Recently, I was using setuptools to build Cython source files for turicreate library, which is a better way of building and linking Cython extentions.

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

The approach is very straightforward. Let setuptools know where to find all the Cython files and it will call `Cythonize` to automatically generate correponding C++ code, and compile them with extra_compile_args. At final stage it will link with `libraries` that are specified.

But something really werid happens to me, when I compile the code that refers to `cmath`, shown below,

```bash
clang -fno-strict-aliasing -I/Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX10.14.sdk/usr/include -DNDEBUG -g -fwrapv -O3 -Wall -Wstrict-prototypes -I/Users/guihaoliang/.pyenv/versions/2.7.16/include/python2.7 -c turicreate/_cython/cy_dataframe.cpp -o build/temp.macosx-10.14-x86_64-2.7/turicreate/_cython/cy_dataframe.o -O3 -DNDEBUG -std=c++11 -w
...
/Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/include/c++/v1/cmath:317:9: error: no
      member named 'isnan' in the global namespace
using ::isnan;
      ~~^

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

It's fairly easy to tell the first error is cause by `::isnan` is defined as a macro but a function, so it cannot use referred as a function with namespace resolution operator `::`. For the second one, it expands `isnan` and recommends you to remove `std::`.

But I used `cmath` many times on my local machine, never ever had this issue before. So I wrote a simple C++ program, `dummy.cc`,

```cpp
#include <cmath>

int main(int argc, char** argv) {
    return !std::isnan(10);
}
```

and compile it with `clang -std=c++11 dummy.cc`, it works without a problem. Therefore I use the full compile flags used above, and it's reproduced. The very suspicious part is the include path, `-I/Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX10.14.sdk/usr/include`. If I remove it, the program compiles again without error.

Then I check the `clang` include order

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

I don't see any `Platform` related headers are searched. Then I added the suspicious search path

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

It's added to the top of the path search list. No wonder it uses the `math.h` defined in that path.


Let's Then I use `-H` in clang to trace the all headers include,

```bash
$ clang -std=c++11 -c dummy.cc -H
. /Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/include/c++/v1/cmath
.. /Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/include/c++/v1/math.h
```

with the wrong header search order,

```bash
$ clang -std=c++11 -c dummy.cc -H -I/Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX10.14.sdk/usr/include
. /Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/include/c++/v1/cmath
.. /Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX10.14.sdk/usr/include/math.h
```

But why it doesn't use `cmath` under `/Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX10.14.sdk/usr/include`. The reason is that path doesn't have `cmath` so it contines to searches from `/Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/include`. And then it restarts its search order, and it searchs from the suspicious path again and then halts with error.

Initially, I thought it's Cython prepends the suspicous path, so I submitted an [issue](https://github.com/cython/cython/issues/3459). It turns out it's the setuptools that uses flags used to compile the python executable.

> Another place where distutils pulls these variables is from the values of sysconfig.get_config_vars('CC', 'CXX', 'OPT', 'BASECFLAGS','CCSHARED', 'LDSHARED'), which are fixed when compiling python.

```
 from distutils import sysconfig
      sysconfig.get_config_vars('CC', 'CXX', 'OPT', 'BASECFLAGS',
                                'CCSHARED', 'LDSHARED', 'SO')
```

So I check checked all the variables in `sysconfig.get_config_vars()`, it shows that,

```bash
'CC': 'clang',
'CXX': 'c++', # which refers to clang++
'CFLAGS': '-fno-strict-aliasing -I/Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX10.14.sdk/usr/include  -DNDEBUG -g -fwrapv -O3 -Wall -Wstrict-prototypes',
'CPPFLAGS': '-I. -IInclude -I./Include -I/usr/local/opt/readline/include -I/usr/local/opt/readline/include -I/usr/local/opt/openssl/include -I/Users/guihaoliang/.pyenv/versions/2.7.16/include',
```

Well, it treats my clang file as C files (even though I specify it as Cpp) and compiles the program using `CC` and `CFLAGS`. No wonder `math.h` doesn't support C++ syntax! What I need to do is to let it treat my Cythonized files as C++ files, and use `CXX` and `CPPFLAGS` instead.
