# deque

## 前言

前一节我们分析了`deque`的基本使用, 本节我们来分析一下`deque`的对`map`的操作, 即插入, 删除等. 但是本节只分析push, pop和删除操作, 而`insert`操作有点复杂还是放到下节来分析.



### push, pop

因为`deque`的是能够双向操作, 所以其push和pop操作都类似于`list`都可以直接有对应的操作. 需要注意的是`list`是链表, 并不会涉及到界线的判断, 而`deque`是由数组来存储的, 就需要随时对界线进行判断.



**push实现**.  

```c++
template <class T, class Alloc = alloc, size_t BufSiz = 0> 
class deque {
    ...
public:                         // push_* and pop_*
    // 对尾进行插入
    // 判断函数是否达到了数组尾部. 没有达到就直接进行插入
  void push_back(const value_type& t) {
    if (finish.cur != finish.last - 1) {
      construct(finish.cur, t);
      ++finish.cur;
    }
    else
      push_back_aux(t);
  }
    // 对头进行插入
    // 判断函数是否达到了数组头部. 没有达到就直接进行插入
  void push_front(const value_type& t) {
    if (start.cur != start.first) {
      construct(start.cur - 1, t);
      --start.cur;
    }
    else
      push_front_aux(t);
  }
    ...
};
```

如果判断数组越界, 就移动到另一个数组进行push操作. 

注意 : `push_back`是先执行构造在移动node, 而`push_front`是先移动node在进行构造. 实现的差异主要是`finish`是指向最后一个元素的后一个地址而`first`指向的就只第一个元素的地址. 下面pop也是一样的.

```c++
// Called only if finish.cur == finish.last - 1.
// 到达了数组的尾部
template <class T, class Alloc, size_t BufSize>
void deque<T, Alloc, BufSize>::push_back_aux(const value_type& t) {
  value_type t_copy = t;
  reserve_map_at_back();
  // 申请空间
  *(finish.node + 1) = allocate_node();
  __STL_TRY {
  	// 执行构造
    construct(finish.cur, t_copy);
    // 移动node, 指向下一个数组的头
    finish.set_node(finish.node + 1);
    finish.cur = finish.first;	// cur只指向当前数组的头
  }
  // 如果分配失败, 释放掉该内存
  __STL_UNWIND(deallocate_node(*(finish.node + 1)));
}

// Called only if start.cur == start.first.
template <class T, class Alloc, size_t BufSize>
void deque<T, Alloc, BufSize>::push_front_aux(const value_type& t) {
  value_type t_copy = t;
  reserve_map_at_front();
  // 申请空间
  *(start.node - 1) = allocate_node();
  __STL_TRY {
  	// 先要移动node, 让其指向上一个数组的尾部
    start.set_node(start.node - 1);
    // cur指向当前数组的尾部
    start.cur = start.last - 1;
    // 执行构造
    construct(start.cur, t_copy);
  }
#     ifdef __STL_USE_EXCEPTIONS
  catch(...) {
    start.set_node(start.node + 1);
    start.cur = start.first;
    deallocate_node(*(start.node - 1));
    throw;
  }
#     endif /* __STL_USE_EXCEPTIONS */
} 
```



**pop实现**. 

```c++
template <class T, class Alloc = alloc, size_t BufSiz = 0> 
class deque {
    ...
public: 
    // 对尾部进行操作
    // 判断是否达到数组的头部. 没有到达就直接释放
    void pop_back() {
    if (finish.cur != finish.first) {
      --finish.cur;
      destroy(finish.cur);
    }
    else
      pop_back_aux();
  }
    // 对头部进行操作
    // 判断是否达到数组的尾部. 没有到达就直接释放
  void pop_front() {
    if (start.cur != start.last - 1) {
      destroy(start.cur);
      ++start.cur;
    }
    else 
      pop_front_aux();
  }
    ...
};
```

pop判断越界后执行以下函数. 

