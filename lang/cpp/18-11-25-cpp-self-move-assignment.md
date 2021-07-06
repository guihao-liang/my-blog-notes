---
layout: post
title:  "CPP Self Move Assignment"
subtitle: "Understand move assignment in depth"
categories: ["cpp"]
date:   2018-11-25 15:22:02 -0800
---

## brief intro

Let me first introduce the context for this problem. Given a list of words, if one word starts with another word, remove it from the list. The solution below is very straightforward:

1. sort the list of word using alphabetic order
2. compare words pointed by two pointers: the first one points to the last word conforms to requirement; the other one just iterate through the whole list.

```cpp
inline void removeRoots(std::vector<string> &dict)
{
    if (dict.size() <= 1)
        return;

    std::sort(std::begin(dict), std::end(dict));

    auto iprv = dict.begin(), irun = iprv;
    auto iend = dict.end();

    while (irun != iend)
    {
        // if *irun cannot be started with *prv
        if (!startsWith(*iprv, *irun))
        {
            *++iprv = std::move(*irun);
        }
        ++irun;
    }

    dict.erase(++iprv, iend);

    for (auto word : dict)
    {
        std::cout << word << std::endl;
    }
}

```

Therefore, given input `[bat, cat]`, the expected output should be `[bat, cat]`. But the output of this snippet is `[bat,]`. The second element is empty? How come?

When `irun` points to `cat`, the `iprv` should point to the first element `bat`.

```cpp
*++iprv = std::move(*irun);
```

This code will increment `iprv` first and then assign the value of what's pointed by `irun`. In our case, it should be equal to

```cpp
*irun = std::move(*irun);
```

Just a simple move assignment, where an element move assigns to itself. It should take no effect, shouldn't it? Recall when we write a move assign, we should first check wether it moves to itself.

```cpp
foo& operator=(foo&& other) // noexcept
{
    if (this != &other)
    {
        /* write your code here */
    }
}
```

This is commonly called **identity check**. _I will keep metioning this concept so you'd better take whatever I say here, even it's something I made up._

And I checked the cpp documenation for basic\_string type:

> basic\_string& operator=( basic\_string&& str ) noexcept(/*see below*/); (2) (since C++11)
> 2) Replaces the contents with those of str using move semantics. Leaves str in valid, but unspecified state. If *this and str are the same object, the function has no effect.

Hard to believe, right?

So I wrote this simple program to verify it!

```cpp
std::string a{"foo"};
std::cout << a << std::endl;
std::cout << (a = std::move(a)) << std::endl;
```

I compiled with clang with `-std=c++11`:

```bash
$ clang --version
Apple LLVM version 8.0.0 (clang-800.0.38)
Target: x86_64-apple-darwin16.7.0
```

The output is an empty string! What!? Should I trust documenation anymore?

```bash
foo

```

---

## libstdc++ implementation

TL;DR: no identity check. You are welcome to skip this section, which does no good to your brain. :-)

Let me first check the [gcc basic_string implementation][1]

```cpp
#if __cplusplus >= 201103L
      basic_string&
      operator=(basic_string&& __str)
      noexcept(_Alloc_traits::_S_nothrow_move())
      {
 if (!_M_is_local() && _Alloc_traits::_S_propagate_on_move_assign()
     && !_Alloc_traits::_S_always_equal()
     && _M_get_allocator() != __str._M_get_allocator())
   {
     // Destroy existing storage before replacing allocator.
     _M_destroy(_M_allocated_capacity);
     _M_data(_M_local_data());
     _M_set_length(0);
   }
 // Replace allocator if POCMA is true.
 std::__alloc_on_move(_M_get_allocator(), __str._M_get_allocator());

 if (__str._M_is_local())
   {
     // We've always got room for a short string, just copy it.
     if (__str.size())
       this->_S_copy(_M_data(), __str._M_data(), __str.size());
     _M_set_length(__str.size());
   }
 else if (_Alloc_traits::_S_propagate_on_move_assign()
     || _Alloc_traits::_S_always_equal()
     || _M_get_allocator() == __str._M_get_allocator())
   {
     // Just move the allocated pointer, our allocator can free it.
     pointer __data = nullptr;
     size_type __capacity;
     if (!_M_is_local())
       {
  if (_Alloc_traits::_S_always_equal())
    {
      // __str can reuse our existing storage.
      __data = _M_data();
      __capacity = _M_allocated_capacity;
    }
  else // __str can't use it, so free it.
    _M_destroy(_M_allocated_capacity);
       }

     _M_data(__str._M_data());
     _M_length(__str.length());
     _M_capacity(__str._M_allocated_capacity);
     if (__data)
       {
  __str._M_data(__data);
  __str._M_capacity(__capacity);
       }
     else
       __str._M_data(__str._M_local_buf);
   }
 else // Need to do a deep copy
   assign(__str);
 __str.clear();
 return *this;
      }
#endif // C++11
```

