# deque

[TOC]

---

## 前言

`deque`的功能很强大, 其复杂度也比`list`, `vector`复杂很多. `deque`是一个`random_access_iterator_tag`类型.  前面分析过`vector`是保存在连续的线性空间, 头插入和删除其代价都很大, 当数组满了还要重新寻找更大的空间; `deque`也是一个保存在连续的线性空间中, 但是它是一个双向开口, 头尾插入和删除都是O(1)的时间复杂度, 空间也是可扩展的, 不会经常寻找新的空间. `deque`的内存操作主要是由`map`来实现的.

打算分为3节来分析`deque`的重要实现部分. 本节分析`deque`的迭代器和构造析构函数.



## __deque_iterator迭代器结构

虽然`deque`在某些方面和`vector`有些相似, 但是迭代器并不是一个普通指针, `deque`的迭代器很复杂, 现在我们就来分析一下.



#### 全局函数

```c++
inline size_t __deque_buf_size(size_t n, size_t sz)
{
  return n != 0 ? n : (sz < 512 ? size_t(512 / sz) : size_t(1));
}
```



#### 类型定义

`deque`是`random_access_iterator_tag`类型. 满足`traits`编程. 

这里重点分析四个参数cur, first, last以及`node`

```c++
#ifndef __STL_NON_TYPE_TMPL_PARAM_BUG
template <class T, class Ref, class Ptr, size_t BufSiz>
struct __deque_iterator {
	// 迭代器定义
  typedef __deque_iterator<T, T&, T*, BufSiz>             iterator;
  typedef __deque_iterator<T, const T&, const T*, BufSiz> const_iterator;
  static size_t buffer_size() {return __deque_buf_size(BufSiz, sizeof(T)); }
#else /* __STL_NON_TYPE_TMPL_PARAM_BUG */
template <class T, class Ref, class Ptr>
struct __deque_iterator {
  typedef __deque_iterator<T, T&, T*>             iterator;
  typedef __deque_iterator<T, const T&, const T*> const_iterator;
  static size_t buffer_size() {return __deque_buf_size(0, sizeof(T)); }
#endif
	// deque是random_access_iterator_tag类型
  typedef random_access_iterator_tag iterator_category;
  // 基本类型的定义, 满足traits编程
  typedef T value_type;
  typedef Ptr pointer;
  typedef Ref reference;
  typedef size_t size_type;
  typedef ptrdiff_t difference_type;
  // node
  typedef T** map_pointer;
  map_pointer node;
	
  typedef __deque_iterator self;
  ...
};
```

```c++
// 满足traits编程
template <class T, class Ref, class Ptr, size_t BufSiz>
inline random_access_iterator_tag
iterator_category(const __deque_iterator<T, Ref, Ptr, BufSiz>&) {
  return random_access_iterator_tag();
}
template <class T, class Ref, class Ptr, size_t BufSiz>
inline T* value_type(const __deque_iterator<T, Ref, Ptr, BufSiz>&) {
  return 0;
}
template <class T, class Ref, class Ptr, size_t BufSiz>
inline ptrdiff_t* distance_type(const __deque_iterator<T, Ref, Ptr, BufSiz>&) {
  return 0;
}
```

`cur`, `first`, `last`这三个变量类似于vector中的3个迭代器一样. 

-   cur : 当前所指的位置
-   first : 当前数组中头的位置
-   last : 当前数组中尾的位置

注意 : 因为deque的空间是由map管理的, 它是一个指向指针的指针, 所以三个参数都是指向当前的数组, 这样的数组可能有多个, 只是每个数组都管理这3个变量.

```c++
template <class T, class Ref, class Ptr, size_t BufSiz>
struct __deque_iterator {
	...
  typedef T value_type;
  T* cur;
  T* first;
  T* last;
  ...
};
```

 每一个`数组`都有一个`node`指针, 他是用来指向`*map`的指针.

```c++
template <class T, class Ref, class Ptr, size_t BufSiz>
struct __deque_iterator {
	...
	// node
  typedef T** map_pointer;
  map_pointer node;
  ...
};
```



