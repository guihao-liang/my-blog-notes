---
layout: post
title: "libcxx allocator implementation"
date:	2018-11-27 14:34:35 -08000
categoires: lang cpp
---

# namespace

Before we go any deeper, you may need to know `_VSTD`, which is used everywhere in libcxx.

```cpp
#define _LIBCPP_NAMESPACE _LIBCPP_CONCAT(__,_LIBCPP_ABI_VERSION)
#define _VSTD std::_LIBCPP_NAMESPACE
```

`_VSTD` is just a tag to wrap std with ABI version.

```
#define _LIBCPP_BEGIN_NAMESPACE_STD namespace std {inline namespace _LIBCPP_NAMESPACE {
#define _LIBCPP_END_NAMESPACE_STD  } }
```

These two symbols are used everywhere as well to wrap the code section into `_VSTD` namespace. The reason for this macro, IMO, is that developers don't want adding extra indentations, which will be introduced by adding namespaces and braces. Many text editors have auto indent feature, and for cpp language, the indentation depth is determined by the number of openning braces.

---

# default allocator is always equal

As it is documented [here][1], since default allocator is stateless, two default allocators are always equal (because they use the same handler for deallocate and allocate). This is an important concept which will is mentioned in my other [post][2].

```cpp
template <class _Tp, class _Up>
inline _LIBCPP_INLINE_VISIBILITY
bool operator==(const allocator<_Tp>&, const allocator<_Up>&) _NOEXCEPT {return true;}

template <class _Tp, class _Up>
inline _LIBCPP_INLINE_VISIBILITY
bool operator!=(const allocator<_Tp>&, const allocator<_Up>&) _NOEXCEPT {return false;}
```

This operator overloading is very tricky. If you want to have a customized allocator, you should take care of these methods; Recall the move assignment implementation mentioned in [post][2], when `propagate_on_container_move_assignment` is `false_type`:

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

as

```cpp
if (__alloc() != __str.__alloc())
```

suggests, if you do assignment between different container objects, the corresponding allocators should be comparable. The rule of thumb here is to use the same allocator type.

Besides, when you design your own allocator, you should be careful about whether to set `propagate_on_container_move_assignment` and `propagate_on_container_copy_assignment` with `true_type`, which will potentially affect the overall performance.

# allocator default definition

```cpp
template <class _Tp>
class _LIBCPP_TEMPLATE_VIS allocator
{
	public:
		typedef size_t            size_type;
		typedef ptrdiff_t         difference_type;
		typedef _Tp*              pointer;
		typedef const _Tp*        const_pointer;
		typedef _Tp&              reference;
		typedef const _Tp&        const_reference;
		typedef _Tp               value_type;

		typedef true_type propagate_on_container_move_assignment;
		typedef true_type is_always_equal;

		template <class _Up> struct rebind {typedef allocator<_Up> other;};

		_LIBCPP_INLINE_VISIBILITY allocator() _NOEXCEPT {}
		template <class _Up> _LIBCPP_INLINE_VISIBILITY allocator(const allocator<_Up>&) _NOEXCEPT {}
		_LIBCPP_INLINE_VISIBILITY pointer address(reference __x) const _NOEXCEPT
		{return _VSTD::addressof(__x);}
		_LIBCPP_INLINE_VISIBILITY const_pointer address(const_reference __x) const _NOEXCEPT
		{return _VSTD::addressof(__x);}
		_LIBCPP_INLINE_VISIBILITY pointer allocate(size_type __n, allocator<void>::const_pointer = 0)
		{
			if (__n > max_size())
				__throw_length_error("allocator<T>::allocate(size_t n)"
						" 'n' exceeds maximum supported size");
			return static_cast<pointer>(_VSTD::__allocate(__n * sizeof(_Tp)));
		}
		_LIBCPP_INLINE_VISIBILITY void deallocate(pointer __p, size_type) _NOEXCEPT
		{_VSTD::__libcpp_deallocate((void*)__p);}
		_LIBCPP_INLINE_VISIBILITY size_type max_size() const _NOEXCEPT
		{return size_type(~0) / sizeof(_Tp);}
#if !defined(_LIBCPP_HAS_NO_RVALUE_REFERENCES) && !defined(_LIBCPP_HAS_NO_VARIADICS)
		template <class _Up, class... _Args>
			_LIBCPP_INLINE_VISIBILITY
			void
			construct(_Up* __p, _Args&&... __args)
			{
				::new((void*)__p) _Up(_VSTD::forward<_Args>(__args)...);
			}
#else  // !defined(_LIBCPP_HAS_NO_RVALUE_REFERENCES) && !defined(_LIBCPP_HAS_NO_VARIADICS)
		_LIBCPP_INLINE_VISIBILITY
			void
			construct(pointer __p)
			{
				::new((void*)__p) _Tp();
			}
# if defined(_LIBCPP_HAS_NO_RVALUE_REFERENCES)

		template <class _A0>
			_LIBCPP_INLINE_VISIBILITY
			void
			construct(pointer __p, _A0& __a0)
			{
				::new((void*)__p) _Tp(__a0);
			}
		template <class _A0>
			_LIBCPP_INLINE_VISIBILITY
			void
			construct(pointer __p, const _A0& __a0)
			{
				::new((void*)__p) _Tp(__a0);
			}
# endif  // defined(_LIBCPP_HAS_NO_RVALUE_REFERENCES)
		template <class _A0, class _A1>
			_LIBCPP_INLINE_VISIBILITY
			void
			construct(pointer __p, _A0& __a0, _A1& __a1)
			{
				::new((void*)__p) _Tp(__a0, __a1);
			}
		template <class _A0, class _A1>
			_LIBCPP_INLINE_VISIBILITY
			void
			construct(pointer __p, const _A0& __a0, _A1& __a1)
			{
				::new((void*)__p) _Tp(__a0, __a1);
			}
		template <class _A0, class _A1>
			_LIBCPP_INLINE_VISIBILITY
			void
			construct(pointer __p, _A0& __a0, const _A1& __a1)
			{
				::new((void*)__p) _Tp(__a0, __a1);
			}
		template <class _A0, class _A1>
			_LIBCPP_INLINE_VISIBILITY
			void
			construct(pointer __p, const _A0& __a0, const _A1& __a1)
			{
				::new((void*)__p) _Tp(__a0, __a1);
			}
#endif  // !defined(_LIBCPP_HAS_NO_RVALUE_REFERENCES) && !defined(_LIBCPP_HAS_NO_VARIADICS)
		_LIBCPP_INLINE_VISIBILITY void destroy(pointer __p) {__p->~_Tp();}
};
```

