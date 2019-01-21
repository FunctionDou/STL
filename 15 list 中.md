# list

## 前言

上节分析了`list`的类型, 构造析构的实现, 本节我们着重探讨`list`的push, pop, 插入和删除等基本操作.



## list实际操作

这里写了一些关于本节大都会用到的操作, 执行完后我们就来分析函数的具体实现.

```c++
int main()
{
	list<int> List;
	List.push_back(1);
	List.push_front(0);
	List.push_back(2);
	List.push_front(3);
	List.insert(++List.begin(), 5);	// list没有重载 + 运算符, 所以不能直接操作迭代器
	
	list<int> List1(List);
	list<int> List2 = List1;
	cout << "list size = " << List2.size() << endl;
	if(!List2.empty())
		for (auto i : List2)
		{
			cout << i << " ";
		}
	cout << endl;

	exit(0);
}
```

输出

```c++
list size = 5
3 5 0 1 2 
```



## list操作



###  push和pop操作

因为`list`是一个循环的双链表, 所以push和pop就必须实现是在头插入, 删除还是在尾插入和删除. push操作都调用`insert`函数, pop操作都调用`erase`函数.

```c++
template <class T, class Alloc = alloc>
class list 
{
    ...
    // 直接在头部或尾部插入
    void push_front(const T& x) { insert(begin(), x); }
    void push_back(const T& x) { insert(end(), x); }
    // 直接在头部或尾部删除
    void pop_front() { erase(begin()); }
    void pop_back() { 
      iterator tmp = end();
      erase(--tmp);
    }
    ...
};
```



### 删除操作

删除元素的操作大都是由`erase`函数来实现的, 其他的所有函数都是直接或间接调用`erase`. `list`是链表, 所以链表怎么实现删除, list就在怎么操作.

```c++
template <class T, class Alloc = alloc>
class list 
{
    ...
	iterator erase(iterator first, iterator last);
    void clear();   
    // 参数是一个迭代器
    // 修改该元素的前后指针指向再单独释放节点就行了
	iterator erase(iterator position) {
      link_type next_node = link_type(position.node->next);
      link_type prev_node = link_type(position.node->prev);
      prev_node->next = next_node;
      next_node->prev = prev_node;
      destroy_node(position.node);
      return iterator(next_node);
    }
    ...
};
// erase的重载, 删除两个迭代器之间的元素
template <class T, class Alloc>
list<T,Alloc>::iterator list<T, Alloc>::erase(iterator first, iterator last) 
{
    // 就是一次次调用erase进行删除
  	while (first != last) 
        erase(first++);
  	return last;
}
// remove调用erase链表清除
template <class T, class Alloc>
void list<T, Alloc>::remove(const T& value) {
  iterator first = begin();
  iterator last = end();
  while (first != last) {
    iterator next = first;
    ++next;
    if (*first == value) erase(first);
    first = next;
  }
}
```

 `clear`函数是删除除空节点以外的所有节点, 即只留下了最初创建的空节点.

```c++
// 删除除空节点以外的所有节点
template <class T, class Alloc> 
void list<T, Alloc>::clear()
{
  link_type cur = (link_type) node->next;
    // 除空节点都删除
  while (cur != node) {
    link_type tmp = cur;
    cur = (link_type) cur->next;
    destroy_node(tmp);
  }
  node->next = node;
  node->prev = node;
}
```



### 重载

`list`也提供了基本操作的重载, 所以我们使用`list`也很方便.

*相等比较*

```c++
// 判断两个list相等
template <class T, class Alloc>
inline bool operator==(const list<T,Alloc>& x, const list<T,Alloc>& y) 
{
  typedef typename list<T,Alloc>::link_type link_type;
  link_type e1 = x.node;
  link_type e2 = y.node;
  link_type n1 = (link_type) e1->next;
  link_type n2 = (link_type) e2->next;
    // 将两个链表执行一一的对比来分析是否相等. 
    // 这里不把元素个数进行一次比较, 主要获取个数时也要遍历整个数组, 所以就不将个数纳入比较
  for ( ; n1 != e1 && n2 != e2 ; n1 = (link_type) n1->next, n2 = (link_type) n2->next)
    if (n1->data != n2->data)
      return false;
  return n1 == e1 && n2 == e2;
}
```

*小于比较*

```c++
template <class T, class Alloc>
inline bool operator<(const list<T, Alloc>& x, const list<T, Alloc>& y) {
  return lexicographical_compare(x.begin(), x.end(), y.begin(), y.end());
}
```

*赋值操作*

需要考虑两个链表的实际大小不一样时的操作.

1.  原链表大 : 复制完后要删除掉原链表多余的元素
2.  原链表小 : 复制完后要还要将x链表的剩余元素以插入的方式插入到原链表中

