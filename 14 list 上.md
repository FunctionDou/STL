# list

## 前言

前几节我们分析了`vector`的实现, `vector`的缺点也很明显, 在频率较高的插入和删除时效率就太低了, 本节我们就来分析在频率较高的插入和删除很也很好的效率的`list`. 

`list`是用链表进行实现的, 而链表对删除, 插入的时间复杂度为O(1), 效率相当高, 但是随机访问的时间复杂度为O(n). `list`将具体实现分成几个部分, 通过嵌套的方式进行调用, 所以list实现也很灵活. 而且**`list`在插入和删除操作后迭代器并不会失效.**

本节只分析关于`list`的结构, 构造和析构函数的实现.



## list基本结构框架

`list`将基本的框架分成`__list_node`(实现节点), `__list_iterator`(实现迭代器)两部分方便随时调用. 下面我们就先慢慢分析. 



### __list_node链表结构

`__list_node`用来实现节点, 数据结构中就储存前后指针和属性. 

```c++
template <class T> struct __list_node 
{
    // 前后指针
  	typedef void* void_pointer;
  	void_pointer next;
  	void_pointer prev;
    // 属性
  	T data;
};
```



### __list_iterator结构

**基本类型**

```c++
template<class T, class Ref, class Ptr> struct __list_iterator 
{
  	typedef __list_iterator<T, T&, T*>             iterator;	// 迭代器
  	typedef __list_iterator<T, const T&, const T*> const_iterator;
  	typedef __list_iterator<T, Ref, Ptr>           self;		
	
    // 迭代器是bidirectional_iterator_tag类型
  	typedef bidirectional_iterator_tag iterator_category;
  	typedef T value_type;
  	typedef Ptr pointer;
  	typedef Ref reference;
  	typedef size_t size_type;
  	typedef ptrdiff_t difference_type;
    ... 
};
```

**构造函数**

```c++
template<class T, class Ref, class Ptr> struct __list_iterator 
{
    ...
    // 定义节点指针
  	typedef __list_node<T>* link_type;
  	link_type node;
	// 构造函数
  	__list_iterator(link_type x) : node(x) {}
  	__list_iterator() {}
  	__list_iterator(const iterator& x) : node(x.node) {}
   ... 
};
```

**重载**

```c++
template<class T, class Ref, class Ptr> struct __list_iterator 
{
    ...
	// 重载
  	bool operator==(const self& x) const { return node == x.node; }
  	bool operator!=(const self& x) const { return node != x.node; }
    // 对*和->操作符进行重载
  	reference operator*() const { return (*node).data; }
#ifndef __SGI_STL_NO_ARROW_OPERATOR
  	pointer operator->() const { return &(operator*()); }
#endif /* __SGI_STL_NO_ARROW_OPERATOR */

    // ++和--是直接操作的指针指向next还是prev, 因为list是一个双向链表
  	self& operator++() 
    { 
	    node = (link_type)((*node).next);
	    return *this;
  	}
  	self operator++(int) 
    { 
	    self tmp = *this;
	    ++*this;
	    return tmp;
  	}
  	self& operator--() 
    { 
	    node = (link_type)((*node).prev);
	    return *this;
  	}
  	self operator--(int) 
    { 
    	self tmp = *this;
    	--*this;
    	return tmp;
  	}
};
```



## list结构

`list`自己定义了嵌套类型满足`traits`编程. `list`迭代器是`bidirectional_iterator_tag`类型, 并不是一个普通指针. 

**list基本类型定义**

**`list`在定义node节点时, 定义的不是一个指针, 这里要注意.**

```c++
template <class T, class Alloc = alloc>
class list 
{
protected:
    typedef void* void_pointer;
    typedef __list_node<T> list_node;	// 节点
    typedef simple_alloc<list_node, Alloc> list_node_allocator;	// 空间配置器
public:      
    // 定义嵌套类型
    typedef T value_type;
    typedef value_type* pointer;
    typedef const value_type* const_pointer;
    typedef value_type& reference;
    typedef const value_type& const_reference;
    typedef list_node* link_type;
    typedef size_t size_type;
    typedef ptrdiff_t difference_type;
    
protected:
    // 定义一个节点, 这里节点并不是一个指针.
    link_type node;
    
public:
    // 定义迭代器
    typedef __list_iterator<T, T&, T*>             iterator;
    typedef __list_iterator<T, const T&, const T*> const_iterator;
	...
};
```



**list构造和析构函数实现**

构造函数前期准备

1.  分配空间`get_node`
2.  释放空间`put_node`
3.  分配并构造`create_node`
4.  析构并释放空间`destroy_node`
5.  对节点进行初始化`empty_initialize`