#### 构造函数

```c++
template <class T, class Ref, class Ptr, size_t BufSiz>
struct __deque_iterator {
	...
  	// 初始化cur指向当前数组位置, last指针数组的尾, node指向y
  	__deque_iterator(T* x, map_pointer y)  : cur(x), first(*y), last(*y + buffer_size()), node(y) {}
  	// 初始化为一个空的deque
  	__deque_iterator() : cur(0), first(0), last(0), node(0) {}
  	// 接受一个迭代器
  	__deque_iterator(const iterator& x) : cur(x.cur), first(x.first), last(x.last), node(x.node) {}
    ...
};
```



#### 重载

`__deque_iterator`实现了基本运算符, `deque`重载的运算符操作都是调用`__deque_iterator`的运算符.

不过先分析一个待会会用到的函数`set_node`. 

因为node是一个指向`*map`的指针, **当数组填充满了后, 要重新指向下一个数组的头, set_node就是更新指向数组的头的功能.**

```c++
template <class T, class Ref, class Ptr, size_t BufSiz>
struct __deque_iterator {
	...
	void set_node(map_pointer new_node) 
	{
		// 让node指针另一个数组的头, 同时修改头和尾的地址
    	node = new_node;
    	first = *new_node;
    	last = first + difference_type(buffer_size());
  	}
  	...
};
```

**重载++和--**

需要注意++和--都可能出现数组越界, 如果判断要越界就得更新`node`的指向.

```c++
template <class T, class Ref, class Ptr, size_t BufSiz>
struct __deque_iterator {
	...  
 	// 这里需要判断是否达到当前数组的尾部
  self& operator++() {
    ++cur;
    // 达到了尾部就需要更新node的指向
    if (cur == last) {
      set_node(node + 1);
      cur = first;
    }
    return *this; 
  }
  // 同理, 需要判断是否到达数组的头. 到达就要更新node指向
  self& operator--() {
    if (cur == first) {
      set_node(node - 1);
      cur = last;
    }
    --cur;
    return *this;
  }
  
  self operator++(int)  {
    self tmp = *this;
    ++*this;
    return tmp;
  }
  self operator--(int) {
    self tmp = *this;
    --*this;
    return tmp;
  }
  ...
};
```

**重载+, -等**. 因为`deque`是`random_access_iterator_tag`类型, 所以支持直接加减操作.

```c++
template <class T, class Ref, class Ptr, size_t BufSiz>
struct __deque_iterator {
	...
	// 重载指针
  	reference operator*() const { return *cur; }
  	reference operator[](difference_type n) const { return *(*this + n); } // 这个会调用重载+运算符
#ifndef __SGI_STL_NO_ARROW_OPERATOR
  	pointer operator->() const { return &(operator*()); }
#endif /* __SGI_STL_NO_ARROW_OPERATOR */

  	self& operator+=(difference_type n) 
  	{
    	difference_type offset = n + (cur - first);	// 要移动的距离
    	// 如果是在的当前数组内并且向前移动就直接指向那个位置就行了.
    	if (offset >= 0 && offset < difference_type(buffer_size()))
    	  	cur += n;
    	 // 向后移动或已经不再当前数组中
    	else 
    	{
        	// 计算需要跨多少个数组
      		difference_type node_offset = offset > 0 ? offset / difference_type(buffer_size()) : -difference_type((-offset - 1) / buffer_size()) - 1;
      		set_node(node + node_offset);
      		cur = first + (offset - node_offset * difference_type(buffer_size()));
    	}
    	return *this;
  	}
  	// 以下都是调用+运算符
	difference_type operator-(const self& x) const 
	{
    	return difference_type(buffer_size()) * (node - x.node - 1) + (cur - first) + (x.last - x.cur);
  	}
  	self operator+(difference_type n) const {
    	self tmp = *this;
    	return tmp += n;
  	}

  	self& operator-=(difference_type n) { return *this += -n; }
 
  	self operator-(difference_type n) const {
  	  	self tmp = *this;
    	return tmp -= n;
  	}
	// 
  	bool operator==(const self& x) const { return cur == x.cur; }
  	bool operator!=(const self& x) const { return !(*this == x); }
  	bool operator<(const self& x) const 
  	{
	    return (node == x.node) ? (cur < x.cur) : (node < x.node);
  	}
};
```

