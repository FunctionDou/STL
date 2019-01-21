# slist

### 前言

不同于`list`是`Forward Iterator`类型的双向链表, `slist`是单向链表, 也是`Bidirectional Iterator`类型. slist主要耗费的空间小, 操作一些特定的操作更加的快, 同样类似与`slist`, 在执行插入和删除操作迭代器都不会像vector使迭代器失效. `slist`主要的问题在于它是单向的, 正向迭代器, 只有`push_front`, 没有push_back的操作, 指定位置之后的插入和删除时间都是O(1), 而在指定位置之前插入就需要重新遍历链表时间就是O(n), 效率很低.



### slist源码

`slist`的构成与`list`还是异曲同工.



#### 节点结构

将节点的每一部分都进行剥离, 节点是一个结构构成, 数据是继承节点.

```c++
struct __slist_node_base
{
  __slist_node_base* next;	// 只有指向下一个节点的指针
};
// 继承节点指针, __slist_node增加数据
template <class T>
struct __slist_node : public __slist_node_base
{
  T data;
};
```



#### 节点的操作

在节点指定节点后插入新的节点.

```c++
inline __slist_node_base* __slist_make_link(__slist_node_base* prev_node, __slist_node_base* new_node)
{
  new_node->next = prev_node->next;
  prev_node->next = new_node;
  return new_node;
}
```

寻找指定节点的prev节点. 因为slist是正向迭代器就只有遍历链表

```c++
inline __slist_node_base* __slist_previous(__slist_node_base* head, const __slist_node_base* node)
{
    // 遍历整个链表来寻找
  while (head && head->next != node)
    head = head->next;
  return head;
}
inline const __slist_node_base* __slist_previous(const __slist_node_base* head,const __slist_node_base* node)
{
    // 遍历整个链表来寻找
  while (head && head->next != node)
    head = head->next;
  return head;
}
```

将指定范围的链表剪切到pos链表后

```c++
inline void __slist_splice_after(__slist_node_base* pos,__slist_node_base* before_first, __slist_node_base* before_last)
{
  if (pos != before_first && pos != before_last) {
    __slist_node_base* first = before_first->next;
    __slist_node_base* after = pos->next;
      // 将before_first之后的调整为before_last后面的链表
    before_first->next = before_last->next;
      // 将[before_first->next, before_last]插入到pos之后
    pos->next = first;
    before_last->next = after;
  }
}
```

将链表进行倒转

```c++
inline __slist_node_base* __slist_reverse(__slist_node_base* node)
{
  __slist_node_base* result = node;
  node = node->next;
  result->next = 0;
  while(node) {
      // 将原链表一一的插入到临时链表的尾, 实现链表的元素倒转
    __slist_node_base* next = node->next;
    node->next = result;
    result = node;
    node = next;
  }
  return result;
}
```



#### 迭代器

之前说过`slist`的迭代器是正向的, 现在就来具体分析.

`__slist_iterator_base`单向链表的迭代器基本结构

```c++
struct __slist_iterator_base
{
    // 定义类型
  typedef size_t size_type;
  typedef ptrdiff_t difference_type;
  typedef forward_iterator_tag iterator_category;
	// 定义节点指针
  __slist_node_base* node;
	// 构造函数
  __slist_iterator_base(__slist_node_base* x) : node(x) {}
    // incr: 只正向.
  void incr() { node = node->next; }
	// 只重载了==,!=
  bool operator==(const __slist_iterator_base& x) const {
    return node == x.node;
  }
  bool operator!=(const __slist_iterator_base& x) const {
    return node != x.node;
  }
};
```

`__slist_iterator`单向链表迭代器结构. 只重载实现了++操作, 没有--, 因为就只是正向迭代, --操作就需要重新遍历链表.