```c++
template <class T, class Alloc = alloc>
class list 
{
    ...
protected:
	// 分配一个元素大小的空间, 返回分配的地址
    link_type get_node() { return list_node_allocator::allocate(); }
    // 释放一个元素大小的内存
    void put_node(link_type p) { list_node_allocator::deallocate(p); }
	// 分配一个元素大小的空间并调用构造初始化内存
    link_type create_node(const T& x) 
    {
      link_type p = get_node();
      __STL_TRY {
        construct(&p->data, x);
      }
      __STL_UNWIND(put_node(p));
      return p;
    }
    // 调用析构并释放一个元素大小的空间
    void destroy_node(link_type p) {
      destroy(&p->data);
      put_node(p);
    }
    // 对节点初始化
    void empty_initialize() 
    { 
      node = get_node();
      node->next = node;
      node->prev = node;
    }  
    ...
};
```

**构造函数**

1.  多个重载, 以实现直接构造n个节点并初始化一个值, 支持传入迭代器进行范围初始化, 也支持接受一个`list`参数, 同样进行范围初始化.
2.  **每个构造函数都会创造一个空的node节点, 为了保证我们在执行任何操作都不会修改迭代器.**

```c++
template <class T, class Alloc = alloc>
class list 
{
    ...
protected: 
    // 构造函数
    list() { empty_initialize(); }	// 默认构造函数, 分配一个空的node节点
    // 都调用同一个函数进行初始化
    list(size_type n, const T& value) { fill_initialize(n, value); }
    list(int n, const T& value) { fill_initialize(n, value); }
    list(long n, const T& value) { fill_initialize(n, value); }
    // 分配n个节点
    explicit list(size_type n) { fill_initialize(n, T()); }

#ifdef __STL_MEMBER_TEMPLATES
    // 接受两个迭代器进行范围的初始化
    template <class InputIterator>
      list(InputIterator first, InputIterator last) {
        range_initialize(first, last);
      }
#else /* __STL_MEMBER_TEMPLATES */
    // 接受两个迭代器进行范围的初始化
    list(const T* first, const T* last) { range_initialize(first, last); }
    list(const_iterator first, const_iterator last) {
      range_initialize(first, last);
    }
#endif /* __STL_MEMBER_TEMPLATES */
    // 接受一个list参数, 进行拷贝
    list(const list<T, Alloc>& x) {
      range_initialize(x.begin(), x.end());
    }
    list<T, Alloc>& operator=(const list<T, Alloc>& x);
    ...
};
```

构造函数内大都调用这个函数, 可以看出来`list`在初始化的时候都会**构造一个空的`node`节点**, 然后对元素进行`insert`插入操作.

```c++
    void fill_initialize(size_type n, const T& value) {
      empty_initialize();
      __STL_TRY {
        insert(begin(), n, value);
      }
      __STL_UNWIND(clear(); put_node(node));
    }
```

**析构函数** , 释放所有的节点空间. 包括最初的空节点.

```c++
    ~list() {
        // 删除初空节点以外的所有节点
      clear();
        // 删除空节点
      put_node(node);
    }
```



## 基本属性获取

要注意一点`list`中的迭代器一般不会被修改, 因为`node`节点始终指向的一个空节点同时`list`是一个循环的链表, 空节点正好在头和尾的中间, 所以`node.next`就是指向头的指针, `node.prev`就是指向结束的指针, `end`返回的是最后一个数据的后一个地址也就是`node`. 清楚这些后就容易看懂下面怎么获取属性了.

```c++
template <class T, class Alloc = alloc>
class list 
{
    ...
public: 
	iterator begin() { return (link_type)((*node).next); }	// 返回指向头的指针
    const_iterator begin() const { return (link_type)((*node).next); }
    iterator end() { return node; }	// 返回最后一个元素的后一个的地址
    const_iterator end() const { return node; }
    
    // 这里是为旋转做准备, rbegin返回最后一个地址, rend返回第一个地址. 我们放在配接器里面分析
    reverse_iterator rbegin() { return reverse_iterator(end()); }
    const_reverse_iterator rbegin() const { 
      return const_reverse_iterator(end()); 
    }
    reverse_iterator rend() { return reverse_iterator(begin()); }
    const_reverse_iterator rend() const { 
      return const_reverse_iterator(begin());
    } 
    
    // 判断是否为空链表, 这是判断只有一个空node来表示链表为空.
    bool empty() const { return node->next == node; }
    // 因为这个链表, 地址并不连续, 所以要自己迭代计算链表的长度.
    size_type size() const {
      size_type result = 0;
      distance(begin(), end(), result);
      return result;
    }
    size_type max_size() const { return size_type(-1); }
    // 返回第一个元素的值
    reference front() { return *begin(); }
    const_reference front() const { return *begin(); }
    // 返回最后一个元素的值
    reference back() { return *(--end()); }
    const_reference back() const { return *(--end()); }
    
    // 交换
    void swap(list<T, Alloc>& x) { __STD::swap(node, x.node); }
    ...
};
template <class T, class Alloc>
inline void swap(list<T, Alloc>& x, list<T, Alloc>& y) 
{
  	x.swap(y);
}
```



## 总结

本节仅仅对list基本类型, 构造, 析构和怎么获取属性做了分析, 将`list`拆开一步步分析, 下节继续分析`list`, 主要探讨其插入,删除操作.

