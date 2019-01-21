# deque

## 前言

前面两节对`deque`基本所有的操作都分析了, 本节就分析`deque`的insert操作的实现. insert的重载函数有很多, 所以没有在上节一起分析, 本节也只是对部分重载函数进行分析, 剩下的列出源码就行了.



## 源码分析

### insert实现

这里先将`insert`的所有重载函数进行罗列.

```c++
iterator insert(iterator position, const value_type& x);
iterator insert(iterator position) ;

// 调用相同的重载函数
void insert(iterator pos, size_type n, const value_type& x);
void insert(iterator pos, int n, const value_type& x);
void insert(iterator pos, long n, const value_type& x);

void insert(iterator pos, InputIterator first, InputIterator last);
void insert(iterator pos, const value_type* first, const value_type* last);
void insert(iterator pos, const_iterator first, const_iterator last);

void insert(iterator pos, InputIterator first, InputIterator last, input_iterator_tag);
void insert(iterator pos, ForwardIterator first, ForwardIterator last,forward_iterator_tag);
```



**iterator insert(iterator position, const value_type& x)** 

```c++
template <class T, class Alloc = alloc, size_t BufSiz = 0> 
class deque {
    ...
public:                         // Insert
  iterator insert(iterator position, const value_type& x) {
      // 如果只是在头尾插入, 直接调用push就行了.
    if (position.cur == start.cur) {
      push_front(x);
      return start;
    }
    else if (position.cur == finish.cur) {
      push_back(x);
      iterator tmp = finish;
      --tmp;
      return tmp;
    }
      // 随机插入
    else {
      return insert_aux(position, x);
    }
  }
};
```



**insert(iterator pos, size_type n, const value_type& x) ** 在指定的位置插入n个元素并初始化.

```c++
template <class T, class Alloc, size_t BufSize>
void deque<T, Alloc, BufSize>::insert(iterator pos, size_type n, const value_type& x) 
{
    // 同样判断是不是直接在头尾进行插入.
  if (pos.cur == start.cur) {
      // 判断还有没有足够的空间
    iterator new_start = reserve_elements_at_front(n);
    uninitialized_fill(new_start, start, x); // 范围初始化
    start = new_start;	// 修改start位置
  }
  else if (pos.cur == finish.cur) {
    iterator new_finish = reserve_elements_at_back(n);	// 判断还有没有足够的空间
    uninitialized_fill(finish, new_finish, x);	// 范围初始化
    finish = new_finish;	// 修改finish位置
  }	
    // 随机插入
  else 
    insert_aux(pos, n, x);
}
  void insert(iterator pos, int n, const value_type& x) {
    insert(pos, (size_type) n, x);
  }
  void insert(iterator pos, long n, const value_type& x) {
    insert(pos, (size_type) n, x);
  }
```

**void insert(iterator pos, InputIterator first, InputIterator last) **. 通过参数的类型选择最优, 高效率的插入方式.

```c++
template <class InputIterator>
void insert(iterator pos, InputIterator first, InputIterator last) 
{
    insert(pos, first, last, iterator_category(first));
}

// input_iterator_tag类型的迭代器
template <class T, class Alloc, size_t BufSize>
template <class InputIterator>
void deque<T, Alloc, BufSize>::insert(iterator pos,InputIterator first, InputIterator last,
input_iterator_tag) 
{
  copy(first, last, inserter(*this, pos));	// 直接调用copy函数
}

// forward_iterator_tag类型的迭代器
template <class T, class Alloc, size_t BufSize>
template <class ForwardIterator>
void deque<T, Alloc, BufSize>::insert(iterator pos,ForwardIterator first,ForwardIterator last,forward_iterator_tag) 
{
  size_type n = 0;
  distance(first, last, n); // 计算迭代器之间的距离
    // 同样, 首尾插入判断
  if (pos.cur == start.cur) {
    iterator new_start = reserve_elements_at_front(n);
    __STL_TRY {
      uninitialized_copy(first, last, new_start);
      start = new_start;
    }
    __STL_UNWIND(destroy_nodes_at_front(new_start));
  }
  else if (pos.cur == finish.cur) {
    iterator new_finish = reserve_elements_at_back(n);
    __STL_TRY {
      uninitialized_copy(first, last, finish);
      finish = new_finish;
    }
    __STL_UNWIND(destroy_nodes_at_back(new_finish));
  }
    // 随机插入
  else
    insert_aux(pos, first, last, n);
}
```