```c++
template <class T, class Ref, class Ptr>
struct __slist_iterator : public __slist_iterator_base
{
	// 定义迭代器
  typedef __slist_iterator<T, T&, T*>             iterator;
  typedef __slist_iterator<T, const T&, const T*> const_iterator;
  typedef __slist_iterator<T, Ref, Ptr>           self;
	// 满足traits编程
  typedef T value_type;
  typedef Ptr pointer;
  typedef Ref reference;
  	// 定义链表节点
  typedef __slist_node<T> list_node;
	// 构造函数
  __slist_iterator(list_node* x) : __slist_iterator_base(x) {}
  __slist_iterator() : __slist_iterator_base(0) {}
  __slist_iterator(const iterator& x) : __slist_iterator_base(x.node) {}
	// 获得当前指针的值
  reference operator*() const { return ((list_node*) node)->data; }
#ifndef __SGI_STL_NO_ARROW_OPERATOR
  pointer operator->() const { return &(operator*()); }
#endif /* __SGI_STL_NO_ARROW_OPERATOR */

	// 对++重载
  self& operator++()
  {
    incr(); // 指向下一个节点
    return *this;
  }
  self operator++(int)
  {
    self tmp = *this;
    incr();
    return tmp;
  }
};

#ifndef __STL_CLASS_PARTIAL_SPECIALIZATION
inline ptrdiff_t* distance_type(const __slist_iterator_base&)
{
  return 0;
}
inline forward_iterator_tag iterator_category(const __slist_iterator_base&)
{
  return forward_iterator_tag();
}
template <class T, class Ref, class Ptr> 
inline T*  value_type(const __slist_iterator<T, Ref, Ptr>&) {
  return 0;
}
#endif /* __STL_CLASS_PARTIAL_SPECIALIZATION */
```

通过遍历整个链表计算链表的长度.

```c++
inline size_t __slist_size(__slist_node_base* node)
{
  size_t result = 0;
  for ( ; node != 0; node = node->next)
    ++result;
  return result;
}
```



#### slist分析

**slist类型定义**.

```c++
template <class T, class Alloc = alloc>
class slist
{
public:
    // 满足traits编程
  typedef T value_type;
  typedef value_type* pointer;
  typedef const value_type* const_pointer;
  typedef value_type& reference;
  typedef const value_type& const_reference;
  typedef size_t size_type;
  typedef ptrdiff_t difference_type;
	
    // 定义迭代器
  typedef __slist_iterator<T, T&, T*>             iterator;
  typedef __slist_iterator<T, const T&, const T*> const_iterator;

private:
  typedef __slist_node<T> list_node;	// 定义链表
  typedef __slist_node_base list_node_base;	// 定义节点
  typedef __slist_iterator_base iterator_base;	// 单向链表迭代器的基本结构定义
  typedef simple_alloc<list_node, Alloc> list_node_allocator;	// 定义空间配置器

private:
  list_node_base head;	// 定义节点的头
    ...	
};
```



##### slist构造函数

```c++
template <class T, class Alloc = alloc>
class slist
{
	...
public:
  slist() { head.next = 0; }	// 默认构造函数, head.next == 0

    // 以下构造函数都调用fill_initialize填充链表
  slist(size_type n, const value_type& x) { fill_initialize(n, x); }
  slist(int n, const value_type& x) { fill_initialize(n, x); }
  slist(long n, const value_type& x) { fill_initialize(n, x); }
  explicit slist(size_type n) { fill_initialize(n, value_type()); }	// 不能隐式调用该构造函数

#ifdef __STL_MEMBER_TEMPLATES
    // 接受两个迭代器, 范围初始化
  template <class InputIterator>
  slist(InputIterator first, InputIterator last) {
    range_initialize(first, last);
  }
#else /* __STL_MEMBER_TEMPLATES */
  slist(const_iterator first, const_iterator last) {
    range_initialize(first, last);
  }
  slist(const value_type* first, const value_type* last) {
    range_initialize(first, last);
  }
#endif /* __STL_MEMBER_TEMPLATES */
	...
};
```

fill_initialize, 初始化

```c++
void fill_initialize(size_type n, const value_type& x) {
    head.next = 0;
    __STL_TRY {
      _insert_after_fill(&head, n, x);
    }
    __STL_UNWIND(clear());
}
```