```c++
// Called only if finish.cur == finish.first.
template <class T, class Alloc, size_t BufSize>
void deque<T, Alloc, BufSize>:: pop_back_aux() {
  deallocate_node(finish.first);	// 先调用析构函数
  finish.set_node(finish.node - 1);	// 再移动node
  finish.cur = finish.last - 1;		// 然后cur指向当前数组的最后位置
  destroy(finish.cur);				// 最后释放内存空间.
}

// Called only if start.cur == start.last - 1.  Note that if the deque
//  has at least one element (a necessary precondition for this member
//  function), and if start.cur == start.last, then the deque must have
//  at least two nodes.
template <class T, class Alloc, size_t BufSize>
void deque<T, Alloc, BufSize>::pop_front_aux() {
  destroy(start.cur);				// 先释放内存空间.
  deallocate_node(start.first);		// 再调用析构函数
  start.set_node(start.node + 1);	// 然后移动node
  start.cur = start.first;			// 最后cur指向当前数组的第一个位置
}   	
```



**reserve_map_at一类函数**.  pop和push都先调用了reserve_map_at_XX函数, 这些函数主要是为了**判断前后空间是否足够**.

```c++
template <class T, class Alloc = alloc, size_t BufSiz = 0> 
class deque {
    ...
  public:
  void new_elements_at_front(size_type new_elements);
  void new_elements_at_back(size_type new_elements);

  void destroy_nodes_at_front(iterator before_start);
  void destroy_nodes_at_back(iterator after_finish);

protected:                      // Allocation of map and nodes
  // Makes sure the map has space for new nodes.  Does not actually
  //  add the nodes.  Can invalidate map pointers.  (And consequently, 
  //  deque iterators.)
	// 始终保证后面要有一个及以上的空数组大小
  void reserve_map_at_back (size_type nodes_to_add = 1) {
    if (nodes_to_add + 1 > map_size - (finish.node - map))
      reallocate_map(nodes_to_add, false);
  }
	// 始终保证前面要有一个及以上的空数组大小
  void reserve_map_at_front (size_type nodes_to_add = 1) {
    if (nodes_to_add > start.node - map)
      reallocate_map(nodes_to_add, true);
  }

  void reallocate_map(size_type nodes_to_add, bool add_at_front);
    ...
};
```

**reallocate_map**函数, 空间不足

1.  deque空间实际足够
    1.  deque内部进行调整start, 和finish
2.  deque空间真的不足
    1.  申请更大的空间
    2.  拷贝元素过去
    3.  修改map和start, finish指向

```c++
template <class T, class Alloc, size_t BufSize>
void deque<T, Alloc, BufSize>::reallocate_map(size_type nodes_to_add, bool add_at_front) 
{
    // 保存现在的空间大小和新的空间大小
  size_type old_num_nodes = finish.node - start.node + 1;
  size_type new_num_nodes = old_num_nodes + nodes_to_add;

  map_pointer new_nstart;
    // map_size > 2 * new_num_nodes 发现deque空间还很充足就只是调整deque内部的元素就行了, 没必要重新开空间
    // 这种情况主要出现在一直往首或尾单方向插入元素, 导致首(尾)前面还有很多余留的空间, 这种情况就这样调整
  if (map_size > 2 * new_num_nodes) {
    new_nstart = map + (map_size - new_num_nodes) / 2 + (add_at_front ? nodes_to_add : 0);
    if (new_nstart < start.node)
      copy(start.node, finish.node + 1, new_nstart);
    else
      copy_backward(start.node, finish.node + 1, new_nstart + old_num_nodes);
  }
    // 空间是真的不够了
  else {
    size_type new_map_size = map_size + max(map_size, nodes_to_add) + 2;
	// 分配空间. 重新定位start的位置
    map_pointer new_map = map_allocator::allocate(new_map_size);
    new_nstart = new_map + (new_map_size - new_num_nodes) / 2 + (add_at_front ? nodes_to_add : 0);
      // 拷贝原deque元素, 最后释放掉原内存空间
    copy(start.node, finish.node + 1, new_nstart);
    map_allocator::deallocate(map, map_size);
	
      // 调整map
    map = new_map;
    map_size = new_map_size;
  }
	// 重新调整start, finish
  start.set_node(new_nstart);
  finish.set_node(new_nstart + old_num_nodes - 1);
}
```