```c++
template <class T, class Alloc, size_t BufSize>
void deque<T, Alloc, BufSize>::insert(iterator pos,const value_type* first,const value_type* last) 
{
  size_type n = last - first;
  if (pos.cur == start.cur) {
    iterator new_start = reserve_elements_at_front(n);
    __STL_TRY {
      uninitialized_copy(first, last, new_start);
      start = new_start;
    }
    __STL_UNWIND(destroy_nodes_at_front(new_start));
  }
  else if (pos.cur == finish.cur) {
    iterator new_finish = reserve_elements_at_back(n);
    __STL_TRY {
      uninitialized_copy(first, last, finish);
      finish = new_finish;
    }
    __STL_UNWIND(destroy_nodes_at_back(new_finish));
  }
  else
    insert_aux(pos, first, last, n);
}

template <class T, class Alloc, size_t BufSize>
void deque<T, Alloc, BufSize>::insert(iterator pos,const_iterator first,const_iterator last)
{
  size_type n = last - first;
  if (pos.cur == start.cur) {
    iterator new_start = reserve_elements_at_front(n);
    __STL_TRY {
      uninitialized_copy(first, last, new_start);
      start = new_start;
    }
    __STL_UNWIND(destroy_nodes_at_front(new_start));
  }
  else if (pos.cur == finish.cur) {
    iterator new_finish = reserve_elements_at_back(n);
    __STL_TRY {
      uninitialized_copy(first, last, finish);
      finish = new_finish;
    }
    __STL_UNWIND(destroy_nodes_at_back(new_finish));
  }
  else
    insert_aux(pos, first, last, n);
}
```



### insert_auto

上面对insert函数做了简单的分析, 可以发现基本每一个`insert`重载函数都会调用了`insert_auto`, 现在我们就来分析该函数的实现.



**insert_aux(iterator pos, const value_type& x) **. 

```c++
template <class T, class Alloc, size_t BufSize>
typename deque<T, Alloc, BufSize>::iterator
deque<T, Alloc, BufSize>::insert_aux(iterator pos, const value_type& x) 
{
  difference_type index = pos - start;
  value_type x_copy = x;
    // 判断插入的位置离头还是尾比较近
    // 离头进
  if (index < size() / 2) {
    push_front(front());	// 将头往前移动
      // 调整将要移动的距离
    iterator front1 = start;
    ++front1;
    iterator front2 = front1;
    ++front2;
    pos = start + index;
    iterator pos1 = pos;
    ++pos1;
      // 用copy进行调整
    copy(front2, pos1, front1);
  }
    // 离尾近
  else {
    push_back(back());	// 将尾往前移动
      // 调整将要移动的距离
    iterator back1 = finish;
    --back1;
    iterator back2 = back1;
    --back2;
    pos = start + index;
      // 用copy进行调整
    copy_backward(pos, back2, back1);
  }
  *pos = x_copy;
  return pos;
}
```



**insert_aux(iterator pos, size_type n, const value_type& x) ** .

```c++
template <class T, class Alloc, size_t BufSize>
void deque<T, Alloc, BufSize>::insert_aux(iterator pos, size_type n, const value_type& x) 
{
  const difference_type elems_before = pos - start;
  size_type length = size();
  value_type x_copy = x;
    // 判断插入的位置离头还是尾比较近
    // 离头近
  if (elems_before < length / 2) {
    iterator new_start = reserve_elements_at_front(n);	// 新的内存空间
    iterator old_start = start;
      // 计算pos的新位置
    pos = start + elems_before;
    __STL_TRY {
        // 到头的距离大于插入的个数n
      if (elems_before >= difference_type(n)) {
          // 一部分一部分的进行调整
        iterator start_n = start + difference_type(n);
        uninitialized_copy(start, start_n, new_start);
        start = new_start;
        copy(start_n, pos, old_start);
        fill(pos - difference_type(n), pos, x_copy);
      }
        // 到头的距离不大于插入的个数n
      else {
        __uninitialized_copy_fill(start, pos, new_start, start, x_copy);
        start = new_start;
        fill(old_start, pos, x_copy);
      }
    }
    __STL_UNWIND(destroy_nodes_at_front(new_start));
  }
    // 离尾近. 执行都是一样的
  else {
    iterator new_finish = reserve_elements_at_back(n);
    iterator old_finish = finish;
    const difference_type elems_after = difference_type(length) - elems_before;
    pos = finish - elems_after;
    __STL_TRY {
      if (elems_after > difference_type(n)) {
        iterator finish_n = finish - difference_type(n);
        uninitialized_copy(finish_n, finish, finish);
        finish = new_finish;
        copy_backward(pos, finish_n, old_finish);
        fill(pos, pos + difference_type(n), x_copy);
      }
      else {
        __uninitialized_fill_copy(finish, pos + difference_type(n),
                                  x_copy,
                                  pos, finish);
        finish = new_finish;
        fill(pos, old_finish, x_copy);
      }
    }
    __STL_UNWIND(destroy_nodes_at_back(new_finish));
  }
}
```

剩下的就不一一进行分析了, 基本要涉及到的操作前面都已经讲的很明白了.