One thing I should point out, `_Alloc_traits::_S_propagate_on_move_assign()` means whether the `allocator` should be propagated (copied or moved) along with other meta data. That's easy to understand,  different allocator types may have different manner to construct and destruct for the memory. When you take the ownership of the memory, you should take its corresponding allocator to destroy the memory as well.

That's not English, is it? Let's first look at how string is actually stored.

```cpp
      // Use empty-base optimization: http://www.cantrip.org/emptyopt.html
      struct _Alloc_hider : allocator_type // TODO check __is_final
      {
        #if __cplusplus < 201103L
            _Alloc_hider(pointer __dat, const _Alloc& __a = _Alloc())
            : allocator_type(__a), _M_p(__dat) { }
        #else
            _Alloc_hider(pointer __dat, const _Alloc& __a)
            : allocator_type(__a), _M_p(__dat) { }

            _Alloc_hider(pointer __dat, _Alloc&& __a = _Alloc())
            : allocator_type(std::move(__a)), _M_p(__dat) { }
        #endif

            pointer _M_p; // The actual data.
      };

      _Alloc_hider  _M_dataplus;

      enum { _S_local_capacity = 15 / sizeof(_CharT) };

      union
      {
        _CharT           _M_local_buf[_S_local_capacity + 1];
        size_type        _M_allocated_capacity;
      };
```

It uses an **anonymous union** to wrap up a char array (+1 for null terminator) and the integer type to store the current capacity;

Take `basic_string<char>` as an example. If the string size is smaller than __15__, then it will use this union to store the payload into `_M_local_buf` (I guess _M stands for member variable). Beside, the contianer will also use the `_M_dataplus._M_p` to store the pointer to the string payload, which will use either be `_M_local_buf` in stack or allocated memory in heap. If heap memory is in use, then the union will use `_M_allcated_capacity` instead.

Now you can understand what `_M_is_local` does. It compares the `_M_dataplus._M_p` with `_M_local_buf`. If it's same, then it uses local storage on stack, otherwise it uses external heap memory.

Then remaining question is: what the heck is `_S_always_equal` (`_S` means static method)?

```cpp
// <bits/alloc_traits>
    typedef std::allocator_traits<_Alloc>  _Base_type;

// <ext/alloc_traits>
    static constexpr bool _S_always_equal()
    { return _Base_type::is_always_equal::value; }

// <bits/alloc_traits>
    /**
    * @brief   Whether all instances of the allocator type compare equal.
    *
    * @c Alloc::is_always_equal if that type exists,
    * otherwise @c is_empty<Alloc>::type
    */
  template<typename _Alloc>
    struct allocator_traits : __allocator_traits_base
    {
        using is_always_equal
            = __detected_or_t<typename is_empty<_Alloc>::type, __equal, _Alloc>;
    };
```

**It means whether the allocator is a `singleton`, which means all of its instances are the same one.**

OMG, I don't want to dig any deeper, stop!

---

## clang implementation

TL;DR: no identity check as well; You are welcome to skip this section, which will indeed hurt your brain. ;-)

Fair, how about clang implementation?

Let me first introduce you what is `false_type` and `true_type`. They are tags wrapping a static const value for false and true, respectively. The tag is ubiquitously used in cpp to guide the template instantiation.

```cpp
#define _LIBCPP_BOOL_CONSTANT(__b) integral_constant<bool,(__b)>

template <class _Tp, _Tp __v>
struct _LIBCPP_TYPE_VIS_ONLY integral_constant
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

typedef _LIBCPP_BOOL_CONSTANT(true)  true_type;
typedef _LIBCPP_BOOL_CONSTANT(false) false_type;
```

Brain has not damaged yet? Let's read more!

```cpp
template <class _CharT, class _Traits, class _Allocator>
inline _LIBCPP_INLINE_VISIBILITY
basic_string<_CharT, _Traits, _Allocator>&
basic_string<_CharT, _Traits, _Allocator>::operator=(basic_string&& __str)
    _NOEXCEPT_(__alloc_traits::propagate_on_container_move_assignment::value &&
               is_nothrow_move_assignable<allocator_type>::value)
{
    __move_assign(__str, integral_constant<bool,
          __alloc_traits::propagate_on_container_move_assignment::value>());
    return *this;
}
```

Now you see where the tag is used.

Keep in mind, `__alloc_traits::propagate_on_container_move_assignment::value` is similar to `_Alloc_traits::_S_propagate_on_move_assign()`.

---

# true_type tag

On my machine, its value is true, that means the allocator should be moved as well (**which also means we are not using `singleton` allocator implementation**).

```cpp
std::cout << std::string::allocator_type::propagate_on_container_move_assignment::value << std::endl;
```

So we should favor all overloaded function with `true_type` tag.