```c++
template <class T, class Alloc, size_t BufSize>
void deque<T, Alloc, BufSize>::destroy_nodes_at_front(iterator before_start) {
  for (map_pointer n = before_start.node; n < start.node; ++n)
    deallocate_node(*n);
}

template <class T, class Alloc, size_t BufSize>
void deque<T, Alloc, BufSize>::destroy_nodes_at_back(iterator after_finish) {
  for (map_pointer n = after_finish.node; n > finish.node; --n)
    deallocate_node(*n);
}
```



```c++
template <class T, class Alloc = alloc, size_t BufSiz = 0> 
class deque {
    ...
public:  
  iterator reserve_elements_at_front(size_type n) {
    size_type vacancies = start.cur - start.first;
    if (n > vacancies) 
      new_elements_at_front(n - vacancies);
    return start - difference_type(n);
  }

  iterator reserve_elements_at_back(size_type n) {
    size_type vacancies = (finish.last - finish.cur) - 1;
    if (n > vacancies)
      new_elements_at_back(n - vacancies);
    return finish + difference_type(n);
  }
    ...
};
template <class T, class Alloc, size_t BufSize>
void deque<T, Alloc, BufSize>::new_elements_at_front(size_type new_elements) {
  size_type new_nodes = (new_elements + buffer_size() - 1) / buffer_size();
  reserve_map_at_front(new_nodes);
  size_type i;
  __STL_TRY {
    for (i = 1; i <= new_nodes; ++i)
      *(start.node - i) = allocate_node();
  }
#       ifdef __STL_USE_EXCEPTIONS
  catch(...) {
    for (size_type j = 1; j < i; ++j)
      deallocate_node(*(start.node - j));      
    throw;
  }
#       endif /* __STL_USE_EXCEPTIONS */
}

template <class T, class Alloc, size_t BufSize>
void deque<T, Alloc, BufSize>::new_elements_at_back(size_type new_elements) {
  size_type new_nodes = (new_elements + buffer_size() - 1) / buffer_size();
  reserve_map_at_back(new_nodes);
  size_type i;
  __STL_TRY {
    for (i = 1; i <= new_nodes; ++i)
      *(finish.node + i) = allocate_node();
  }
#       ifdef __STL_USE_EXCEPTIONS
  catch(...) {
    for (size_type j = 1; j < i; ++j)
      deallocate_node(*(finish.node + j));      
    throw;
  }
#       endif /* __STL_USE_EXCEPTIONS */
}
```





### 删除操作

不知道还记得我们最开始构造函数调用`create_map_and_nodes`考虑到`deque`实现前后插入时间复杂度为O(1), 保证了在前后留出了空间, 所以push和pop都可以在前面的数组进行操作.

好了, 现在就来看erase. 因为deque的是由数组构成, 所以地址空间是连续的. 删除也就像`vector`一样, 要移动所有的元素, `deque`为了保证效率尽量高, 就判断删除的位置是中间偏后还是中间偏前来进行移动.

```c++
template <class T, class Alloc = alloc, size_t BufSiz = 0> 
class deque {
    ...
public:                         // Erase
  iterator erase(iterator pos) 
  {
    iterator next = pos;
    ++next;
    difference_type index = pos - start;
      // 删除的地方是中间偏前, 移动前面的元素
    if (index < (size() >> 1)) 
    {
      copy_backward(start, pos, next);
      pop_front();
    }
      // 删除的地方是中间偏后, 移动后面的元素
    else {
      copy(next, finish, pos);
      pop_back();
    }
    return start + index;
  }
	// 范围删除, 实际也是调用上面的erase函数.
  iterator erase(iterator first, iterator last);
  void clear(); 
    ...
};
```