以上就是`__deque_iterator`的实现, 真的还是挺多的. 其中必须要知道的就是上面介绍的四个参数, 因为`deque`主要就是通过这4个参数来获取元素的.



## deque实际操作

下面要分析了很多关于`deque`的函数, 这里就简单的选择一些函数来写一下, 至少有个印象.

```c++
/*************************************************************************
    > File Name: deque.cpp
    > Author: Function_Dou
    > Mail: NOT
    > Created Time: 2018年11月24日 星期六 12时44分05秒
 ************************************************************************/

#include <iostream>
#include <stdlib.h>
#include <deque>

using namespace std;

int main()
{
	int a[10] = { 1, 2 ,3, 4, 5, 6 };
	deque<int> d;
	d.insert(d.begin(), a, a + 10);
	for(const auto& i : d)
		cout << i << " ";	// 1 2 3 4 5 6 0 0 0 0 
	cout << endl;

	deque<int> d1(d);
	deque<int> d2(a, a+10);
	deque<int> d3(d.begin(), d.end());
	cout << bool(d1==d2) << endl;	// 1
	cout << "size = " << d3.size();	// size = 10

	exit(0);
}
```

接下里的`deque`结构就很复杂, 现在就来看一下吧.



## deque结构

分析了`__deque_iterator`结构后, 再来分析`deque`的基本函数就比较轻松了, `deque`的很多函数都会调用`__deque_iterator`中的操作, 本节也只探讨其基本的构造, 析构, 重载等操作.



#### 基本类型定义

`deque`满足`traits`编程的嵌套定义类型. 

```c++
template <class T, class Alloc = alloc, size_t BufSiz = 0> 
class deque {
public:                         // Basic types
    // 满足traits编程
  typedef T value_type;
  typedef value_type* pointer;
  typedef const value_type* const_pointer;
  typedef value_type& reference;
  typedef const value_type& const_reference;
  typedef size_t size_type;
  typedef ptrdiff_t difference_type;

public:                         // Iterators
    // 定义迭代器
#ifndef __STL_NON_TYPE_TMPL_PARAM_BUG
  typedef __deque_iterator<T, T&, T*, BufSiz>              iterator;
  typedef __deque_iterator<T, const T&, const T&, BufSiz>  const_iterator;
#else /* __STL_NON_TYPE_TMPL_PARAM_BUG */
  typedef __deque_iterator<T, T&, T*>                      iterator;
  typedef __deque_iterator<T, const T&, const T*>          const_iterator;
#endif /* __STL_NON_TYPE_TMPL_PARAM_BUG */

#ifdef __STL_CLASS_PARTIAL_SPECIALIZATION
  typedef reverse_iterator<const_iterator> const_reverse_iterator;
  typedef reverse_iterator<iterator> reverse_iterator;
#else /* __STL_CLASS_PARTIAL_SPECIALIZATION */
  typedef reverse_iterator<const_iterator, value_type, const_reference, 
                           difference_type>  
          const_reverse_iterator;
  typedef reverse_iterator<iterator, value_type, reference, difference_type>
          reverse_iterator; 
#endif /* __STL_CLASS_PARTIAL_SPECIALIZATION */

protected:                      // Internal typedefs
    // map, 指向指针的指针
  typedef pointer* map_pointer;
  typedef simple_alloc<value_type, Alloc> data_allocator;	// value_type类型的空间配置器
  typedef simple_alloc<pointer, Alloc> map_allocator;		// 指针类型的空间配置器
    ...
};
```

注意`map_pointer`是一个指向指针的指针, `deque`就保存一个`map_pointer`, 用它来指向我们分配的内存空间, 用来管理数据.



#### 构造和析构函数

**构造函数**. 有多个重载函数, 接受大部分不同的参数类型. 基本上每一个构造函数都会调用`create_map_and_nodes`, 这就是构造函数的核心, 待会就来分析这个函数实现.