```c++
template <class T, class Alloc>
list<T, Alloc>& list<T, Alloc>::operator=(const list<T, Alloc>& x) {
  if (this != &x) {
    iterator first1 = begin();
    iterator last1 = end();
    const_iterator first2 = x.begin();
    const_iterator last2 = x.end();
    // 直到两个链表有一个空间用尽
    while (first1 != last1 && first2 != last2) 
    	*first1++ = *first2++;
    //	原链表大, 复制完后要删除掉原链表多余的元素
    if (first2 == last2)
      erase(first1, last1);
    //  原链表小, 复制完后要还要将x链表的剩余元素以插入的方式插入到原链表中
    else
      insert(last1, first2, last2);
  }
  return *this;
}
```



### resize操作

`resize`重新修改`list`的大小

```c++
template <class T, class Alloc = alloc>
class list 
{
    ...
    resize(size_type new_size, const T& x);
	void resize(size_type new_size) { resize(new_size, T()); }
    ...
}; 
template <class T, class Alloc>
void list<T, Alloc>::resize(size_type new_size, const T& x)
{
  iterator i = begin();
  size_type len = 0;
  for ( ; i != end() && len < new_size; ++i, ++len)
    ;
  // 如果链表长度大于new_size的大小, 那就删除后面多余的节点
  if (len == new_size)
    erase(i, end());
    // i == end(), 扩大链表的节点
  else                          
    insert(end(), new_size - len, x);
}
```



### unique操作

**`unique`函数是将数值相同且连续的元素删除, 只保留一个副本.** 记住, `unique`*并不是删除所有的相同元素, 而是连续的相同元素, 如果要删除所有相同元素就要对list做一个排序在进行unique操作.* 

一般unique同sort一起用的. `sort`函数准备放在下一节来分析.

```c++
template <class T, class Alloc> template <class BinaryPredicate>
void list<T, Alloc>::unique(BinaryPredicate binary_pred) {
  iterator first = begin();
  iterator last = end();
  if (first == last) return;
  iterator next = first;
  // 删除连续相同的元素, 留一个副本
  while (++next != last) {
    if (binary_pred(*first, *next))
      erase(next);
    else
      first = next;
    next = first;
  }
}
```



### insert操作

`insert`函数有很多的重载函数, 满足足够用户的各种插入方法了. 但是最核心的还是`iterator insert(iterator position, const T& x)`, 每一个重载函数都是直接或间接的调用该函数.

<font color=#b20>`insert`是将元素插入到指定地址的前一个位置.</font>

```c++
template <class T, class Alloc = alloc>
class list 
{
    ...
public:
    // 最基本的insert操作, 之插入一个元素
    iterator insert(iterator position, const T& x) 
    {
        // 将元素插入指定位置的前一个地址
      link_type tmp = create_node(x);
      tmp->next = position.node;
      tmp->prev = position.node->prev;
      (link_type(position.node->prev))->next = tmp;
      position.node->prev = tmp;
      return tmp;
    }
    
   // 以下重载函数都是调用iterator insert(iterator position, const T& x)函数
   iterator insert(iterator position) { return insert(position, T()); }
#ifdef __STL_MEMBER_TEMPLATES
    template <class InputIterator>
      void insert(iterator position, InputIterator first, InputIterator last);
#else /* __STL_MEMBER_TEMPLATES */
    void insert(iterator position, const T* first, const T* last);
    void insert(iterator position,
        const_iterator first, const_iterator last);
#endif /* __STL_MEMBER_TEMPLATES */
    void insert(iterator pos, size_type n, const T& x);
    void insert(iterator pos, int n, const T& x) {
      insert(pos, (size_type)n, x);
    }
    void insert(iterator pos, long n, const T& x) {
      insert(pos, (size_type)n, x);
    }
    void resize(size_type new_size, const T& x);
    ...
};

#ifdef __STL_MEMBER_TEMPLATES
template <class T, class Alloc> template <class InputIterator>
void list<T, Alloc>::insert(iterator position, InputIterator first, InputIterator last) 
{
  	for ( ; first != last; ++first)
    	insert(position, *first);
}
#else /* __STL_MEMBER_TEMPLATES */
template <class T, class Alloc>
void list<T, Alloc>::insert(iterator position, const T* first, const T* last) {
  for ( ; first != last; ++first)
    insert(position, *first);
}
template <class T, class Alloc>
void list<T, Alloc>::insert(iterator position,
    const_iterator first, const_iterator last) {
  for ( ; first != last; ++first)
    insert(position, *first);
}
#endif /* __STL_MEMBER_TEMPLATES */
template <class T, class Alloc>
void list<T, Alloc>::insert(iterator position, size_type n, const T& x) {
  for ( ; n > 0; --n)
    insert(position, x);
}
```





## 总结

本节分析了`list`的插入, 删除, 重载等操作, 这些都是链表的基本操作,  相信大家看的时候应该也没有什么问题, 我将最难的部分--`sort`等操作放在下一节来分析.

这里还是提醒一下: 

1.  **节点实际是以`node`空节点开始的**
2.  **插入操作是将元素插入到指定位置的前一个地址进行插入的.**