**erase(iterator first, iterator last)**

```c++

template <class T, class Alloc, size_t BufSize>
deque<T, Alloc, BufSize>::iterator 
deque<T, Alloc, BufSize>::erase(iterator first, iterator last) 
{
  if (first == start && last == finish) {
    clear();
    return finish;
  }
  else {
      // 计算出两个迭代器的距离, 毕竟是连续的, 可以直接计算
    difference_type n = last - first;
      // 同样, 选择前后哪种方法移动.
    difference_type elems_before = first - start;
      // 删除的地方是中间偏前, 移动前面的元素
    if (elems_before < (size() - n) / 2) {
      copy_backward(start, first, last);
      iterator new_start = start + n;
      destroy(start, new_start);
        // 可能会涉及到跨数组的问题(用户使用并不知道)
      for (map_pointer cur = start.node; cur < new_start.node; ++cur)
        data_allocator::deallocate(*cur, buffer_size());
      start = new_start;
    }
      // 删除的地方是中间偏后, 移动后面的元素
    else {
      copy(last, finish, first);
      iterator new_finish = finish - n;
      destroy(new_finish, finish);
        // 可能会涉及到跨数组的问题(用户使用并不知道)
      for (map_pointer cur = new_finish.node + 1; cur <= finish.node; ++cur)
        data_allocator::deallocate(*cur, buffer_size());
      finish = new_finish;
    }
    return start + elems_before;
  }
}
```

**clear函数**. 删除所有元素. 分两步执行:

1.  从第二个数组开始到倒数第二个数组一次性全部删除, 毕竟中间的数组肯定都是满的, 前后两个数组就不一定是填充满的.
2.  删除前后两个数组的元素

```c++
template <class T, class Alloc, size_t BufSize>
void deque<T, Alloc, BufSize>::clear() {
	// 从第二个数组开始到倒数第二个数组一次性全部删除
	// 毕竟中间的数组肯定都是满的, 前后两个数组就不一定是填充满的.
  for (map_pointer node = start.node + 1; node < finish.node; ++node) {
    destroy(*node, *node + buffer_size());
    data_allocator::deallocate(*node, buffer_size());
  }
	// 删除前后两个数组的元素.
  if (start.node != finish.node) {
    destroy(start.cur, start.last);
    destroy(finish.first, finish.cur);
    data_allocator::deallocate(finish.first, buffer_size());
  }
  else
    destroy(start.cur, finish.cur);

  finish = start;
}
```



### swap

`deque`的swap操作也只是交换了start, finish, map, 并没有交换所有的元素.

```c++
template <class T, class Alloc = alloc, size_t BufSiz = 0> 
class deque {
        ...
    	void swap(deque& x)
        {
        	__STD::swap(start, x.start);
        	__STD::swap(finish, x.finish);
        	__STD::swap(map, x.map);
        	__STD::swap(map_size, x.map_size);
      	}
        ...
};
template <class T, class Alloc, size_t BufSiz>
inline void swap(deque<T, Alloc, BufSiz>& x, deque<T, Alloc, BufSiz>& y) {
  x.swap(y);
}
```



### resize函数

**resize函数**. 重新将`deque`进行调整, 实现与`list`一样的.

```c++
template <class T, class Alloc = alloc, size_t BufSiz = 0> 
class deque {
    ...
public: 
    void resize(size_type new_size) { resize(new_size, value_type()); }
    void resize(size_type new_size, const value_type& x) {
    const size_type len = size();
    // 元素大小大于了要修改的大小, 则释放掉超过的元素
    if (new_size < len) 
      erase(start + new_size, finish);
    // 元素不够, 就从end开始到要求的大小为止都初始化x
    else
      insert(finish, new_size - len, x);
  }
};
```





## 总结

本节又分析了很多`deque`操作的实现, push和pop都是经常会用的操作, 下一节我们分析关于`deque`实现insert操作的.

