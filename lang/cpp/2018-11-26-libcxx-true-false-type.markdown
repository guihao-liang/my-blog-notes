---
layout: post
title: "libcxx true/false type"
date:	2018-11-26 14:34:35 -08000
categoires: lang cpp
---

```cpp
template <class _Tp, _Tp __v>
struct _LIBCPP_TEMPLATE_VIS integral_constant
{
    static _LIBCPP_CONSTEXPR const _Tp      value = __v;
    typedef _Tp               value_type;
    typedef integral_constant type;
    _LIBCPP_INLINE_VISIBILITY
        _LIBCPP_CONSTEXPR operator value_type() const _NOEXCEPT {return value;}
#if _LIBCPP_STD_VER > 11
    _LIBCPP_INLINE_VISIBILITY
         constexpr value_type operator ()() const _NOEXCEPT {return value;}
#endif
};

template <class _Tp, _Tp __v>
_LIBCPP_CONSTEXPR const _Tp integral_constant<_Tp, __v>::value;

#if _LIBCPP_STD_VER > 14
template <bool __b>
using bool_constant = integral_constant<bool, __b>;
#define _LIBCPP_BOOL_CONSTANT(__b) bool_constant<(__b)>
#else
#define _LIBCPP_BOOL_CONSTANT(__b) integral_constant<bool,(__b)>
#endif

typedef _LIBCPP_BOOL_CONSTANT(true)  true_type;
typedef _LIBCPP_BOOL_CONSTANT(false) false_type;<Paste>
```

[libcxx::type_traits][https://github.com/llvm-mirror/libcxx/blob/master/include/type_traits]
