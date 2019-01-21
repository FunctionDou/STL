slist

### 前言

上节我们对`slist`的基本构成, 构造析构做了分析, 本节 我们就来分析关于`slist`的基本元素操作. 



### slist分析



#### 基本属性信息

`slist`是只有正向迭代, 所以只能直接获取头部的数据.

```c++
template <class T, class Alloc = alloc>
class slist
{
	...
private:
  slist& operator= (const slist& L);
public:
	// 获取头部元素的地址
  iterator begin() { return iterator((list_node*)head.next); }
  const_iterator begin() const { return const_iterator((list_node*)head.next);}
	// 尾部的地址就是设置的0
  iterator end() { return iterator(0); }
  const_iterator end() const { return const_iterator(0); }

    // slist的长度是通过链表一个个访问计算出来的, 时间复杂度为O(n)
  size_type size() const { return __slist_size(head.next); }	
  
  size_type max_size() const { return size_type(-1); }	// 最大容纳的节点数
  bool empty() const { return head.next == 0; }	// 头部是否为空来判断链表为空
    // 获取前一个节点, slist只能正向迭代, 所以就只能从头到尾的进行查找
    // 这里调用的是__slist_previous函数实现, 这个在上一节已经分析过了
  iterator previous(const_iterator pos) {
    return iterator((list_node*) __slist_previous(&head, pos.node));
  }
  const_iterator previous(const_iterator pos) const {
    return const_iterator((list_node*) __slist_previous(&head, pos.node));
  }
	...
};
```

**swap**. slist进行交换并不是交换所以的元素, 实际只是交换了`head`的指向

```c++
template <class T, class Alloc = alloc>
class slist
{
	...
public:
  void swap(slist& L)
  {
    list_node_base* tmp = head.next;
    head.next = L.head.next;
    L.head.next = tmp;
  }
  ...
};
template <class T, class Alloc>
inline void swap(slist<T, Alloc>& x, slist<T, Alloc>& y) {
  x.swap(y);
}
```



#### pop和push

slist的push和pop都只能在头部进行操作, 是头插法.

```c++
template <class T, class Alloc = alloc>
class slist
{
	...
public:
    // 获取头部元素的数据
  reference front() { return ((list_node*) head.next)->data; }
  const_reference front() const { return ((list_node*) head.next)->data; }
    // 在头部进行插入操作
  void push_front(const value_type& x)   {
    __slist_make_link(&head, create_node(x));
  }
    // 删除第一个元素
  void pop_front() {
    list_node* node = (list_node*) head.next;
    head.next = node->next;
    destroy_node(node);	// 析构并释放掉
  }
    ...
};
```



#### 运算符重载

重载=, 私有

```c++
template <class T, class Alloc = alloc>
class slist
{
	...
private:
    // = 运算符定义是私有的
  slist& operator= (const slist& L);
    ...
};

// 这定义的是私有, 无法直接调用=
template <class T, class Alloc>
slist<T, Alloc>& slist<T,Alloc>::operator=(const slist<T, Alloc>& L)
{
    // 不是同一链表
  if (&L != this) {
    list_node_base* p1 = &head;
    list_node* n1 = (list_node*) head.next;
    const list_node* n2 = (const list_node*) L.head.next;
    while (n1 && n2) {
        // 深拷贝, 直到一个链表结束
      n1->data = n2->data;
      p1 = n1;
      n1 = (list_node*) n1->next;
      n2 = (const list_node*) n2->next;
    }
      // 原链表还有剩,  就删除原链表多余的数据
    if (n2 == 0)
      erase_after(p1, 0);	
      // 否则就将n2的剩余元素初始化
    else
      _insert_after_range(p1, const_iterator((list_node*)n2), const_iterator(0));
  }
  return *this;
} 
```

重载==, 共有

```c++
template <class T, class Alloc = alloc>
class slist
{
	...
public:
    // 友元
  friend bool operator== __STL_NULL_TMPL_ARGS(const slist<T, Alloc>& L1,
                                              const slist<T, Alloc>& L2);
    ...
};
template <class T, class Alloc>
bool operator==(const slist<T, Alloc>& L1, const slist<T, Alloc>& L2)
{
  typedef typename slist<T,Alloc>::list_node list_node;
  list_node* n1 = (list_node*) L1.head.next;
  list_node* n2 = (list_node*) L2.head.next;
    // 一个个元素进行比较
  while (n1 && n2 && n1->data == n2->data) {
    n1 = (list_node*) n1->next;
    n2 = (list_node*) n2->next;
  }
  return n1 == 0 && n2 == 0;
}
```

