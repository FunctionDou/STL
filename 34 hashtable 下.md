# hashtable

### 前言

上节分析了hashtable的迭代器, 其构造函数, 基本函数, 本节我们分析hashtable的插入, 删除等操作.



### hashtable分析

**重载**

```c++
template <class Value, class Key, class HashFcn, class ExtractKey, class EqualKey, class Alloc>
class hashtable {
	...
public:
  hashtable& operator= (const hashtable& ht)
  {
    if (&ht != this) {
      clear();	// 清除原表中的数据
      // 重新进行赋值
      hash = ht.hash;
      equals = ht.equals;
      get_key = ht.get_key;
      copy_from(ht);
    }
    return *this;
  }
  friend bool
  operator== __STL_NULL_TMPL_ARGS (const hashtable&, const hashtable&);
  ...
};

// 重载++
template <class V, class K, class HF, class ExK, class EqK, class A>
__hashtable_iterator<V, K, HF, ExK, EqK, A>&
__hashtable_iterator<V, K, HF, ExK, EqK, A>::operator++()
{
  const node* old = cur;
  cur = cur->next;
    // cur指向了NULL
  if (!cur) {
    size_type bucket = ht->bkt_num(old->val);
      // 寻找桶中下一个链表不为空的链表的第一个元素
    while (!cur && ++bucket < ht->buckets.size())
      cur = ht->buckets[bucket];
  }
  return *this;
}

template <class V, class K, class HF, class ExK, class EqK, class A>
inline __hashtable_iterator<V, K, HF, ExK, EqK, A>
__hashtable_iterator<V, K, HF, ExK, EqK, A>::operator++(int)
{
  iterator tmp = *this;
  ++*this;
  return tmp;
}

template <class V, class K, class HF, class Ex, class Eq, class A>
bool operator==(const hashtable<V, K, HF, Ex, Eq, A>& ht1,
                const hashtable<V, K, HF, Ex, Eq, A>& ht2)
{
  typedef typename hashtable<V, K, HF, Ex, Eq, A>::node node;
    // 先判断桶的大小
  if (ht1.buckets.size() != ht2.buckets.size())
    return false;
    // 其次比较桶中每个指向的链表
  for (int n = 0; n < ht1.buckets.size(); ++n) {
    node* cur1 = ht1.buckets[n];
    node* cur2 = ht2.buckets[n];
      // 比较链表中的元素也是否相等
    for ( ; cur1 && cur2 && cur1->val == cur2->val;
          cur1 = cur1->next, cur2 = cur2->next)
      {}
      // 有一个链表还有剩余的元素就表示不相等
    if (cur1 || cur2)
      return false;
  }
  return true;
}  
```



**插入**

hashtable也有`insert_equal`和`insert_unqiue`两种插入方式, 而不可重复插入返回的是pair结构, 可重复插入返回的是迭代器. 

同时兼顾了`ForwardIterator`和`InputIterator`的类型迭代器

```c++
template <class Value, class Key, class HashFcn, class ExtractKey, class EqualKey, class Alloc>
class hashtable {
	...
public: 
	// 不可重复插入
  pair<iterator, bool> insert_unique(const value_type& obj)
  {
    resize(num_elements + 1);
    return insert_unique_noresize(obj);
  }
	// 可重复插入
  iterator insert_equal(const value_type& obj)
  {
    resize(num_elements + 1);
    return insert_equal_noresize(obj);
  }
	// 不可重复插入返回的是pair结构
  pair<iterator, bool> insert_unique_noresize(const value_type& obj);
  	// 可重复插入返回的是迭代器
  iterator insert_equal_noresize(const value_type& obj);
 
 // 以下是insert的各个重载函数
#ifdef __STL_MEMBER_TEMPLATES
	// 针对InputIterator的迭代器
  template <class InputIterator>
  void insert_unique(InputIterator f, InputIterator l)
  {
    insert_unique(f, l, iterator_category(f));
  }
  template <class InputIterator>
  void insert_equal(InputIterator f, InputIterator l)
  {
    insert_equal(f, l, iterator_category(f));
  }
  template <class InputIterator>
  void insert_unique(InputIterator f, InputIterator l,input_iterator_tag)
  {
    for ( ; f != l; ++f)
      insert_unique(*f);
  }
  template <class InputIterator>
  void insert_equal(InputIterator f, InputIterator l,input_iterator_tag)
  {
    for ( ; f != l; ++f)
      insert_equal(*f);
  }
  // 针对ForwardIterator类型的迭代器, 一个个进行插入
  template <class ForwardIterator>
  void insert_unique(ForwardIterator f, ForwardIterator l,forward_iterator_tag)
  {
    size_type n = 0;
    distance(f, l, n);
    resize(num_elements + n);
    for ( ; n > 0; --n, ++f)
      insert_unique_noresize(*f);
  }
  template <class ForwardIterator>
  void insert_equal(ForwardIterator f, ForwardIterator l,forward_iterator_tag)
  {
    size_type n = 0;
    distance(f, l, n);
    resize(num_elements + n);
    for ( ; n > 0; --n, ++f)
      insert_equal_noresize(*f);
  }
#else /* __STL_MEMBER_TEMPLATES */
  void insert_unique(const value_type* f, const value_type* l)
  {
    size_type n = l - f;
    resize(num_elements + n);
    for ( ; n > 0; --n, ++f)
      insert_unique_noresize(*f);
  }
  void insert_equal(const value_type* f, const value_type* l)
  {
    size_type n = l - f;
    resize(num_elements + n);
    for ( ; n > 0; --n, ++f)
      insert_equal_noresize(*f);
  }
  void insert_unique(const_iterator f, const_iterator l)
  {
    size_type n = 0;
    distance(f, l, n);
    resize(num_elements + n);
    for ( ; n > 0; --n, ++f)
      insert_unique_noresize(*f);
  }
  void insert_equal(const_iterator f, const_iterator l)
  {
    size_type n = 0;
    distance(f, l, n);
    resize(num_elements + n);
    for ( ; n > 0; --n, ++f)
      insert_equal_noresize(*f);
  }
#endif /*__STL_MEMBER_TEMPLATES */
	...
};
```