```c++
template <class T, class Alloc = alloc, size_t BufSiz = 0> 
class deque {
    ...
public:                         // Basic types
  deque() : start(), finish(), map(0), map_size(0)	// 默认构造函数
  {
    create_map_and_nodes(0);
  }
  deque(const deque& x) : start(), finish(), map(0), map_size(0)	// 接受一个deque
  {
    create_map_and_nodes(x.size());
    __STL_TRY {
      uninitialized_copy(x.begin(), x.end(), start);
    }
    __STL_UNWIND(destroy_map_and_nodes());
  }
    // 接受 n:初始化大小, value:初始化的值
  deque(size_type n, const value_type& value) : start(), finish(), map(0), map_size(0)
  {
    fill_initialize(n, value);
  }
  deque(int n, const value_type& value) : start(), finish(), map(0), map_size(0)
  {
    fill_initialize(n, value);
  } 
  deque(long n, const value_type& value) : start(), finish(), map(0), map_size(0)
  {
    fill_initialize(n, value);
  }
    // 接受 n:初始化大小
  explicit deque(size_type n) : start(), finish(), map(0), map_size(0)
  {
    fill_initialize(n, value_type());
  }
#ifdef __STL_MEMBER_TEMPLATES
  template <class InputIterator>
  deque(InputIterator first, InputIterator last) : start(), finish(), map(0), map_size(0)
  {
    range_initialize(first, last, iterator_category(first));
  }
#else /* __STL_MEMBER_TEMPLATES */
  deque(const value_type* first, const value_type* last) : start(), finish(), map(0), map_size(0)
  {
    create_map_and_nodes(last - first);
    __STL_TRY {
      uninitialized_copy(first, last, start);
    }
    __STL_UNWIND(destroy_map_and_nodes());
  }
    // 接受两个迭代器, 构造一个范围
  deque(const_iterator first, const_iterator last) : start(), finish(), map(0), map_size(0)
  {
    create_map_and_nodes(last - first);
    __STL_TRY {
      uninitialized_copy(first, last, start);
    }
    __STL_UNWIND(destroy_map_and_nodes());
  }

#endif /* __STL_MEMBER_TEMPLATES */
    ...
};
```

`create_map_and_nodes`函数实现.

1.  计算初始化类型参数的个数
2.  两者取最大的. num_nodes是保证前后都留有位置
3.  计算出数组的头前面留出来的位置保存并在nstart.
4.  为每一个a[cur]分配一个buffer_size的数组, 即这样就实现了二维数组即map(*这里描述为二维数组并不准确, 毕竟指针跟数组是不一样的, 二维数组的地址并不是连续的, 这里这样说只是为了描述起来容易理解.*)
5.  修改start, finish以及分别指向的数组的cur指针的位置, 说通俗就是start和finish分别指向第一个和最后一个元素的位置.

```c++
template <class T, class Alloc, size_t BufSize>
void deque<T, Alloc, BufSize>::create_map_and_nodes(size_type num_elements) 
{
    // 计算初始化类型参数的个数
  size_type num_nodes = num_elements / buffer_size() + 1;
	// 因为deque是头尾插入都是O(1), 就是deque在头和尾都留有空间方便头尾插入
    // 两者取最大的. num_nodes是保证前后都留有位置
  map_size = max(initial_map_size(), num_nodes + 2);
  map = map_allocator::allocate(map_size);	// 分配空间

    // 计算出数组的头前面留出来的位置保存并在nstart.
  map_pointer nstart = map + (map_size - num_nodes) / 2;
  map_pointer nfinish = nstart + num_nodes - 1;
    
  map_pointer cur;
  __STL_TRY 
  {
      // 为每一个a[cur]分配一个buffer_size的数组, 即这样就实现了二维数组即map
    for (cur = nstart; cur <= nfinish; ++cur)
      *cur = allocate_node();
  }
#     ifdef  __STL_USE_EXCEPTIONS 
  catch(...) {
    for (map_pointer n = nstart; n < cur; ++n)
      deallocate_node(*n);
    map_allocator::deallocate(map, map_size);
    throw;
  }
#     endif /* __STL_USE_EXCEPTIONS */
	// 修改start, finish, cur指针的位置
  start.set_node(nstart);
  finish.set_node(nfinish);
  start.cur = start.first;
  finish.cur = finish.first + num_elements % buffer_size();
}
```

