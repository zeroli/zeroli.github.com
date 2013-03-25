---
layout: post
title: "STL allocator"
description: "关于STL 中allocator的接口与实现，wiki有比较清楚的定义。SGI STL版本的allocator并没有遵守C++标准。它只提供simple_alloc类共container使用，设计的allocator名字叫做alloc，有二级配置器。"
category: cpp
tags: [STL源码阅读]
---
{% include JB/setup %}

###写在前面
关于STL 中allocator的接口与实现，[wiki](http://en.wikipedia.org/wiki/Allocator_)有比较清楚的定义。

###STL allocator
* SGI allocator  

SGI STL版本的allocator并没有遵守C++标准。它只提供simple_alloc类共container使用，设计的allocator名字叫做alloc，有二级配置器。
第一级配置采用malloc/free来实现allocate/deallocate，第二级配置器采用针对申请的内存有特别处理，如果大小大于128字节, 
则转调用第一级配置来分配和释放内存，否则采用memory pool的方式来分配这些小额的内存。在SGI STL中，默认采用第二级内存分配器。  
具体的可以阅读侯捷的《STL源码剖析》第二章。

在SGI STL 3.3中，已经提供了一个符合C++标准的allocator接口，通过对内部的alloc转调用来实现。
![](/assets/image/1345811764_3494.png)

a)
![](/assets/image/1345811794_3169.png)

b)
![](/assets/image/1345811804_8633.png)

c)
![](/assets/image/1345811810_5017.png)

d)
![](/assets/image/1345811816_1757.png)

e)
![](/assets/image/1345811821_5102.png)

* GCC STL allocator  

GCC C++的STL内部借鉴SGI STL的实现，不过它对外的的STL allocator严格遵守C++的标准  
看一下它的代码：（定义在bits/allocator.h文件中）  

{% highlight cpp linenos %}
namespace std
{
template
class allocator;


template<>
class allocator
{
public:
typedef size_t size_type;
typedef ptrdiff_t difference_type;
typedef void* pointer;
typedef const void* const_pointer;
typedef void value_type;


template
struct rebind
{ typedef allocator other; };
};


/**
* @brief The "standard" allocator, as per [20.4].
*
* (See @link Allocators allocators info @endlink for more.)
*/
template
class allocator: public ___glibcxx_base_allocator
{
public:
typedef size_t size_type;
typedef ptrdiff_t difference_type;
typedef _Tp* pointer;
typedef const _Tp* const_pointer;
typedef _Tp& reference;
typedef const _Tp& const_reference;


typedef _Tp value_type;


template
struct rebind
{ typedef allocator other; };


allocator() throw() { }


allocator(const allocator& a) throw()
: ___glibcxx_base_allocator(a) { }


template
allocator(const allocator&) throw() { }


~allocator() throw() { }


// Inherit everything else.
};


template
inline bool
operator==(const allocator&, const allocator&)
{ return true; }


template
inline bool
operator!=(const allocator&, const allocator&)
{ return false; }


// Inhibit implicit instantiations for required instantiations,
// which are defined via explicit instantiations elsewhere.
// NB: This syntax is a GNU extension.
#if _GLIBCXX_EXTERN_TEMPLATE
extern template class allocator;


extern template class allocator;
#endif


// Undefine.
#undef ___glibcxx_base_allocator
} // namespace std
{% endhighlight %}