```c++
#ifdef __STL_MEMBER_TEMPLATES  
template <class T, class Alloc, size_t BufSize>
template <class ForwardIterator>
void deque<T, Alloc, BufSize>::insert_aux(iterator pos,
                                          ForwardIterator first,
                                          ForwardIterator last,
                                          size_type n)
{
  const difference_type elems_before = pos - start;
  size_type length = size();
  if (elems_before < length / 2) {
    iterator new_start = reserve_elements_at_front(n);
    iterator old_start = start;
    pos = start + elems_before;
    __STL_TRY {
      if (elems_before >= difference_type(n)) {
        iterator start_n = start + difference_type(n); 
        uninitialized_copy(start, start_n, new_start);
        start = new_start;
        copy(start_n, pos, old_start);
        copy(first, last, pos - difference_type(n));
      }
      else {
        ForwardIterator mid = first;
        advance(mid, difference_type(n) - elems_before);
        __uninitialized_copy_copy(start, pos, first, mid, new_start);
        start = new_start;
        copy(mid, last, old_start);
      }
    }
    __STL_UNWIND(destroy_nodes_at_front(new_start));
  }
  else {
    iterator new_finish = reserve_elements_at_back(n);
    iterator old_finish = finish;
    const difference_type elems_after = difference_type(length) - elems_before;
    pos = finish - elems_after;
    __STL_TRY {
      if (elems_after > difference_type(n)) {
        iterator finish_n = finish - difference_type(n);
        uninitialized_copy(finish_n, finish, finish);
        finish = new_finish;
        copy_backward(pos, finish_n, old_finish);
        copy(first, last, pos);
      }
      else {
        ForwardIterator mid = first;
        advance(mid, elems_after);
        __uninitialized_copy_copy(mid, last, pos, finish, finish);
        finish = new_finish;
        copy(first, mid, pos);
      }
    }
    __STL_UNWIND(destroy_nodes_at_back(new_finish));
  }
}

#else /* __STL_MEMBER_TEMPLATES */

template <class T, class Alloc, size_t BufSize>
void deque<T, Alloc, BufSize>::insert_aux(iterator pos,
                                          const value_type* first,
                                          const value_type* last,
                                          size_type n)
{
  const difference_type elems_before = pos - start;
  size_type length = size();
  if (elems_before < length / 2) {
    iterator new_start = reserve_elements_at_front(n);
    iterator old_start = start;
    pos = start + elems_before;
    __STL_TRY {
      if (elems_before >= difference_type(n)) {
        iterator start_n = start + difference_type(n);
        uninitialized_copy(start, start_n, new_start);
        start = new_start;
        copy(start_n, pos, old_start);
        copy(first, last, pos - difference_type(n));
      }
      else {
        const value_type* mid = first + (difference_type(n) - elems_before);
        __uninitialized_copy_copy(start, pos, first, mid, new_start);
        start = new_start;
        copy(mid, last, old_start);
      }
    }
    __STL_UNWIND(destroy_nodes_at_front(new_start));
  }
  else {
    iterator new_finish = reserve_elements_at_back(n);
    iterator old_finish = finish;
    const difference_type elems_after = difference_type(length) - elems_before;
    pos = finish - elems_after;
    __STL_TRY {
      if (elems_after > difference_type(n)) {
        iterator finish_n = finish - difference_type(n);
        uninitialized_copy(finish_n, finish, finish);
        finish = new_finish;
        copy_backward(pos, finish_n, old_finish);
        copy(first, last, pos);
      }
      else {
        const value_type* mid = first + elems_after;
        __uninitialized_copy_copy(mid, last, pos, finish, finish);
        finish = new_finish;
        copy(first, mid, pos);
      }
    }
    __STL_UNWIND(destroy_nodes_at_back(new_finish));
  }
}

template <class T, class Alloc, size_t BufSize>
void deque<T, Alloc, BufSize>::insert_aux(iterator pos,
                                          const_iterator first,
                                          const_iterator last,
                                          size_type n)
{
  const difference_type elems_before = pos - start;
  size_type length = size();
  if (elems_before < length / 2) {
    iterator new_start = reserve_elements_at_front(n);
    iterator old_start = start;
    pos = start + elems_before;
    __STL_TRY {
      if (elems_before >= n) {
        iterator start_n = start + n;
        uninitialized_copy(start, start_n, new_start);
        start = new_start;
        copy(start_n, pos, old_start);
        copy(first, last, pos - difference_type(n));
      }
      else {
        const_iterator mid = first + (n - elems_before);
        __uninitialized_copy_copy(start, pos, first, mid, new_start);
        start = new_start;
        copy(mid, last, old_start);
      }
    }
    __STL_UNWIND(destroy_nodes_at_front(new_start));
  }
  else {
    iterator new_finish = reserve_elements_at_back(n);
    iterator old_finish = finish;
    const difference_type elems_after = length - elems_before;
    pos = finish - elems_after;
    __STL_TRY {
      if (elems_after > n) {
        iterator finish_n = finish - difference_type(n);
        uninitialized_copy(finish_n, finish, finish);
        finish = new_finish;
        copy_backward(pos, finish_n, old_finish);
        copy(first, last, pos);
      }
      else {
        const_iterator mid = first + elems_after;
        __uninitialized_copy_copy(mid, last, pos, finish, finish);
        finish = new_finish;
        copy(first, mid, pos);
      }
    }
    __STL_UNWIND(destroy_nodes_at_back(new_finish));
  }
}
```



## 总结

本节分析了一部分关于`insert`的源码, `deque`的也暂时分析完了, 下节就开始分析其他的序列容器了.