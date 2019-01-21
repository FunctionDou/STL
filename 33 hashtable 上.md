# hashtable

### 前言

前面我们分析过RB-tree关联容器, `RB-tree`在插入(可重复和不可重复), 删除等操作时间复杂度都是O(nlngn), 以及满足5个规则, 以及以他为底层的配接器; 本节就来分析`hashtable`另个关联容器, 他在插入, 删除等操作都可以做到O(1)的时间复杂度.



### 哈希表概念

#### 哈希方法

1.  **直接定址法**：取关键字或关键字的某个线性函数值为散列地址。（这种散列函数叫做自身函数）
2.  **数字分析法**：假设关键字是以r为基的数，并且哈希表中可能出现的关键字都是事先知道的，则可取关键字的若干数位组成哈希地址。
3.  **平方取中法**：取关键字平方后的中间几位为哈希地址。通常在选定哈希函数时不一定能知道关键字的全部情况，取其中的哪几位也不一定合适，而一个数平方后的中间几位数和数的每一位都相关，由此使随机分布的关键字得到的哈希地址也是随机的。取的位数由表长决定。
4.  **折叠法**：将关键字分割成位数相同的几部分（最后一部分的位数可以不同），然后取这几部分的叠加和（舍去进位）作为哈希地址。
5.  **随机数法**
  6.**除留余数法**：取关键字被某个不大于散列表表长m的数p除后所得的余数为散列地址。不仅可以对关键字直接取模，也可在折叠法、平方取中法等运算之后取模。对p的选择很重要，一般取素数或m，若p选择不好，容易产生冲突。

*hashtable解决冲突的办法就是开链.*



#### 冲突处理

哈希表的冲突处理也有很多种.

1.  开放定址法
    - 线性探测 : 本来的位置被占有(冲突), 重新再往后找到第一个有空的位置插入进去
    - 二次探测 : 本来的位置被占有(冲突), 每次有冲突就平方一次重新查找
2.  开链 : 本来的位置被占有(冲突), 形成一个链表插入到链表中

**装载因子 :  装入表中的元素 / 表的实际大小.** 装载因子越大说明冲突的可能性就越大.



### hashtable分析

#### 桶与节点. 

桶 : 定义的哈希表大小, 以vector为桶

节点 : 链表

```c++
// 这里链表是自定义的, 并没有采用list和slist
template <class Value>
struct __hashtable_node
{
  __hashtable_node* next;
  Value val;
}; 
```



```c++
// 前置声明
template <class Value, class Key, class HashFcn,
          class ExtractKey, class EqualKey, class Alloc = alloc>
class hashtable;

template <class Value, class Key, class HashFcn,
          class ExtractKey, class EqualKey, class Alloc>
struct __hashtable_iterator;

template <class Value, class Key, class HashFcn,
          class ExtractKey, class EqualKey, class Alloc>
struct __hashtable_const_iterator;
```



#### hashtable迭代器

hashtable迭代器是`forward_iterator_tag`类型, 正向迭代器, 所以他也就没有重载`--` , 没有回退.

`__hashtable_const_iterator`与`__hashtable_iterator`一样, 这里就只分析后者

```c++
template <class Value, class Key, class HashFcn, class ExtractKey, class EqualKey, class Alloc>
struct __hashtable_iterator {
  typedef hashtable<Value, Key, HashFcn, ExtractKey, EqualKey, Alloc>  hashtable;
  typedef __hashtable_iterator<Value, Key, HashFcn, ExtractKey, EqualKey, Alloc>
          iterator;
  typedef __hashtable_const_iterator<Value, Key, HashFcn,  ExtractKey, EqualKey, Alloc>
          const_iterator;
  typedef __hashtable_node<Value> node;

  typedef forward_iterator_tag iterator_category;	// 正向迭代器
  typedef Value value_type;
  typedef ptrdiff_t difference_type;
  typedef size_t size_type;
  typedef Value& reference;
  typedef Value* pointer;

  node* cur;		// 定义节点
  hashtable* ht;	// 定义哈希表指针

  __hashtable_iterator(node* n, hashtable* tab) : cur(n), ht(tab) {}
  __hashtable_iterator() {}
  // 重载指针
  reference operator*() const { return cur->val; }
#ifndef __SGI_STL_NO_ARROW_OPERATOR
  pointer operator->() const { return &(operator*()); }
#endif /* __SGI_STL_NO_ARROW_OPERATOR */
	// 重在++, 因为是正向迭代器, 所以没有--
  iterator& operator++();
  iterator operator++(int);
  bool operator==(const iterator& it) const { return cur == it.cur; }
  bool operator!=(const iterator& it) const { return cur != it.cur; }
};
```



#### 定义哈希表大小

定义了哈希表的大小, 默认long为32位, 定义了28个数组大小. 哈希表的的大小都是素数, 减少冲突