**range_initialize函数**

`range_initialize`是保证不同的迭代器类型能正常的工作, 虽然`deque`迭代器是`random_access_iterator_tag`类型, 但是可能我们在执行构造函数的时候, 传入的类型是`list`等其他容器的迭代器, 就会调用该函数. 

主要实现不同迭代器之前的差异而考虑的, 迭代器要考虑"向下兼容". 

```c++
template <class T, class Alloc, size_t BufSize>
template <class InputIterator>
void deque<T, Alloc, BufSize>::range_initialize(InputIterator first,
                                                InputIterator last,
                                                input_iterator_tag) {
  create_map_and_nodes(0);
    // 一个个进行插入操作
  for ( ; first != last; ++first)
    push_back(*first);
}

template <class T, class Alloc, size_t BufSize>
template <class ForwardIterator>
void deque<T, Alloc, BufSize>::range_initialize(ForwardIterator first,
                                                ForwardIterator last,
                                                forward_iterator_tag) {
  size_type n = 0;
    // 计算距离, 申请空间. 失败则释放所有空间
  distance(first, last, n);
  create_map_and_nodes(n);
  __STL_TRY {
    uninitialized_copy(first, last, start);
  }
  __STL_UNWIND(destroy_map_and_nodes());
}
```

**fill_initialize**函数. 

```c++
template <class T, class Alloc, size_t BufSize>
void deque<T, Alloc, BufSize>::fill_initialize(size_type n, const value_type& value) 
{
    // 申请空间
  create_map_and_nodes(n);
  map_pointer cur;
  __STL_TRY {
      //对每个空间进行初始化
    for (cur = start.node; cur < finish.node; ++cur)
      uninitialized_fill(*cur, *cur + buffer_size(), value);
      // 最后一个数组单独处理. 毕竟最后一个数组一般不是会全部填充满
    uninitialized_fill(finish.first, finish.cur, value);
  }
#       ifdef __STL_USE_EXCEPTIONS
  catch(...) {
    for (map_pointer n = start.node; n < cur; ++n)
      destroy(*n, *n + buffer_size());
    destroy_map_and_nodes();
    throw;
  }
#       endif /* __STL_USE_EXCEPTIONS */
}
```





**析构函数**, 分步释放内存. 

```c++
template <class T, class Alloc = alloc, size_t BufSiz = 0> 
class deque {
    ...
public: 
   ~deque() {
    destroy(start, finish);
    destroy_map_and_nodes();
  }
};
```

`deque`是一个"二维数组"并且每个数组之间并不连续, 所以需要一个数组一个数组的执行释放.

```c++
// This is only used as a cleanup function in catch clauses.
template <class T, class Alloc, size_t BufSize>
void deque<T, Alloc, BufSize>::destroy_map_and_nodes() {
// 便利所有的数组, 一个个析构
  for (map_pointer cur = start.node; cur <= finish.node; ++cur)
    deallocate_node(*cur);
    // 内存释放
  map_allocator::deallocate(map, map_size);
}
template <class T, class Alloc = alloc, size_t BufSiz = 0> 
class deque {
    ...
public: 
  pointer allocate_node() { return data_allocator::allocate(buffer_size()); }
  void deallocate_node(pointer n) {
    data_allocator::deallocate(n, buffer_size());
  }
};
```



### deque基本属性获取方法

要注意`deque`的`first`是指向第一个元素的地址, `finish`是指向最后一个元素的后一个地址, 这里两个都是指向一个结构体的指针即迭代器. 

