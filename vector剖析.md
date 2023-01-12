vector相关代码位于`stl_vector.h`和`vector.cc`中,第二个文件是内部使用的头文件，用户不应该直接包含使用
```C++
/** @file bits/vector.tcc
 *  This is an internal header file, included by other library headers.
 *  Do not attempt to use it directly. @headername{vector}
 */
```
# stl_vector.h
`vector`类继承于`_Vector_base`
## _Vector_base（分析内部数据成员）
是`vector`的基类结构体，首先有声明成员
```C++
  typedef typename __gnu_cxx::__alloc_traits<_Alloc>::template
	rebind<_Tp>::other _Tp_alloc_type;
      typedef typename __gnu_cxx::__alloc_traits<_Tp_alloc_type>::pointer
       	pointer;
```
为什么不直接使用`_Alloc`作为`_Tp_alloc_type`?
```C++
typedef _Alloc _Tp_alloc_type;
 typedef typename __gnu_cxx::__alloc_traits<_Tp_alloc_type>::pointer pointer;
```
可能传入的allocator不是模板类，需要使用rebind获得一个特定类型？(todo)
\
`_Vector_base`定义了两个结构体
### _Vector_impl_data
该结构体存储了三个指针：`_M_start` `_M_finish` `M_end_of_storage`,所以`vector`本质上就是一段连续的内存，使用指针进行管理。
```C++
struct _Vector_impl_data
{
	pointer _M_start;
	pointer _M_finish;
	pointer _M_end_of_storage;
    //...
}
```
其还定义了构造和复制构造函数
```C++
_Vector_impl_data() _GLIBCXX_NOEXCEPT
	: _M_start(), _M_finish(), _M_end_of_storage()
	{ }

#if __cplusplus >= 201103L
	_Vector_impl_data(_Vector_impl_data&& __x) noexcept
	: _M_start(__x._M_start), _M_finish(__x._M_finish),
	  _M_end_of_storage(__x._M_end_of_storage)
	{ __x._M_start = __x._M_finish = __x._M_end_of_storage = pointer(); }
#endif
```
`_M_copy_data`从另一个同类型结构体获得指针
```C++
void
	_M_copy_data(_Vector_impl_data const& __x) _GLIBCXX_NOEXCEPT
	{
	  _M_start = __x._M_start;
	  _M_finish = __x._M_finish;
	  _M_end_of_storage = __x._M_end_of_storage;
	}
```
`_M_swap_data`不使用`swap`，因为可能会影响TBAA（基于类型的别名分析。判断两个名字是否有可能指向同一处内存），从而影响一些优化操作,具体影响机制有待分析。(todo)
```C++
void
_M_swap_data(_Vector_impl_data& __x) _GLIBCXX_NOEXCEPT
{
    // Do not use std::swap(_M_start, __x._M_start), etc as it loses
    // information used by TBAA.
    _Vector_impl_data __tmp;
    __tmp._M_copy_data(*this);
    _M_copy_data(__x);
    __x._M_copy_data(__tmp);
}
```
### _Vector_impl
继承了`_Tp_alloc_type`,`_Vector_impl_data`,定义了一些构造函数，除此之外没有什么特别的。
```C++
struct _Vector_impl
	: public _Tp_alloc_type, public _Vector_impl_data
{
    //...
}
```
## _Vector_base（分析成员函数）
获取allocator
```C++
public:
      typedef _Alloc allocator_type;

      _Tp_alloc_type&
      _M_get_Tp_allocator() _GLIBCXX_NOEXCEPT
      { return this->_M_impl; }

      const _Tp_alloc_type&
      _M_get_Tp_allocator() const _GLIBCXX_NOEXCEPT
      { return this->_M_impl; }

      allocator_type
      get_allocator() const _GLIBCXX_NOEXCEPT
      { return allocator_type(_M_get_Tp_allocator());}
```
接下来定义构造函数(选择一部分进行分析)
```C++
public:
      _Vector_impl _M_impl;
//vector<T>v(10);
_Vector_base(size_t __n)
    : _M_impl()
    { _M_create_storage(__n); }

_Vector_base(_Vector_base&& __x, const allocator_type& __a)
      : _M_impl(__a)
      {
	if (__x.get_allocator() == __a)
	  this->_M_impl._M_swap_data(__x._M_impl);
	else
	  {
	    size_t __n = __x._M_impl._M_finish - __x._M_impl._M_start;
	    _M_create_storage(__n);
	  }
}
```
其中`__x.get_allocator() == __a`判断配置器是否可以共用，即一个配置器开辟的空间是否可由另一个配置器操作，包括销毁。如果可以，那么`this`可以直接接管`__x`的空间，如果不可以，则重新配置内存。默认的配置器是无状态的（可交换，相等）。
（todo)为何下面的函数只是调换了参数顺序，就不做这种判断了？
```C++
_Vector_base(const allocator_type& __a, _Vector_base&& __x)
      : _M_impl(_Tp_alloc_type(__a), std::move(__x._M_impl))
      { }
```
使用`_M_allocate`调用配置器来分配内存,相应地用`_M_deallocate`释放
```C++
pointer
_M_allocate(size_t __n)
{
    typedef __gnu_cxx::__alloc_traits<_Tp_alloc_type> _Tr;
    return __n != 0 ? _Tr::allocate(_M_impl, __n) : pointer();
}
void
_M_deallocate(pointer __p, size_t __n)
{
	typedef __gnu_cxx::__alloc_traits<_Tp_alloc_type> _Tr;
	if (__p)
	  _Tr::deallocate(_M_impl, __p, __n);
}
```
` _M_create_storage`使用上述函数，并设置指针。
```C++
protected:
void
_M_create_storage(size_t __n)
{
	this->_M_impl._M_start = this->_M_allocate(__n);
	this->_M_impl._M_finish = this->_M_impl._M_start;
	this->_M_impl._M_end_of_storage = this->_M_impl._M_start + __n;
}
```
## vector
`vector`继承了`_Vector_base`
```C++
template<typename _Tp, typename _Alloc = std::allocator<_Tp> >
class vector : protected _Vector_base<_Tp, _Alloc>
{
 typedef _Vector_base<_Tp, _Alloc>			_Base;
typedef typename _Base::_Tp_alloc_type		_Tp_alloc_type;
typedef __gnu_cxx::__alloc_traits<_Tp_alloc_type>	_Alloc_traits; 
//...   
}
```
首先关注几个私有函数
```C++
private:
#if __cplusplus >= 201103L
      static constexpr bool
      _S_nothrow_relocate(true_type)
{
      return noexcept(std::__relocate_a(std::declval<pointer>(),
                                    std::declval<pointer>(),
                                    std::declval<pointer>(),
                                    std::declval<_Tp_alloc_type&>()));
}

static constexpr bool
_S_nothrow_relocate(false_type)
{ return false; }

static constexpr bool
_S_use_relocate()
{
// Instantiating std::__relocate_a might cause an error outside the
// immediate context (in __relocate_object_a's noexcept-specifier),
// so only do it if we know the type can be move-inserted into *this.
return _S_nothrow_relocate(__is_move_insertable<_Tp_alloc_type>{});
}
```
`_S_use_relocate()`可用于判断`std::__relocate_a`是否会抛出异常。
`std::__relocate_a`函数用于在给定迭代器范围内，通过`Alloc`的`construct`函数对`Tp`类型进行`移动构造`，从而**将数据从旧的内存地址移动到新的内存地址**。
观察`_S_use_relocate()`函数体，其中使用到了`__is_move_insertable<>`,若参数不满足*MoveIsertable*，则具现化为`false_type`
```
[cppreference.com](https://en.cppreference.com/w/cpp/named_req/MoveInsertable)

MoveInsertable (since C++11)
Specifies that an object of the type can be constructed into uninitialized storage from an rvalue of that type by a given allocator.
```
当不满足这个条件时，使用`std::__relocate_a`执行移动构造可能会出现错误。所以`_S_use_relocate()`返回`false`（即可能会抛出异常）
如果满足MoveInsertable，还需要判断`__relocate_a`内部的调用是否会抛出异常。
```C++
static constexpr bool
_S_nothrow_relocate(true_type)
{
      return noexcept(std::__relocate_a(
      std::declval<pointer>(),
      std::declval<pointer>(),
      std::declval<pointer>(),
      std::declval<_Tp_alloc_type&>()));
}
```
`__relocate_a`的实现位于`stl_uninitialized.h`。待日后分析此头文件时再详细剖析。目前仅需要知道