range_initialize, 自定义范围初始化

```c++
template <class T, class Alloc = alloc>
class slist
{
	...
private:
#ifdef __STL_MEMBER_TEMPLATES
  template <class InputIterator>
  void range_initialize(InputIterator first, InputIterator last) {
    head.next = 0;
    __STL_TRY {
      _insert_after_range(&head, first, last);
    }
    __STL_UNWIND(clear());
  }
#else /* __STL_MEMBER_TEMPLATES */
  void range_initialize(const value_type* first, const value_type* last) {
    head.next = 0;
    __STL_TRY {
      _insert_after_range(&head, first, last);
    }
    __STL_UNWIND(clear());
  }
  void range_initialize(const_iterator first, const_iterator last) {
    head.next = 0;
    __STL_TRY {
      _insert_after_range(&head, first, last);
    }
    __STL_UNWIND(clear());
  }
#endif /* __STL_MEMBER_TEMPLATES */
	...
};
```

range_initialize和fill_initialize函数都是调用`_insert_after_range`, _insert_after_range也是有多个重载函数.

**_insert_after_range**, 初始化成员属性

```c++
template <class T, class Alloc = alloc>
class slist
{
	...
private:
  void _insert_after_fill(list_node_base* pos,
                          size_type n, const value_type& x) {
      // 构造出n个节点并初始化为x
    for (size_type i = 0; i < n; ++i)
      pos = __slist_make_link(pos, create_node(x));
  }

#ifdef __STL_MEMBER_TEMPLATES
  template <class InIter>
  void _insert_after_range(list_node_base* pos, InIter first, InIter last) {
      // 构造出 [first, last) 个节点并初始化为first
    while (first != last) {
      pos = __slist_make_link(pos, create_node(*first));
      ++first;
    }
  }
#else /* __STL_MEMBER_TEMPLATES */
  void _insert_after_range(list_node_base* pos,
                           const_iterator first, const_iterator last) {
      // 构造出 [first, last) 个节点并初始化为first
    while (first != last) {
      pos = __slist_make_link(pos, create_node(*first));
      ++first;
    }
  }
  void _insert_after_range(list_node_base* pos,
                           const value_type* first, const value_type* last) {
      // 构造出 [first, last) 个节点并初始化为first
    while (first != last) {
      pos = __slist_make_link(pos, create_node(*first));
      ++first;
    }
  }
#endif /* __STL_MEMBER_TEMPLATES */
    ...
};
```



**`create_node`函数, 调用空间配置器分配空间然后在调用构造函数**

```c++
template <class T, class Alloc = alloc>
class slist
{
	...
private:
  static list_node* create_node(const value_type& x) {
      // 空间配置器分配空间
    list_node* node = list_node_allocator::allocate();
    __STL_TRY {
        // 调用构造函数
      construct(&node->data, x);
      node->next = 0;
    }
      // 如果分配失败就释放掉之前分配的所有空间
    __STL_UNWIND(list_node_allocator::deallocate(node));
    return node;
  }
	...
};
```

##### 析构函数

```c++
template <class T, class Alloc = alloc>
class slist
{
	...
private:
    ~slist() { clear(); }	// 调用clear函数将所有元素进行删除和释放
    ...
};
```

**`destroy_node`函数, 调用析构函数之后再释放空间**

```c++
template <class T, class Alloc = alloc>
class slist
{
	...
private:
  static void destroy_node(list_node* node) {
    destroy(&node->data);	// 调用析构函数
    list_node_allocator::deallocate(node);	// 释放内存空间
  }
	...
};
```



##### 拷贝构造函数

```c++
template <class T, class Alloc = alloc>
class slist
{
	...
private:
    // 直接调用range_initialize函数进行初始化
  slist(const slist& L) { range_initialize(L.begin(), L.end()); }
    ...
};
```



### 总结

本节只是对`slist`的基本构成, 构造函数和析构函数做了一个探讨, 下一节就准备分析`slist`的删除, 插入等具体操作的实现.    