```c++
template <class T, class Alloc = alloc, size_t BufSiz = 0> 
class deque {
    ...
protected:                      // Internal typedefs
...
    // 获取缓冲区大小
  static size_type buffer_size() {
    return __deque_buf_size(BufSiz, sizeof(value_type));
  }
  static size_type initial_map_size() { return 8; }

protected:                      // Data members
  iterator start;	// 指向第一个元素的地址
  iterator finish;	// 指向最后一个元素的后一个地址, 即尾

  map_pointer map;		// 定义map, 指向指针的指针
  size_type map_size;	// map的实际大小
public:                         // Basic accessors
  iterator begin() { return start; }	// 获取头地址
  const_iterator begin() const { return start; }
  iterator end() { return finish; }		// 获取尾地址
  const_iterator end() const { return finish; }
	// 倒转后获取首尾地址.
  reverse_iterator rbegin() { return reverse_iterator(finish); }
  reverse_iterator rend() { return reverse_iterator(start); }
  const_reverse_iterator rbegin() const {
    return const_reverse_iterator(finish);
  }
  const_reverse_iterator rend() const {
    return const_reverse_iterator(start);
  }

  // 获取第一个和最后一个元素的值
  reference front() { return *start; }
  reference back() {
    iterator tmp = finish;
    --tmp;
    return *tmp;
  }
  const_reference front() const { return *start; }
  const_reference back() const {
    const_iterator tmp = finish;
    --tmp;
    return *tmp;
  }
  size_type size() const { return finish - start;; } 	// 获取数组的大小
  size_type max_size() const { return size_type(-1); }	
  bool empty() const { return finish == start; }	// 判断deque是否为空
    
  reference operator[](size_type n) { return start[difference_type(n)]; }
  const_reference operator[](size_type n) const {
    return start[difference_type(n)];
  }
    ...
};
```



### 重载

`deuqe`是`random_access_iterator_tag`类型, 但是这里并没有对+, -进行重载, 其实是迭代器部分我们都已经实现了, 也就不必要再重载该运算符了.

```c++
template <class T, class Alloc = alloc, size_t BufSiz = 0> 
class deque {
    ...
public:                         // Constructor, destructor.
    // 重载 = 
    // 原deque比赋值的大, 就必须清除多余的元素, 否则就需要将多余的元素进行插入
  deque& operator= (const deque& x) {
    const size_type len = size();
    if (&x != this) {
        // 清楚原deque的多余元素
      if (len >= x.size())
        erase(copy(x.begin(), x.end(), start), finish);
      else {
        const_iterator mid = x.begin() + difference_type(len);
        copy(x.begin(), mid, start);
        insert(finish, mid, x.end());
      }
    }
    return *this;
  }        

#ifdef __STL_NON_TYPE_TMPL_PARAM_BUG
public:
    // 重载==, !=, <
  bool operator==(const deque<T, Alloc, 0>& x) const {
    return size() == x.size() && equal(begin(), end(), x.begin());
  }
  bool operator!=(const deque<T, Alloc, 0>& x) const {
    return size() != x.size() || !equal(begin(), end(), x.begin());
  }
  bool operator<(const deque<T, Alloc, 0>& x) const {
    return lexicographical_compare(begin(), end(), x.begin(), x.end());
  }
#endif /* __STL_NON_TYPE_TMPL_PARAM_BUG */
	...
};

template <class T, class Alloc, size_t BufSiz>
bool operator==(const deque<T, Alloc, BufSiz>& x,
                const deque<T, Alloc, BufSiz>& y) {
  return x.size() == y.size() && equal(x.begin(), x.end(), y.begin());
}

template <class T, class Alloc, size_t BufSiz>
bool operator<(const deque<T, Alloc, BufSiz>& x,
               const deque<T, Alloc, BufSiz>& y) {
  return lexicographical_compare(x.begin(), x.end(), y.begin(), y.end());
}
```



## 总结

`deque`的构造析构等基本属性获取的方法分析了, 注意的 `range_initialize`函数是我们看不到的实现. 而它确实非常重要的, 有了它, `deque`就能够对非`random_access_iterator_tag`类型的迭代器起作用, 考虑到"向下兼容". 下一节再来分析`deque`的删除操作.
