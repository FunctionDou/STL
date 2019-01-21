# hash_set

### 前言

前面我们分析了hashtable, 后面的几节我们来分析其衍生出来的配接器, 本节分析的`hash_set`, 他与`set`最大的不同在与前者的元素是无序排列的, 后者是有序排列



### hash_set操作

```c++
int main()
{
	__gnu_cxx::hash_set<const char *> h_set(1);
	h_set.insert("one");
	h_set.insert("two");
	cout << "h_set size = " << h_set.size() << endl;	// h_set size = 2

	__gnu_cxx::hash_set<const char *> set_h(h_set.begin(), h_set.end());
    // 这里不相等的原因两者的桶并不是一样大, 所以数据存放也不同
	if(h_set == set_h)
		h_set.clear();
	cout << "h_set size = " << h_set.size() << endl;	// h_set size = 2
	cout << "set_h size = " << set_h.size() << endl;	// set_h size = 2

	cout << (*set_h.find("one")) << endl;	// one
	for (const auto &i : set_h)
		cout << i << " ";	// two one
	
	exit(0);
}
```



### hash_set分析

hash_set将哈希表的接口在进行了一次封装, 实现与set类似的功能.

#### 基本类型

```c++
#ifndef __STL_LIMITED_DEFAULT_TEMPLATES
template <class Value, class HashFcn = hash<Value>,
          class EqualKey = equal_to<Value>,
          class Alloc = alloc>
#else
template <class Value, class HashFcn, class EqualKey, class Alloc = alloc>
#endif
class hash_set
{
private:
    // 定义hashtable
  typedef hashtable<Value, Value, HashFcn, identity<Value>,  EqualKey, Alloc> ht;
  ht rep;

public:
  typedef typename ht::key_type key_type;
  typedef typename ht::value_type value_type;
  typedef typename ht::hasher hasher;
  typedef typename ht::key_equal key_equal;

    // 定义为const类型, 键值不允许修改
  typedef typename ht::size_type size_type;
  typedef typename ht::difference_type difference_type;
  typedef typename ht::const_pointer pointer;
  typedef typename ht::const_pointer const_pointer;
  typedef typename ht::const_reference reference;
  typedef typename ht::const_reference const_reference;

    // 定义迭代器
  typedef typename ht::const_iterator iterator;
  typedef typename ht::const_iterator const_iterator;
    // 仿函数
  hasher hash_funct() const { return rep.hash_funct(); }
  key_equal key_eq() const { return rep.key_eq(); }
    ...
};
```



#### 构造函数

```c++
class hash_set
{
    ...
public:
  hash_set() : rep(100, hasher(), key_equal()) {}	// 默认构造函数, 表大小默认为100最近的素数
  explicit hash_set(size_type n) : rep(n, hasher(), key_equal()) {}
  hash_set(size_type n, const hasher& hf) : rep(n, hf, key_equal()) {}
  hash_set(size_type n, const hasher& hf, const key_equal& eql)
    : rep(n, hf, eql) {}

#ifdef __STL_MEMBER_TEMPLATES
  template <class InputIterator>
  hash_set(InputIterator f, InputIterator l)
    : rep(100, hasher(), key_equal()) { rep.insert_unique(f, l); }
  template <class InputIterator>
  hash_set(InputIterator f, InputIterator l, size_type n)
    : rep(n, hasher(), key_equal()) { rep.insert_unique(f, l); }
  template <class InputIterator>
  hash_set(InputIterator f, InputIterator l, size_type n,
           const hasher& hf)
    : rep(n, hf, key_equal()) { rep.insert_unique(f, l); }
  template <class InputIterator>
  hash_set(InputIterator f, InputIterator l, size_type n,
           const hasher& hf, const key_equal& eql)
    : rep(n, hf, eql) { rep.insert_unique(f, l); }
 ...
};
```



#### 功能实现

```c++
class hash_set
{
    ...
public:
    // 都是调用hashtable的接口
  size_type size() const { return rep.size(); }	// 哈希表的元素个数
  size_type max_size() const { return rep.max_size(); }
  bool empty() const { return rep.empty(); }
  void swap(hash_set& hs) { rep.swap(hs.rep); }
  friend bool operator== __STL_NULL_TMPL_ARGS (const hash_set&,
                                               const hash_set&);

  iterator begin() const { return rep.begin(); }
  iterator end() const { return rep.end(); }
	...
};
```

**插入删除等操作**

insert调用的是`insert_unqiue`函数

```c++
class hash_set
{
    ...
public:
    // 都是调用hashtable的接口, 这里insert_unqiue函数
  pair<iterator, bool> insert(const value_type& obj)
    {
      pair<typename ht::iterator, bool> p = rep.insert_unique(obj);
      return pair<iterator, bool>(p.first, p.second);
    }
#ifdef __STL_MEMBER_TEMPLATES
  template <class InputIterator>
  void insert(InputIterator f, InputIterator l) { rep.insert_unique(f,l); }
#else
  void insert(const value_type* f, const value_type* l) {
    rep.insert_unique(f,l);
  }
  void insert(const_iterator f, const_iterator l) {rep.insert_unique(f, l); }
#endif /*__STL_MEMBER_TEMPLATES */
  pair<iterator, bool> insert_noresize(const value_type& obj)
  {
    pair<typename ht::iterator, bool> p = rep.insert_unique_noresize(obj);
    return pair<iterator, bool>(p.first, p.second);
  }

    // 查找
  iterator find(const key_type& key) const { return rep.find(key); }

    // 计数
  size_type count(const key_type& key) const { return rep.count(key); }
  
  pair<iterator, iterator> equal_range(const key_type& key) const
    { return rep.equal_range(key); }

    // 删除
  size_type erase(const key_type& key) {return rep.erase(key); }
  void erase(iterator it) { rep.erase(it); }
  void erase(iterator f, iterator l) { rep.erase(f, l); }
  void clear() { rep.clear(); }

public:
  void resize(size_type hint) { rep.resize(hint); }
  size_type bucket_count() const { return rep.bucket_count(); }
  size_type max_bucket_count() const { return rep.max_bucket_count(); }
  size_type elems_in_bucket(size_type n) const
    { return rep.elems_in_bucket(n); }
};
```

**重载**

```c++
template <class Value, class HashFcn, class EqualKey, class Alloc>
inline bool operator==(const hash_set<Value, HashFcn, EqualKey, Alloc>& hs1,
                       const hash_set<Value, HashFcn, EqualKey, Alloc>& hs2)
{
  return hs1.rep == hs2.rep;
}

template <class Val, class HashFcn, class EqualKey, class Alloc>
inline void swap(hash_set<Val, HashFcn, EqualKey, Alloc>& hs1,
                 hash_set<Val, HashFcn, EqualKey, Alloc>& hs2) {
  hs1.swap(hs2);
}
```



### 总结

本节也只是简单对hash_set做了分析, 因为所有的操作都是通过调用`hashtable`的接口实现的, 而且也与`set`的功能类似, 最大的不同就是两者的有序和无序. 下一节我们分析hash_multiset, 这也是跟multiset类似的功能.