```c++
// Note: assumes long is at least 32 bits.
static const int __stl_num_primes = 28;
static const unsigned long __stl_prime_list[__stl_num_primes] =
{
  53,         97,         193,       389,       769,
  1543,       3079,       6151,      12289,     24593,
  49157,      98317,      196613,    393241,    786433,
  1572869,    3145739,    6291469,   12582917,  25165843,
  50331653,   100663319,  201326611, 402653189, 805306457, 
  1610612741, 3221225473, 4294967291
};
// 找到大于n最近的素数
inline unsigned long __stl_next_prime(unsigned long n)
{
  const unsigned long* first = __stl_prime_list;
  const unsigned long* last = __stl_prime_list + __stl_num_primes;
  const unsigned long* pos = lower_bound(first, last, n);
  return pos == last ? *(last - 1) : *pos;
}
```



#### 哈希表

**hashtable类型定义**

模板参数含义 :

1.  Value : 节点的实值类型 
2.  Key : 节点的键值类型 
3.  HashFcn : hash function的类型 
4.  ExtractKey : 从节点中取出键值的方法(函数或仿函数)
5.  EqualKey : 判断键值是否相同的方法(函数或仿函数)

```c++
template <class Value, class Key, class HashFcn, class ExtractKey, class EqualKey, class Alloc>
class hashtable {
public:
  typedef Key key_type;
  typedef Value value_type;
  typedef HashFcn hasher;
  typedef EqualKey key_equal;

  typedef size_t            size_type;
  typedef ptrdiff_t         difference_type;
  typedef value_type*       pointer;
  typedef const value_type* const_pointer;
  typedef value_type&       reference;
  typedef const value_type& const_reference;

	// 这里返回的都是仿函数
  hasher hash_funct() const { return hash; }
  key_equal key_eq() const { return equals; }

private:
	// 这里定义的都是函数或者仿函数
  hasher hash;
  key_equal equals;
  ExtractKey get_key;

  typedef __hashtable_node<Value> node;
  typedef simple_alloc<node, Alloc> node_allocator;

  vector<node*,Alloc> buckets;	// 以vector作为桶, node*
  size_type num_elements;		// 哈希表中元素个数的计数

public:
  typedef __hashtable_iterator<Value, Key, HashFcn, ExtractKey, EqualKey, Alloc> iterator;

  typedef __hashtable_const_iterator<Value, Key, HashFcn, ExtractKey, EqualKey, Alloc> const_iterator;

// 迭代器定义为友元
  friend struct
  __hashtable_iterator<Value, Key, HashFcn, ExtractKey, EqualKey, Alloc>;
  friend struct
  __hashtable_const_iterator<Value, Key, HashFcn, ExtractKey, EqualKey, Alloc>;
  ...
};
```



**构造与析构函数**

```c++
template <class Value, class Key, class HashFcn, class ExtractKey, class EqualKey, class Alloc>
class hashtable {
	...
public:
	// 构造函数, 没有定义默认构造函数
  hashtable(size_type n, const HashFcn&  hf,const EqualKey&   eql,const ExtractKey& ext)
    : hash(hf), equals(eql), get_key(ext), num_elements(0)
  {
    initialize_buckets(n);
  }

  hashtable(size_type n, const HashFcn&  hf, const EqualKey&   eql)
    : hash(hf), equals(eql), get_key(ExtractKey()), num_elements(0)
  {
    initialize_buckets(n);
  }
	// 拷贝构造函数
  hashtable(const hashtable& ht)
    : hash(ht.hash), equals(ht.equals), get_key(ht.get_key), num_elements(0)
  {
    copy_from(ht);
  }
  // 析构函数
  ~hashtable() { clear(); }
  ...
};
```



**基本属性获取**

```c++
template <class Value, class Key, class HashFcn, class ExtractKey, class EqualKey, class Alloc>
class hashtable {
	...
public:
  size_type size() const { return num_elements; }
  size_type max_size() const { return size_type(-1); }
  bool empty() const { return size() == 0; }

	// 交换, 并不是交换所有数据, 只是交换了其指针指向和个数
  void swap(hashtable& ht)
  {
    __STD::swap(hash, ht.hash);
    __STD::swap(equals, ht.equals);
    __STD::swap(get_key, ht.get_key);
    buckets.swap(ht.buckets);
    __STD::swap(num_elements, ht.num_elements);
  }

  iterator begin()
  { 
    for (size_type n = 0; n < buckets.size(); ++n)
    	// 从头遍历桶, 如果有不空的链表存在, 就返回该链表的第一个元素
      if (buckets[n])
        return iterator(buckets[n], this);
    // 没有元素就返回end.
    return end();
  }
	// end返回0
  iterator end() { return iterator(0, this); }

  const_iterator begin() const
  {
    for (size_type n = 0; n < buckets.size(); ++n)
      if (buckets[n])
        return const_iterator(buckets[n], this);
    return end();
  }
  const_iterator end() const { return const_iterator(0, this); }
  
  // 返回桶的大小
  size_type bucket_count() const { return buckets.size(); }

  size_type max_bucket_count() const
    { return __stl_prime_list[__stl_num_primes - 1]; } 

 // 返回指定位置的节点的个数
  size_type elems_in_bucket(size_type bucket) const
  {
    size_type result = 0;
    for (node* cur = buckets[bucket]; cur; cur = cur->next)
      result += 1;
    return result;
  }
  ...
};
```



### 总结

本节只是分析了哈希表的基本构成是桶(vector), 链表(解决冲突). `hashtable`是`forward_iterator_tag`类型的正向迭代器,没有`--`操作, 下节我们继续分析剩下的代码.