重载<, 共有

```c++
template <class T, class Alloc>
inline bool operator<(const slist<T, Alloc>& L1, const slist<T, Alloc>& L2)
{	// 直接进行比较
  return lexicographical_compare(L1.begin(), L1.end(), L2.begin(), L2.end());
}
```



#### 插入操作

插入操作的核心是在上节分析的`__slist_make_link`函数, **将元素插入指定位置后.**



**insert_after** : 将元素插入指定位置后

```c++
template <class T, class Alloc = alloc>
class slist
{
	...
private: // 私有, 由内部函数调用
  list_node* _insert_after(list_node_base* pos, const value_type& x) {
    return (list_node*) (__slist_make_link(pos, create_node(x)));
  }
public:
  iterator insert_after(iterator pos, const value_type& x) {
    return iterator(_insert_after(pos.node, x));	// 调用 _insert_after
  }

  iterator insert_after(iterator pos) {
    return insert_after(pos, value_type());	// 调用 _insert_after
  }

  void insert_after(iterator pos, size_type n, const value_type& x) {
    _insert_after_fill(pos.node, n, x);	// 调用 _insert_after_fill
  }
  template <class InIter>
  void insert_after(iterator pos, InIter first, InIter last) {
    _insert_after_range(pos.node, first, last);	// 调用 _insert_after_fill
  }
    ...
};
```



**insert** : 将元素插入指定位置之前

```c++
template <class T, class Alloc = alloc>
class slist
{
	...
public:
    // 元素插入在指定位置之前
    // 先调用__slist_previous获取指定位置之前的位置, 在执行_insert_after
  iterator insert(iterator pos, const value_type& x) {
    return iterator(_insert_after(__slist_previous(&head, pos.node), x));
  }
  iterator insert(iterator pos) {
    return iterator(_insert_after(__slist_previous(&head, pos.node),
                                  value_type()));
  }
    // 无返回值, 且元素插入在指定位置之前
  void insert(iterator pos, size_type n, const value_type& x) {
    _insert_after_fill(__slist_previous(&head, pos.node), n, x);
  } 
  template <class InIter>
  void insert(iterator pos, InIter first, InIter last) {
    _insert_after_range(__slist_previous(&head, pos.node), first, last);
  }
    ...
};
```



#### 删除操作

**arase_after删除指定位置的元素** .

```c++
template <class T, class Alloc = alloc>
class slist
{
	...
private: // 私有, 由内部函数调用
    // 删除一个元素
  list_node_base* erase_after(list_node_base* pos) {
    list_node* next = (list_node*) (pos->next);
    list_node_base* next_next = next->next;
    pos->next = next_next;
    destroy_node(next);
    return next_next;
  }
   
  list_node_base* erase_after(list_node_base* before_first,
                              list_node_base* last_node) {
    list_node* cur = (list_node*) (before_first->next);
      // 将[first, last) 范围的元素进行删除
    while (cur != last_node) {
      list_node* tmp = cur;
      cur = (list_node*) cur->next;
      destroy_node(tmp);
    }
    before_first->next = last_node;
    return last_node;
  }
    
public:
    // 调用erase_after函数
  iterator erase_after(iterator pos) {
    return iterator((list_node*)erase_after(pos.node));
  }
  iterator erase_after(iterator before_first, iterator last) {
    return iterator((list_node*)erase_after(before_first.node, last.node));
  }
    ...
};
```



**erase删除指定位置之前的元素**

```c++
template <class T, class Alloc = alloc>
class slist
{
	...
public:
    // 删除指定位置之前的元素
  iterator erase(iterator pos) {
    return (list_node*) erase_after(__slist_previous(&head, pos.node));
  }
  iterator erase(iterator first, iterator last) {
    return (list_node*) erase_after(__slist_previous(&head, first.node),
                                    last.node);
  }
    ...
};
```



**resize重新调整slist的链表**.