1. 若`Tp`是`trival`类型，且`Alloc`是默认的`allocator`，则不会抛出异常
2. 否则，需要根据`Alloc`的`construct`是否会抛出异常来决定(若`Alloc`没有实现`construct`，则根据`std::is_nothrow_constructible<_Tp, _Args...>::value`决定是否抛出异常)

\
继续分析
```C++
static pointer
      _S_do_relocate(pointer __first, pointer __last, pointer __result,
		     _Tp_alloc_type& __alloc, true_type) noexcept
      {
	return std::__relocate_a(__first, __last, __result, __alloc);
      }

      static pointer
      _S_do_relocate(pointer, pointer, pointer __result,
		     _Tp_alloc_type&, false_type) noexcept
      { return __result; }

      static pointer
      _S_relocate(pointer __first, pointer __last, pointer __result,
		  _Tp_alloc_type& __alloc) noexcept
      {
	using __do_it = __bool_constant<_S_use_relocate()>;
	return _S_do_relocate(__first, __last, __result, __alloc, __do_it{});
      }
```
`_S_relocate`使用`__bool_constant<_S_use_relocate()>;`决定函数的重载版本。


现在分析部分常见函数
```C++
/**
*  @brief  Creates a %vector with default constructed elements.
*  @param  __n  The number of elements to initially create.
*  @param  __a  An allocator.
*
*  This constructor fills the %vector with @a __n default
*  constructed elements.
*/
explicit
vector(size_type __n, const allocator_type& __a = allocator_type())
: _Base(_S_check_init_len(__n, __a), __a)
{ _M_default_initialize(__n); }
```
可见调用`vector<T>(3)`时会先检查长度是否合适，可以通过分析`_S_check_init_len()`知道`vector`能够容纳多少`T`类型的元素。
```C++
// Called by constructors to check initial size.
static size_type
_S_check_init_len(size_type __n, const allocator_type& __a)
{
	if (__n > _S_max_size(_Tp_alloc_type(__a)))
	  __throw_length_error(
	      __N("cannot create std::vector larger than max_size()"));
	return __n;
}
static size_type
_S_max_size(const _Tp_alloc_type& __a) _GLIBCXX_NOEXCEPT
{
	// std::distance(begin(), end()) cannot be greater than PTRDIFF_MAX,
	// and realistically we can't store more than PTRDIFF_MAX/sizeof(T)
	// (even if std::allocator_traits::max_size says we can).

	const size_t __diffmax
	  = __gnu_cxx::__numeric_traits<ptrdiff_t>::__max / sizeof(_Tp);
	const size_t __allocmax = _Alloc_traits::max_size(__a);
	return (std::min)(__diffmax, __allocmax);
}
```
根据注释,两个指针的差值不能超过`PTRDIFF_MAX`(我的平台上是8字节long的最大值)，在默认配置器中,`max_size`调用是会进行检查的
```C++
    private:
_GLIBCXX_CONSTEXPR size_type
_M_max_size() const _GLIBCXX_USE_NOEXCEPT
{
#if __PTRDIFF_MAX__ < __SIZE_MAX__
	return std::size_t(__PTRDIFF_MAX__) / sizeof(_Tp);
#else
	return std::size_t(-1) / sizeof(_Tp);
#endif
}
```
至于为什么`_S_max_size`需要再检查一遍，猜测是因为考虑到用户实现的配置器可能没有进行检查。
再来看函数体，其中调用了`_M_default_initialize`
```C++
// Called by the vector(n) constructor.
void
_M_default_initialize(size_type __n)
{
    this->_M_impl._M_finish =
    std::__uninitialized_default_n_a(this->_M_impl._M_start, __n,
                    _M_get_Tp_allocator());
}
```
调用了`std::__uninitialized_default_n_a`，其位于`stl_uninitialized.h`，该头文件用于构造对象（并不分配内存），日后对该文件进行分析。(todo)
\
继续看之后的构造函数
```C++
/**
*  @brief  Creates a %vector with copies of an exemplar element.
*  @param  __n  The number of elements to initially create.
*  @param  __value  An element to copy.
*  @param  __a  An allocator.
*
*  This constructor fills the %vector with @a __n copies of @a __value.
*/
vector(size_type __n, const value_type& __value,
    const allocator_type& __a = allocator_type())
: _Base(_S_check_init_len(__n, __a), __a)
{ _M_fill_initialize(__n, __value); }
```
对应`vector<T>(3,T())`，`_M_fill_initialize`最终调用到`__uninit_fill_n`
```C++
 template<typename _ForwardIterator, typename _Size, typename _Tp>
        static _ForwardIterator
        __uninit_fill_n(_ForwardIterator __first, _Size __n,
			const _Tp& __x)
        {
	  _ForwardIterator __cur = __first;
	  __try
	    {
	      for (; __n > 0; --__n, (void) ++__cur)
		std::_Construct(std::__addressof(*__cur), __x);
	      return __cur;
	    }
	  __catch(...)
	    {
	      std::_Destroy(__first, __cur);
	      __throw_exception_again;
	    }
	}
 template<typename _Tp, typename... _Args>
    _GLIBCXX20_CONSTEXPR
    inline void
    _Construct(_Tp* __p, _Args&&... __args)
    {
#if __cplusplus >= 202002L && __has_builtin(__builtin_is_constant_evaluated)
      ::new(static_cast<void*>(__p)) _Tp(std::forward<_Args>(__args)...);
    }	
```
可以观察到使用了`placement new`，在指定的地址上构造对象`new(ptr)Obj(Arg);`
\
分析复制构造函数
```C++
/**
*  @brief  %Vector copy constructor.
*  @param  __x  A %vector of identical element and allocator types.
*
*  All the elements of @a __x are copied, but any unused capacity in
*  @a __x  will not be copied
*  (i.e. capacity() == size() in the new %vector).
*
*  The newly-created %vector uses a copy of the allocator object used
*  by @a __x (unless the allocator traits dictate a different object).
*/
vector(const vector& __x)
: _Base(__x.size(),
_Alloc_traits::_S_select_on_copy(__x._M_get_Tp_allocator()))
{
this->_M_impl._M_finish =
std::__uninitialized_copy_a(__x.begin(), __x.end(),
                this->_M_impl._M_start,
                _M_get_Tp_allocator());
}
```
`_Alloc_traits::_S_select_on_copy(__x._M_get_Tp_allocator())`之前已经分析过，用于在复制容器时决定如何复制配置器。对于默认配置器，直接返回原配置器。