`::new` is prepended with namespace operator because it escapes one level up to namespace `std`. So it uses definition from [new][libcxx::new].

The code for `allocator<_Tp>::construct` is very straightforward. I will explain the following two methods.

---

## get the address of

Let's have a close look at `addressof`:

```cpp
#ifndef _LIBCPP_HAS_NO_BUILTIN_ADDRESSOF

template <class _Tp>
inline _LIBCPP_CONSTEXPR_AFTER_CXX14
_LIBCPP_NO_CFI _LIBCPP_INLINE_VISIBILITY
_Tp*
addressof(_Tp& __x) _NOEXCEPT
{
    return __builtin_addressof(__x);
}

#else

template <class _Tp>
inline _LIBCPP_NO_CFI _LIBCPP_INLINE_VISIBILITY
_Tp*
addressof(_Tp& __x) _NOEXCEPT
{
  return reinterpret_cast<_Tp *>(
      const_cast<char *>(&reinterpret_cast<const volatile char &>(__x)));
}

#endif // _LIBCPP_HAS_NO_BUILTIN_ADDRESSOF
```

As you can see, it just takes the memory address of the variable, regardless of its authentic type (`reinterpret_cast`). But why not use `&__x` directly?

---

## memory allocation new

```cpp
inline _LIBCPP_INLINE_VISIBILITY void *__allocate(size_t __size) {
#ifdef _LIBCPP_HAS_NO_BUILTIN_OPERATOR_NEW_DELETE
  return ::operator new(__size);
#else
  return __builtin_operator_new(__size);
#endif
}
```

You can check the [source][libcxx::new] for more detailed info. I guess `::operator new` is using ABI to acquire fresh chunks of memory. Need to verify it in the future since currently I don't really understand how ABI is invoked. I may have a post to address this topic in the future.

---

# Open questions

Some other stuff I don't really understand is `Automatic Reference Counting`. Just leave it here as a TODO. ;-)

```cpp
#if defined(_LIBCPP_HAS_OBJC_ARC) && !defined(_LIBCPP_PREDEFINED_OBJC_ARC_ADDRESSOF)
// Objective-C++ Automatic Reference Counting uses qualified pointers
// that require special addressof() signatures. When
// _LIBCPP_PREDEFINED_OBJC_ARC_ADDRESSOF is defined, the compiler
// itself is providing these definitions. Otherwise, we provide them.
template <class _Tp>
inline _LIBCPP_INLINE_VISIBILITY
__strong _Tp*
addressof(__strong _Tp& __x) _NOEXCEPT
{
  return &__x;
}

#ifdef _LIBCPP_HAS_OBJC_ARC_WEAK
template <class _Tp>
inline _LIBCPP_INLINE_VISIBILITY
__weak _Tp*
addressof(__weak _Tp& __x) _NOEXCEPT
{
  return &__x;
}
#endif

template <class _Tp>
inline _LIBCPP_INLINE_VISIBILITY
__autoreleasing _Tp*
addressof(__autoreleasing _Tp& __x) _NOEXCEPT
{
  return &__x;
}

template <class _Tp>
inline _LIBCPP_INLINE_VISIBILITY
__unsafe_unretained _Tp*
addressof(__unsafe_unretained _Tp& __x) _NOEXCEPT
{
  return &__x;
}
#endif
```


[1]: [https://en.cppreference.com/w/cpp/memory/allocator/operator_cmp]
[2]: [./2018-11-25-cpp-self-move-assignment.md]
[libcxx::new]: [https://github.com/llvm-mirror/libcxx/blob/master/include/new]
[libcxx::memory]: [https://github.com/llvm-mirror/libcxx/blob/master/include/memory]
[libcxx::sso-allocator]: [https://github.com/llvm-mirror/libcxx/blob/master/include/__sso_allocator]
[libcxx::config]: [https://github.com/llvm-mirror/libcxx/blob/master/include/__config]