```c++
template <class T, class Alloc>
void slist<T, Alloc>::resize(size_type len, const T& x)
{
  list_node_base* cur = &head;
    // 定位出指定大小的后链表的位置
  while (cur->next != 0 && len > 0) {
    --len;
    cur = cur->next;
  }
    // 链表过长将cur之后的全部删除
  if (cur->next) 
    erase_after(cur, 0);
    // 否则链表过短, 重新插入
  else
    _insert_after_fill(cur, len, x);
}

template <class T, class Alloc = alloc>
class slist
{
	...
public:
  void resize(size_type new_size, const T& x);
  void resize(size_type new_size) { resize(new_size, T()); }
    ...
};
```



**remove删除链表中所有指定的元素**

```c++
template <class T, class Alloc>
void slist<T,Alloc>::remove(const T& val)
{
  list_node_base* cur = &head;
  while (cur && cur->next) {
      // 删除指定元素
    if (((list_node*) cur->next)->data == val)
      erase_after(cur);
    else
      cur = cur->next;
  }
}
```

```c++
template <class T, class Alloc> 
template <class Predicate> void slist<T,Alloc>::remove_if(Predicate pred)
{
  list_node_base* cur = &head;
  while (cur->next) {
      // 自定义条件判断
    if (pred(((list_node*) cur->next)->data))
      erase_after(cur);
    else
      cur = cur->next;
  }
}
```



**unique删除连续的相同数据, 只留第一个**

```c++
template <class T, class Alloc> 
void slist<T,Alloc>::unique()
{
  list_node_base* cur = head.next;
  if (cur) {
    while (cur->next) {
        // 判断下一个是否相同
      if (((list_node*)cur)->data == ((list_node*)(cur->next))->data)
        erase_after(cur);
      else
        cur = cur->next;
    }
  }
}
```



**merge将两个链表进行合并并排序(前提是两个链表已经排序好了)**

```c++
template <class T, class Alloc>
void slist<T,Alloc>::merge(slist<T,Alloc>& L)
{
  list_node_base* n1 = &head;
  while (n1->next && L.head.next) {
      // 安从大到小排序
    if (((list_node*) L.head.next)->data < ((list_node*) n1->next)->data) 
      __slist_splice_after(n1, &L.head, L.head.next);
    n1 = n1->next;
  }
  if (L.head.next) {
    n1->next = L.head.next;
    L.head.next = 0;
  }
}
```



#### sort

`slist`排序跟`list`的排序操作是一样的, 实现也是类似的. 这里便使用`list`的sort步骤来分析

这个sort的分析 : 

-   这里将每个重要的参数列出来解释其含义
    1.  `fill` : 当前可以处理的元素个数为2^fill个
    2.  `counter[fill]` : 可以容纳2^(fill+1)个元素
    3.  `carry` : 一个临时中转站, 每次将一元素插入到counter[i]链表中.

在处理的元素个数不足2^fill个时，在`counter[i](0<i<fill)`之前转移元素

具体是显示步骤是：

1.  每次读一个数据到`carry`中，并将carry的数据转移到`counter[0]`中
    1.  当`counter[0]`中的数据个数少于2时，持续转移数据到counter[0]中
    2.  当counter[0]的数据个数等于2时，将counter[0]中的数据转移到counter[1]...从counter[i]转移到counter[i+1],直到counter[fill]中数据个数达到2^(fill+1)个。
2.  ++fill, 重复步骤1

```c++
template <class T, class Alloc>
void slist<T,Alloc>::sort()
{
  if (head.next && head.next->next) {
    slist carry;
    slist counter[64];
    int fill = 0;
    while (!empty()) {
      __slist_splice_after(&carry.head, &head, head.next);
      int i = 0;
      while (i < fill && !counter[i].empty()) {
        counter[i].merge(carry);
        carry.swap(counter[i]);
        ++i;
      }
      carry.swap(counter[i]);
      if (i == fill)
        ++fill;
    }

    for (int i = 1; i < fill; ++i)
      counter[i].merge(counter[i-1]);
    this->swap(counter[fill-1]);
  }
}
```



### 总结

本节分析了`slist`具体操作的实现, 其实很多实现都与`list`很像, 引入`slist`也是因为有时需要的功能并不是很全面, 只要插入等简单操作, 使用list的代价相对大, 这时用`slist`就是很好的选择. 注意, list只能正向迭代, 每次使用`insert`都会遍历整个链表, `insert_after`只需要O(1).