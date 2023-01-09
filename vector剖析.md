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
现在分析构造函数
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
对应`vector<T>(3,T())`，`_M_fill_initialize`的实现如下
```C++
// Called by the first initialize_dispatch above and by the
// vector(n,value,a) constructor.
void
_M_fill_initialize(size_type __n, const value_type& __value)
{
this->_M_impl._M_finish =
    std::__uninitialized_fill_n_a(this->_M_impl._M_start, __n, __value,
                _M_get_Tp_allocator());
}
```
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
``