`insert_unique_noresize`不可重复插入

```c++
template <class V, class K, class HF, class Ex, class Eq, class A>
pair<typename hashtable<V, K, HF, Ex, Eq, A>::iterator, bool> 
hashtable<V, K, HF, Ex, Eq, A>::insert_unique_noresize(const value_type& obj)
{
    // 确定插入的桶的具体位置
  const size_type n = bkt_num(obj);
  node* first = buckets[n];

    // 将元素插入到链表中
  for (node* cur = first; cur; cur = cur->next) 
      // 判断该元素在链表中是否已经存在了
    if (equals(get_key(cur->val), get_key(obj)))
      return pair<iterator, bool>(iterator(cur, this), false);	// 存在pair第二个参数返回false

    // 元素不存在链表中, 将它插入到链表的头部
  node* tmp = new_node(obj);
  tmp->next = first;
  buckets[n] = tmp;
  ++num_elements;	// 计数++
  return pair<iterator, bool>(iterator(tmp, this), true);	// 返回pair结构
}
```

`insert_equal_noresize`可重复插入

```c++
template <class V, class K, class HF, class Ex, class Eq, class A>
typename hashtable<V, K, HF, Ex, Eq, A>::iterator 
hashtable<V, K, HF, Ex, Eq, A>::insert_equal_noresize(const value_type& obj)
{
    // 确定插入的桶的具体位置
  const size_type n = bkt_num(obj);
  node* first = buckets[n];
	
    // 将元素插入到链表中
  for (node* cur = first; cur; cur = cur->next) 
      // 判断该元素在链表中是否已经存在了, 则将元素插入到重复数据的位置
    if (equals(get_key(cur->val), get_key(obj))) {
      node* tmp = new_node(obj);
      tmp->next = cur->next;
      cur->next = tmp;
      ++num_elements;
      return iterator(tmp, this);
    }

    // 元素不存在链表中, 将它插入到链表的头部
  node* tmp = new_node(obj);
  tmp->next = first;
  buckets[n] = tmp;	
  ++num_elements;	// 计数++
  return iterator(tmp, this);	// 返回pair结构
}
```



**查找**

hashtable的查找分为`find`和`find_or_insert`两种, 前者只是查找, 后者则有和插入的功能

```c++
template <class Value, class Key, class HashFcn, class ExtractKey, class EqualKey, class Alloc>
class hashtable {
	...
public: 
	// 找到指定的桶的位置再在链表中进行遍历
  iterator find(const key_type& key) 
  {
    size_type n = bkt_num_key(key);
    node* first;
    // 找到指定的位置并返回
    for ( first = buckets[n];first && !equals(get_key(first->val), key);  first = first->next)
      {}
    return iterator(first, this);
  } 
  ...
};
```

`find_or_insert`如果找到了就返回该元素的数据, 没有找到指定元素就将其插入到链表的头部.