template 类`___glibcxx_base_allocator`定义在具体的平台相关的头文件中，例如i386-redhat-linux/bits/c++allocator.h:
{% highlight cpp linenos %}
// Define new_allocator as the base class to std::allocator.
#include <ext/new_allocator.h>
#define ___glibcxx_base_allocator __gnu_cxx::new_allocator
{% endhighlight %}
根据GCC/libstdc++的[DOC](http://gcc.gnu.org/onlinedocs/libstdc++/manual/bk01pt04ch11.html)， 知道
> The current default choice for allocator is \_\_gnu_cxx::new_allocator.  
可以看出GCC c++的allocator其实采用的是new/delete-based allocation.  
{% highlight cpp linenos %}
namespace __gnu_cxx
{
/**
* @brief An allocator that uses global new, as per [20.4].
*
* This is precisely the allocator defined in the C++ Standard.
* - all allocation calls operator new
* - all deallocation calls operator delete
*
* (See @link Allocators allocators info @endlink for more.)
*/
template
class new_allocator
{
public:
typedef size_t size_type;
typedef ptrdiff_t difference_type;
typedef _Tp* pointer;
typedef const _Tp* const_pointer;
typedef _Tp& reference;
typedef const _Tp& const_reference;
typedef _Tp value_type;


template
struct rebind
{ typedef new_allocator other; };


new_allocator() throw() { }


new_allocator(const new_allocator&) throw() { }


template
new_allocator(const new_allocator&) throw() { }


~new_allocator() throw() { }


pointer
address(reference __x) const { return &__x; }


const_pointer
address(const_reference __x) const { return &__x; }


// NB: __n is permitted to be 0. The C++ standard says nothing
// about what the return value is when __n == 0.
pointer
allocate(size_type __n, const void* = 0)
{ return static_cast(::operator new(__n * sizeof(_Tp))); }


// __p is not permitted to be a null pointer.
void
deallocate(pointer __p, size_type)
{ ::operator delete(__p); }


size_type
max_size() const throw()
{ return size_t(-1) / sizeof(_Tp); }


// _GLIBCXX_RESOLVE_LIB_DEFECTS
// 402. wrong new expression in [some_] allocator::construct
void
construct(pointer __p, const _Tp& __val)
{ ::new(__p) _Tp(__val); }


void
destroy(pointer __p) { __p->~_Tp(); }
};


template
inline bool
operator==(const new_allocator&, const new_allocator&)
{ return true; }


template
inline bool
operator!=(const new_allocator&, const new_allocator&)
{ return false; }
} // namespace __gnu_cxx
{% endhighlight %}
不过GNU GCC提供了更多很有意思的allocator定制  
`__gnu_cxx::malloc_allocator(malloc_allocator.h)`  
`__gnu_cxx::bitmap_allocator (bitmap_allocator.h)`  
`__gnu_cxx::pool_allocator(pool_allocator.h)`  
`__gnu_cxx::__mt_alloc(mt_allocator.h)`  
可通过`--enable-libstdcxx-allocator` 选项来开启其它的allocator，或者在定义一个container时，显式的描述模板参数表示用哪一个allocator:
{% highlight cpp linenos %}
#include <ext/malloc_allocator.h>
std::vector<int, __gnu_cxx::malloc_allocator<int> > malloc_vector;
{% endhighlight %}
![](/assets/image/2012061716291867.png)


* P.J. Plauger版本的STL  

MSVC的STL采用了P.J. Plauger的版本：allocator类定义在了include/xmemory文件中（以VS2008为例）:  
它的_Allocate模板函数直接采用全局的operator new来分配内存。  
{% highlight cpp linenos %}
return ((_Ty _FARQ *)::operator new(_Count * sizeof (_Ty)));
//TEMPLATE FUNCTION _Construct
template<typename _T1,
class _T2> inline
void _Construct(_T1 _FARQ *_Ptr, const _T2& _Val)
{ // construct object at _Ptr with value _Val
void _FARQ *_Vptr = _Ptr;
::new (_Vptr) _T1(_Val); 
}

// TEMPLATE FUNCTION _Destroy
template inline
void _Destroy(_Ty _FARQ *_Ptr)
{ // destroy object at _Ptr
_DESTRUCTOR(_Ty, _Ptr);
}

template <> inline
void _Destroy(char _FARQ *)
{ // destroy a char (do nothing)
}

template <> inline
void _Destroy(wchar_t _FARQ *)
{ // destroy a wchar_t (do nothing)
}

// TEMPLATE CLASS _Allocator_base
template
struct _Allocator_base
{ // base class for generic allocators
typedef _Ty value_type;
};


// TEMPLATE CLASS _Allocator_base
template
struct _Allocator_base
{ // base class for generic allocators for const _Ty
typedef _Ty value_type;
};


// TEMPLATE CLASS allocator
template
class allocator
: public _Allocator_base
{ // generic allocator for objects of class _Ty
public:
typedef _Allocator_base _Mybase;
typedef typename _Mybase::value_type value_type;
typedef value_type _FARQ *pointer;
typedef value_type _FARQ& reference;
typedef const value_type _FARQ *const_pointer;
typedef const value_type _FARQ& const_reference;


typedef _SIZT size_type;
typedef _PDFT difference_type;


template
struct rebind
{ // convert an allocator to an allocator
typedef allocator other;
};


pointer address(reference _Val) const
{ // return address of mutable _Val
return (&_Val);
}


const_pointer address(const_reference _Val) const
{ // return address of nonmutable _Val
return (&_Val);
}


allocator() _THROW0()
{ // construct default allocator (do nothing)
}


allocator(const allocator&) _THROW0()
{ // construct by copying (do nothing)
}


template
allocator(const allocator&) _THROW0()
{ // construct from a related allocator (do nothing)
}


template
allocator& operator=(const allocator&)
{ // assign from a related allocator (do nothing)
return (*this);
}


void deallocate(pointer _Ptr, size_type)
{ // deallocate object at _Ptr, ignore size
::operator delete(_Ptr);
}


pointer allocate(size_type _Count)
{ // allocate array of _Count elements
return (_Allocate(_Count, (pointer)0));
}


pointer allocate(size_type _Count, const void _FARQ *)
{ // allocate array of _Count elements, ignore hint
return (allocate(_Count));
}


void construct(pointer _Ptr, const _Ty& _Val)
{ // construct object at _Ptr with value _Val
_Construct(_Ptr, _Val);
}


void destroy(pointer _Ptr)
{ // destroy object at _Ptr
_Destroy(_Ptr);
}


_SIZT max_size() const _THROW0()
{ // estimate maximum array size
_SIZT _Count = (_SIZT)(-1) / sizeof (_Ty);
return (0 < _Count ? _Count : 1);
}
};


// allocator TEMPLATE OPERATORS
template inline
bool operator==(const allocator&, const allocator&) _THROW0()
{ // test for allocator equality (always true)
return (true);
}


template inline
bool operator!=(const allocator&, const allocator&) _THROW0()
{ // test for allocator inequality (always false)
return (false);
}


// CLASS allocator
template<> class _CRTIMP2_PURE allocator
{ // generic allocator for type void
public:
typedef void _Ty;
typedef _Ty _FARQ *pointer;
typedef const _Ty _FARQ *const_pointer;
typedef _Ty value_type;


template
struct rebind
{ // convert an allocator to an allocator
typedef allocator other;
};


allocator() _THROW0()
{ // construct default allocator (do nothing)
}


allocator(const allocator&) _THROW0()
{ // construct by copying (do nothing)
}


template
allocator(const allocator&) _THROW0()
{ // construct from related allocator (do nothing)
}


template
allocator& operator=(const allocator&)
{ // assign from a related allocator (do nothing)
return (*this);
}
};
{% endhighlight %}