```cpp
template <class _CharT, class _Traits, class _Allocator>
inline _LIBCPP_INLINE_VISIBILITY
void
basic_string<_CharT, _Traits, _Allocator>::__move_assign(basic_string& __str, true_type)
    _NOEXCEPT_(is_nothrow_move_assignable<allocator_type>::value)
{
    clear();
    shrink_to_fit();
    __r_.first() = __str.__r_.first();
    __move_assign_alloc(__str);
    __str.__zero();
}

_LIBCPP_INLINE_VISIBILITY
void
__move_assign_alloc(basic_string& __str)
    _NOEXCEPT_(
        !__alloc_traits::propagate_on_container_move_assignment::value ||
        is_nothrow_move_assignable<allocator_type>::value)
{__move_assign_alloc(__str, integral_constant<bool,
                  __alloc_traits::propagate_on_container_move_assignment::value>());}

_LIBCPP_INLINE_VISIBILITY
void __move_assign_alloc(basic_string& __c, true_type)
    _NOEXCEPT_(is_nothrow_move_assignable<allocator_type>::value)
    {
        __alloc() = _VSTD::move(__c.__alloc());
    }
```

Now, we should have the conclusion, on what the move assignment does:

1. clear the memory
2. revoke memory allocated
3. copy the address of the moved-from object
4. move the allocator object from moved-from object
5. set the moved-from object to zero

No wonder I get an empty string back by move-assigning an variable to itself.

One more thing intrigues me,

```cpp
    _LIBCPP_INLINE_VISIBILITY
    void __zero() _NOEXCEPT
        {
            size_type (&__a)[__n_words] = __r_.first().__r.__words;
            for (unsigned __i = 0; __i < __n_words; ++__i)
                __a[__i] = 0;
        }
```

I happen to know it allocates memory by word, which also is the underlying byte size for `size_type`. It's nasty!

---

# what about false_type tag

Now, we know we are using the singleton implementation for the string that we are going to move to.

```cpp
template <class _CharT, class _Traits, class _Allocator>
inline _LIBCPP_INLINE_VISIBILITY
void
basic_string<_CharT, _Traits, _Allocator>::__move_assign(basic_string& __str, false_type)
{
    if (__alloc() != __str.__alloc())
        assign(__str);
    else
        __move_assign(__str, true_type());
}
```

The first line checks whether we are using the same allocator singleton. If not, copy the underlying memory instead. Neat!

Since we are doing self assignment, the allocator should be the same. The `_alloc()` returns the reference to the allocator singleton. When the allocator is the same, you don't need to copy the allocator. The it should use `true_type()` to guide the template overloading. Here goes the overloaded `__move_assign` method.

```cpp
template <class _CharT, class _Traits, class _Allocator>
inline _LIBCPP_INLINE_VISIBILITY
void
basic_string<_CharT, _Traits, _Allocator>::__move_assign(basic_string& __str, true_type)
    _NOEXCEPT_(is_nothrow_move_assignable<allocator_type>::value)
{
    clear();
    shrink_to_fit();
    __r_.first() = __str.__r_.first();
    __move_assign_alloc(__str);
    __str.__zero();
}
```

Aha, we visited this, familiar? Without any identity check! Boom! No wonder I get an empty string back!

One thing is suspicious, what does `__move_assign_alloc` do?

```cpp
_LIBCPP_INLINE_VISIBILITY
void
__move_assign_alloc(basic_string& __str)
    _NOEXCEPT_(
        !__alloc_traits::propagate_on_container_move_assignment::value ||
        is_nothrow_move_assignable<allocator_type>::value)
{__move_assign_alloc(__str, integral_constant<bool,
                  __alloc_traits::propagate_on_container_move_assignment::value>());}

_LIBCPP_INLINE_VISIBILITY
void __move_assign_alloc(basic_string&, false_type)
    _NOEXCEPT
    {}
```

It does nothing, as expected since we are not going to modify our allocator singleton.

---

## Conclusion

You should read this section at the very beginning. Sorry, I should have mentioned earlier.

The rule of thumb is that you shall avoid doing any self move assignment like below:

```cpp
std::string bar {"bar"};
bar = std::move(bar);
```

The reason that standard implementation doesn't do identity check is because it may harm the performance. When you do move, 99.9999% will happen between two different instance to transfer ownership of underlying resources. There's no good reason to waste CPU time to do the check.

Therefore, you should do the identity check by your own if the `move-to-object` has chance of being identical to the `moved-from-object`.

Back to our problem at the beginning of this post, let's add an extra identity check to it:

```cpp
while (irun != iend)
{
    if (!startsWith(*iprv, *irun) && ++iprv != irun)
    {
        *iprv = std::move(*irun);
    }
    // cout << "***" << *irun << endl;
    ++irun;
}
```

Solved! Nasty!

[1]: https://github.com/gcc-mirror/gcc/blob/master/libstdc%2B%2B-v3/include/bits/basic_string.h