```c++
template <class V, class K, class HF, class Ex, class Eq, class A>
typename hashtable<V, K, HF, Ex, Eq, A>::reference 
hashtable<V, K, HF, Ex, Eq, A>::find_or_insert(const value_type& obj)
{
  resize(num_elements + 1);

  size_type n = bkt_num(obj);
  node* first = buckets[n];
	// 如果找到了就返回该元素的数据
  for (node* cur = first; cur; cur = cur->next)
    if (equals(get_key(cur->val), get_key(obj)))
      return cur->val;

    // 没有找到指定元素就将其插入到链表的头部
  node* tmp = new_node(obj);
  tmp->next = first;
  buckets[n] = tmp;
  ++num_elements;
  return tmp->val;
}
```



**删除**

erase有很多个重载函数, 这里就具体分析一个就行了.

```c++
template <class Value, class Key, class HashFcn, class ExtractKey, class EqualKey, class Alloc>
class hashtable {
	...
public: 
  size_type erase(const key_type& key);
  void erase(const iterator& it);
  void erase(iterator first, iterator last);

  void erase(const const_iterator& it);
  void erase(const_iterator first, const_iterator last);

  void resize(size_type num_elements_hint);
  void clear();
  ...
};
```

```c++
template <class V, class K, class HF, class Ex, class Eq, class A>
typename hashtable<V, K, HF, Ex, Eq, A>::size_type 
hashtable<V, K, HF, Ex, Eq, A>::erase(const key_type& key)
{
  const size_type n = bkt_num_key(key);
  node* first = buckets[n];
  size_type erased = 0;
	
    // 找到key具体在哪一个链表
  if (first) {
    node* cur = first;
    node* next = cur->next;
      // 遍历链表
    while (next) {
        // 元素在中间
      if (equals(get_key(next->val), key)) {
        cur->next = next->next;
        delete_node(next);
        next = cur->next;
        ++erased;
        --num_elements;
      }
        // 在头部
      else {
        cur = next;
        next = cur->next;
      }
    }
      // 析构, 释放
    if (equals(get_key(first->val), key)) {
      buckets[n] = first->next;
      delete_node(first);
      ++erased;
      --num_elements;
    }
  }
  return erased;
}
```



范围的删除

```c++
template <class V, class K, class HF, class Ex, class Eq, class A>
void 
hashtable<V, K, HF, Ex, Eq, A>::erase_bucket(const size_type n, node* last)
{
  node* cur = buckets[n];
  while (cur != last) {
    node* next = cur->next;
    delete_node(cur);
    cur = next;
    buckets[n] = cur;
    --num_elements;
  }
}

template <class V, class K, class HF, class Ex, class Eq, class A>
void hashtable<V, K, HF, Ex, Eq, A>::clear()
{
  for (size_type i = 0; i < buckets.size(); ++i) {
    node* cur = buckets[i];
    while (cur != 0) {
      node* next = cur->next;
      delete_node(cur);
      cur = next;
    }
    buckets[i] = 0;
  }
  num_elements = 0;
}

```



**复制**

```c++
template <class V, class K, class HF, class Ex, class Eq, class A>
void hashtable<V, K, HF, Ex, Eq, A>::copy_from(const hashtable& ht)
{
  buckets.clear(); //把元素全部删除
  buckets.reserve(ht.buckets.size());	// 重新调整桶的大小
    // 重新插入
  buckets.insert(buckets.end(), ht.buckets.size(), (node*) 0);
  __STL_TRY {
      // 插入
    for (size_type i = 0; i < ht.buckets.size(); ++i) {
      if (const node* cur = ht.buckets[i]) {
        node* copy = new_node(cur->val);
        buckets[i] = copy;

        for (node* next = cur->next; next; cur = next, next = cur->next) {
          copy->next = new_node(next->val);
          copy = copy->next;
        }
      }
    }
    num_elements = ht.num_elements;
  }
  __STL_UNWIND(clear());
}
```



### 总结

本节分析了`hashtable`的插入, 删除等操作, 在插入过程中如果发现桶满了vector会自动的选择更大的空间, 所以hashtable就没有在自己实现. 

我们在来总结他与`RB-tree` : 

1.  在插入过程, 前者平均是O(1)后者则是O(nlngn)
2.  前者插入是无序的, 后者有序
3.  前者桶满后效率很低, 后者不会考虑到满
4.  两者都实现了可重复和不可重复
5.  前者正向迭代, 没有--, 后者迭代器实现了--

关于`hash_fun`就不再准备分析了, 因为里面就只有几个struct类型的仿函数, 而hashtable没有直接定义`string`等类型的仿函数, 也就无法直接处理这类类型, 需要用户自己来定义, 有兴趣的可以自己看一下. 接下来我们会分析以`hashtable`衍生的配接器, 与RB-tree很类似.