默认移动构造函数
```C++
/**
*  @brief  %Vector move constructor.
*
*  The newly-created %vector contains the exact contents of the
*  moved instance.
*  The contents of the moved instance are a valid, but unspecified
*  %vector.
*/
vector(vector&&) noexcept = default;
```
根据迭代器构建vector，根据迭代器类型，构造函数的复杂度会不同。如果是`forward`,`bidirectianal`或者`random-access`类型的迭代器，那么会调用`N`(`N`取决于迭代器的距离)次元素的构造函数，并且不需要内存重分配(`realloc`)。如果迭代器是`input`类型，那么最多会调用`2N`次元素的构造函数(如何计算todo)，并且伴随`logN`次的内存重分配。
关于迭代器的详细分析，可见`迭代器`章节(todo)
```C++
/**
       *  @brief  Builds a %vector from a range.
       *  @param  __first  An input iterator.
       *  @param  __last  An input iterator.
       *  @param  __a  An allocator.
       *
       *  Create a %vector consisting of copies of the elements from
       *  [first,last).
       *
       *  If the iterators are forward, bidirectional, or
       *  random-access, then this will call the elements' copy
       *  constructor N times (where N is distance(first,last)) and do
       *  no memory reallocation.  But if only input iterators are
       *  used, then this will do at most 2N calls to the copy
       *  constructor, and logN memory reallocations.
       */
#if __cplusplus >= 201103L
template<typename _InputIterator,
	typename = std::_RequireInputIter<_InputIterator>>
vector(_InputIterator __first, _InputIterator __last,
	const allocator_type& __a = allocator_type())
: _Base(__a)
{
	  _M_range_initialize(__first, __last,
			      std::__iterator_category(__first));
}
```
根据迭代器类型`std::__iterator_category(__first)`，`_M_range_initialize`函数会有不同版本被调用。
如果是`InputIterator`，则会使用`for`循环，并且使用`emplace_back`(如果是C++11及之后的版本)，这会造成`realloc`
```C++
// Called by the second initialize_dispatch above
template<typename _InputIterator>
void
_M_range_initialize(_InputIterator __first, _InputIterator __last,
		std::input_iterator_tag)
{
	  __try {
	    for (; __first != __last; ++__first)
#if __cplusplus >= 201103L
	      emplace_back(*__first);
#else
	      push_back(*__first);
#endif
	  } __catch(...) {
	    clear();
	    __throw_exception_again;
	  }
}
```
如果是`ForwardIterator`及其派生类型，则
1. 首先分配足够的内存
2. 逐个构建元素
```C++
// Called by the second initialize_dispatch above
      template<typename _ForwardIterator>
	void
	_M_range_initialize(_ForwardIterator __first, _ForwardIterator __last,
			    std::forward_iterator_tag)
{
	  const size_type __n = std::distance(__first, __last);
	  this->_M_impl._M_start
	    = this->_M_allocate(_S_check_init_len(__n, _M_get_Tp_allocator()));
	  this->_M_impl._M_end_of_storage = this->_M_impl._M_start + __n;
	  this->_M_impl._M_finish =
	    std::__uninitialized_copy_a(__first, __last,
					this->_M_impl._M_start,
					_M_get_Tp_allocator());
}
```
为什么对于`InputIterator`就不能这么做？(todo)
\
`assign(n,v)`调用，若`n>capacity`，则分配内存，若`n>size`，则对`size`内的元素重新`fill`，size外的位置调用构造函数。若`n<=size`，则对`[0,n]`重新构建，`[n+1,size]`的元素执行析构并调整`size`为`n`
```C++
/**
       *  @brief  Assigns a given value to a %vector.
       *  @param  __n  Number of elements to be assigned.
       *  @param  __val  Value to be assigned.
       *
       *  This function fills a %vector with @a __n copies of the given
       *  value.  Note that the assignment completely changes the
       *  %vector and that the resulting %vector's size is the same as
       *  the number of elements assigned.
       */
      void
      assign(size_type __n, const value_type& __val)
      { _M_fill_assign(__n, __val); }
template<typename _Tp, typename _Alloc>
    void
    vector<_Tp, _Alloc>::
    _M_fill_assign(size_t __n, const value_type& __val)
    {
      if (__n > capacity())
	{
	  vector __tmp(__n, __val, _M_get_Tp_allocator());
	  __tmp._M_impl._M_swap_data(this->_M_impl);
	  //与其realloc后将旧元素搬过去，不如直接创建临时vector再交换
	}
      else if (__n > size())
	{
	  std::fill(begin(), end(), __val);
	  const size_type __add = __n - size();
	  _GLIBCXX_ASAN_ANNOTATE_GROW(__add);
	  this->_M_impl._M_finish =
	    std::__uninitialized_fill_n_a(this->_M_impl._M_finish,
					  __add, __val, _M_get_Tp_allocator());
	  _GLIBCXX_ASAN_ANNOTATE_GREW(__add);
	}
      else
        _M_erase_at_end(std::fill_n(this->_M_impl._M_start, __n, __val));
    }
```
返回vector迭代器的函数,利用指针构造出迭代器后返回
```C++
// iterators
      /**
       *  Returns a read/write iterator that points to the first
       *  element in the %vector.  Iteration is done in ordinary
       *  element order.
       */
      iterator
      begin() _GLIBCXX_NOEXCEPT
      { return iterator(this->_M_impl._M_start); }

      /**
       *  Returns a read-only (constant) iterator that points to the
       *  first element in the %vector.  Iteration is done in ordinary
       *  element order.
       */
      const_iterator
      begin() const _GLIBCXX_NOEXCEPT
      { return const_iterator(this->_M_impl._M_start); }

      /**
       *  Returns a read/write iterator that points one past the last
       *  element in the %vector.  Iteration is done in ordinary
       *  element order.
       */
      iterator
      end() _GLIBCXX_NOEXCEPT
      { return iterator(this->_M_impl._M_finish); }

      /**
       *  Returns a read-only (constant) iterator that points one past
       *  the last element in the %vector.  Iteration is done in
       *  ordinary element order.
       */
      const_iterator
      end() const _GLIBCXX_NOEXCEPT
      { return const_iterator(this->_M_impl._M_finish); }

      /**
       *  Returns a read/write reverse iterator that points to the
       *  last element in the %vector.  Iteration is done in reverse
       *  element order.
       */
      reverse_iterator
      rbegin() _GLIBCXX_NOEXCEPT
      { return reverse_iterator(end()); }

      /**
       *  Returns a read-only (constant) reverse iterator that points
       *  to the last element in the %vector.  Iteration is done in
       *  reverse element order.
       */
      const_reverse_iterator
      rbegin() const _GLIBCXX_NOEXCEPT
      { return const_reverse_iterator(end()); }

      /**
       *  Returns a read/write reverse iterator that points to one
       *  before the first element in the %vector.  Iteration is done
       *  in reverse element order.
       */
      reverse_iterator
      rend() _GLIBCXX_NOEXCEPT
      { return reverse_iterator(begin()); }

      /**
       *  Returns a read-only (constant) reverse iterator that points
       *  to one before the first element in the %vector.  Iteration
       *  is done in reverse element order.
       */
      const_reverse_iterator
      rend() const _GLIBCXX_NOEXCEPT
      { return const_reverse_iterator(begin()); }

#if __cplusplus >= 201103L
      /**
       *  Returns a read-only (constant) iterator that points to the
       *  first element in the %vector.  Iteration is done in ordinary
       *  element order.
       */
      const_iterator
      cbegin() const noexcept
      { return const_iterator(this->_M_impl._M_start); }

      /**
       *  Returns a read-only (constant) iterator that points one past
       *  the last element in the %vector.  Iteration is done in
       *  ordinary element order.
       */
      const_iterator
      cend() const noexcept
      { return const_iterator(this->_M_impl._M_finish); }

      /**
       *  Returns a read-only (constant) reverse iterator that points
       *  to the last element in the %vector.  Iteration is done in
       *  reverse element order.
       */
      const_reverse_iterator
      crbegin() const noexcept
      { return const_reverse_iterator(end()); }

      /**
       *  Returns a read-only (constant) reverse iterator that points
       *  to one before the first element in the %vector.  Iteration
       *  is done in reverse element order.
       */
      const_reverse_iterator
      crend() const noexcept
      { return const_reverse_iterator(begin()); }
```
`size()`是将指针直接相减得到的,`max_size()`返回vecotr能够分配最大数量，使用`_S_max_size`进行判断。`capacity()`同样通过指针相减进行计算，`empty()`检查`begin()`和`end()`是否相等。
```C++
size_type
size() const _GLIBCXX_NOEXCEPT
{ return size_type(this->_M_impl._M_finish - this->_M_impl._M_start); }

/**  Returns the size() of the largest possible %vector.  */
size_type
max_size() const _GLIBCXX_NOEXCEPT
{ return _S_max_size(_M_get_Tp_allocator()); }
/**
*  Returns the total number of elements that the %vector can
*  hold before needing to allocate more memory.
*/
size_type
capacity() const _GLIBCXX_NOEXCEPT
{ return size_type(this->_M_impl._M_end_of_storage
	- this->_M_impl._M_start); }

/**
*  Returns true if the %vector is empty.  (Thus begin() would
*  equal end().)
*/
_GLIBCXX_NODISCARD bool
empty() const _GLIBCXX_NOEXCEPT
{ return begin() == end(); }
```
`reserve()`用于调整vector的容量，可以避免频繁的`realloc`，可见`vecotr.tcc`文件查看详情。
```C++
/**
       *  @brief  Attempt to preallocate enough memory for specified number of
       *          elements.
       *  @param  __n  Number of elements required.
       *  @throw  std::length_error  If @a n exceeds @c max_size().
       *
       *  This function attempts to reserve enough memory for the
       *  %vector to hold the specified number of elements.  If the
       *  number requested is more than max_size(), length_error is
       *  thrown.
       *
       *  The advantage of this function is that if optimal code is a
       *  necessity and the user can determine the number of elements
       *  that will be required, the user can reserve the memory in
       *  %advance, and thus prevent a possible reallocation of memory
       *  and copying of %vector data.
       */
      void
      reserve(size_type __n);
```
重载`[]`，使用指针位移去访问数据，这种方式不会检查是否越界，可以使用`at()`检查越界
```C++
// element access
      /**
       *  @brief  Subscript access to the data contained in the %vector.
       *  @param __n The index of the element for which data should be
       *  accessed.
       *  @return  Read/write reference to data.
       *
       *  This operator allows for easy, array-style, data access.
       *  Note that data access with this operator is unchecked and
       *  out_of_range lookups are not defined. (For checked lookups
       *  see at().)
       */
      reference
      operator[](size_type __n) _GLIBCXX_NOEXCEPT
      {
	__glibcxx_requires_subscript(__n);
	return *(this->_M_impl._M_start + __n);
      }

      /**
       *  @brief  Subscript access to the data contained in the %vector.
       *  @param __n The index of the element for which data should be
       *  accessed.
       *  @return  Read-only (constant) reference to data.
       *
       *  This operator allows for easy, array-style, data access.
       *  Note that data access with this operator is unchecked and
       *  out_of_range lookups are not defined. (For checked lookups
       *  see at().)
       */
      const_reference
      operator[](size_type __n) const _GLIBCXX_NOEXCEPT
      {
	__glibcxx_requires_subscript(__n);
	return *(this->_M_impl._M_start + __n);
      }
```
`at()`使用`_M_range_check(__n)`执行检查
```C++
protected:
      /// Safety check used only from at().
      void
      _M_range_check(size_type __n) const
      {
	if (__n >= this->size())
	  __throw_out_of_range_fmt(__N("vector::_M_range_check: __n "
				       "(which is %zu) >= this->size() "
				       "(which is %zu)"),
				   __n, this->size());
      }

    public:
      /**
       *  @brief  Provides access to the data contained in the %vector.
       *  @param __n The index of the element for which data should be
       *  accessed.
       *  @return  Read/write reference to data.
       *  @throw  std::out_of_range  If @a __n is an invalid index.
       *
       *  This function provides for safer data access.  The parameter
       *  is first checked that it is in the range of the vector.  The
       *  function throws out_of_range if the check fails.
       */
      reference
      at(size_type __n)
      {
	_M_range_check(__n);
	return (*this)[__n];
      }
/**
       *  @brief  Provides access to the data contained in the %vector.
       *  @param __n The index of the element for which data should be
       *  accessed.
       *  @return  Read-only (constant) reference to data.
       *  @throw  std::out_of_range  If @a __n is an invalid index.
       *
       *  This function provides for safer data access.  The parameter
       *  is first checked that it is in the range of the vector.  The
       *  function throws out_of_range if the check fails.
       */
      const_reference
      at(size_type __n) const
      {
	_M_range_check(__n);
	return (*this)[__n];
      }
```
`push_back()`可以接收左值和右值，当接收右值时，传递给`emplace_back()`。`push_back()`可能会导致扩容，扩容逻辑在`_M_realloc_insert()`中
```C++
/**
       *  @brief  Add data to the end of the %vector.
       *  @param  __x  Data to be added.
       *
       *  This is a typical stack operation.  The function creates an
       *  element at the end of the %vector and assigns the given data
       *  to it.  Due to the nature of a %vector this operation can be
       *  done in constant time if the %vector has preallocated space
       *  available.
       */
      void
      push_back(const value_type& __x)
      {
	if (this->_M_impl._M_finish != this->_M_impl._M_end_of_storage)
	  {
	    _GLIBCXX_ASAN_ANNOTATE_GROW(1);
	    _Alloc_traits::construct(this->_M_impl, this->_M_impl._M_finish,
				     __x);
	    ++this->_M_impl._M_finish;
	    _GLIBCXX_ASAN_ANNOTATE_GREW(1);
	  }
	else
	  _M_realloc_insert(end(), __x);
      }

#if __cplusplus >= 201103L
      void
      push_back(value_type&& __x)
      { emplace_back(std::move(__x)); }
```
观察`_M_realloc_insert`，其定义位于`vector.tcc`，其作用是在vector`[start,finish)`范围的某下标`position`处插入元素时，执行扩容。基本逻辑是

1. 决定扩容后的vector总长度
2. 重新分配内存
3. 在内存上构造对象，首先构造`position`处（即插入的值），然后依次移动构造`position`左部分和右部分的元素。（数据从旧地址搬运到新地址）
4. 析构原内存空间的元素，并释放旧内存
5. 调整vector的指针，指向新内存。
```C++
  template<typename _Tp, typename _Alloc>
    template<typename... _Args>
      void
      vector<_Tp, _Alloc>::
      _M_realloc_insert(iterator __position, _Args&&... __args)
{
      const size_type __len =
//1
	_M_check_len(size_type(1), "vector::_M_realloc_insert");
      pointer __old_start = this->_M_impl._M_start;
      pointer __old_finish = this->_M_impl._M_finish;
      const size_type __elems_before = __position - begin();
//2
      pointer __new_start(this->_M_allocate(__len));
      pointer __new_finish(__new_start);
      __try
	{
	  // The order of the three operations is dictated by the C++11
	  // case, where the moves could alter a new element belonging
	  // to the existing vector.  This is an issue only for callers
	  // taking the element by lvalue ref (see last bullet of C++11
	  // [res.on.arguments]).
	  _Alloc_traits::construct(this->_M_impl,
				   __new_start + __elems_before,
#if __cplusplus >= 201103L
				   std::forward<_Args>(__args)...);
#else
				   __x);
#endif
	  __new_finish = pointer();

#if __cplusplus >= 201103L
//3
	  if _GLIBCXX17_CONSTEXPR (_S_use_relocate())
	    {
	      __new_finish = _S_relocate(__old_start, __position.base(),
					 __new_start, _M_get_Tp_allocator());

	      ++__new_finish;

	      __new_finish = _S_relocate(__position.base(), __old_finish,
					 __new_finish, _M_get_Tp_allocator());
	    }
	  else
#endif
	    {
	      __new_finish
		= std::__uninitialized_move_if_noexcept_a
		(__old_start, __position.base(),
		 __new_start, _M_get_Tp_allocator());

	      ++__new_finish;

	      __new_finish
		= std::__uninitialized_move_if_noexcept_a
		(__position.base(), __old_finish,
		 __new_finish, _M_get_Tp_allocator());
	    }
	}
      __catch(...)
	{
	  if (!__new_finish)
	    _Alloc_traits::destroy(this->_M_impl,
				   __new_start + __elems_before);
	  else
	    std::_Destroy(__new_start, __new_finish, _M_get_Tp_allocator());
	  _M_deallocate(__new_start, __len);
	  __throw_exception_again;
	}
//4
#if __cplusplus >= 201103L
      if _GLIBCXX17_CONSTEXPR (!_S_use_relocate())
#endif
	std::_Destroy(__old_start, __old_finish, _M_get_Tp_allocator());
      _GLIBCXX_ASAN_ANNOTATE_REINIT;
      _M_deallocate(__old_start,
		    this->_M_impl._M_end_of_storage - __old_start);
//5
      this->_M_impl._M_start = __new_start;
      this->_M_impl._M_finish = __new_finish;
      this->_M_impl._M_end_of_storage = __new_start + __len;
}
```
`_M_check_len(n)`表示若vector要增长`n`字节大小，则实际上应该将`vector`扩容到多少字节（如果`n＜size()`，则照原长度的二倍增长)
```C++
// Called by _M_fill_insert, _M_insert_aux etc.
      size_type
      _M_check_len(size_type __n, const char* __s) const
      {
	if (max_size() - size() < __n)
	  __throw_length_error(__N(__s));

	const size_type __len = size() + (std::max)(size(), __n);
	return (__len < size() || __len > max_size()) ? max_size() : __len;
      }
```
因为传入的`__n`其值为1，实际上`__len=2*size()`，**所以`push_back`造成的扩容是按照两倍扩容的**
\
在构造左右部分的对象时，有如下判断语句
```C++
if _GLIBCXX17_CONSTEXPR (_S_use_relocate())
```
`_S_use_relocate()`用于判断`_S_relocate()`类型是否能执行，否则使用`std::__uninitialized_move_if_noexcept_a`，此模板函数定义位于`stl_intialized.h`，若构造过程中出现异常，则会首先析构以及构造的对象，再抛出异常。
\
注意到整个构造过程处于`__try`代码块中，在`__catch`中执行析构。有四种情况

1. 没有任何元素构建成功。此时`__new_finish = pointer();`不会执行，因为在构建`position`处的对象时就抛出异常了，此时`__new_finish!=0`,执行else块，但因为`__new_finish==__new_star`，所以最终没有执行析构
2. 只构建了`position`,意味着在构建左部分时抛出了异常，此时`__new_finish = pointer();`已经执行，所以执行if代码块，析构`postion`处即可。
3. 只构建了`postion`和左部分，此时`__new_finish`已经不等于0，所以执行else代码块，因为右部分在抛出异常前已经析构，所以仅需要析构`[__new_start,__new_finish)`，即`position`及其左部分
4. 全部构建，此时不会抛出异常，不需要catch处理。

继续观察逻辑，在构建成功后，需要对旧地址元素执行析构，因为`_S_relocate`会执行这一步，为了提高效率避免重复析构，需要进行判断
```C++
//4
#if __cplusplus >= 201103L
      if _GLIBCXX17_CONSTEXPR (!_S_use_relocate())
#endif
	std::_Destroy(__old_start, __old_finish, _M_get_Tp_allocator());
      _GLIBCXX_ASAN_ANNOTATE_REINIT;
```
这之后就是释放旧内存空间，调整vector的指针。

