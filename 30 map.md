# map

### 前言

上一节分析了pair结构, 正是为`map`分析做铺垫, map本身实现也不难, 其数据存储是pair, 存储结构是RB-tree, 即map也并不能说是关联容器, 而应该是配接器. 



### map操作

map的insert必须是以pair为存储结构, 当然也可以直接使用make_pair构造一个临时pair, 这个函数我们上节分析pair的时候讲过.

```c++
int main()
{
	map<string, int> m;
	pair<string, int> p;
	p.first = "zero", p.second = 0;
	m.insert(p);
	m.insert(make_pair("one", 1));

	if(!m.empty())
	{
		cout << m["one"] << " " << m["two"] << endl;	// 1 0
		cout << (*m.find("one")).first << " " << (*m.find("one")).second << endl;	// one 1
	}

	exit(0);
}
```

上面唯一比较复杂的就是`(*m.find("one")).second`的操作, 这在我们下面分析重载`[]`时会具体分析, 下面我们就来分析map吧.



### map分析

**map基本结构定义**

map对象实例化`map<const T1, T2>`, 键值是不能直接修改的, 而数据可以修改.

```c++
#ifndef __STL_LIMITED_DEFAULT_TEMPLATES
template <class Key, class T, class Compare = less<Key>, class Alloc = alloc>
#else
template <class Key, class T, class Compare, class Alloc = alloc>
#endif
class map {
public:
  typedef Key key_type;	// 定义键值
  typedef T data_type;	// 定义数据
  typedef T mapped_type;
  typedef pair<const Key, T> value_type; // 这里定义了map的数据类型为pair, 且键值为const类型, 不能修改
  typedef Compare key_compare;
    
private:
  typedef rb_tree<key_type, value_type, 
                  select1st<value_type>, key_compare, Alloc> rep_type;	// 定义红黑树, map是以rb-tree结构为基础的
  rep_type t;  // red-black tree representing map	
public:
	// 定义类型
  typedef typename rep_type::pointer pointer;
  typedef typename rep_type::const_pointer const_pointer;
  typedef typename rep_type::reference reference;
  typedef typename rep_type::const_reference const_reference;
  typedef typename rep_type::iterator iterator;
  typedef typename rep_type::const_iterator const_iterator;
  typedef typename rep_type::reverse_iterator reverse_iterator;
  typedef typename rep_type::const_reverse_iterator const_reverse_iterator;
  typedef typename rep_type::size_type size_type;
  typedef typename rep_type::difference_type difference_type;
	...
};
```



**嵌套类** : 这是一个仿函数, 为键值key提供比较接口

```c++
class map {
public:
	...
    // 这是一个仿函数, 为键值key提供比较接口
  class value_compare : public binary_function<value_type, value_type, bool> 
  {
  friend class map<Key, T, Compare, Alloc>;
  protected :
    Compare comp;
    value_compare(Compare c) : comp(c) {}
  public:
      // 重载(), 可进行临时比较
    bool operator()(const value_type& x, const value_type& y) const {
      return comp(x.first, y.first);
    }
  };
    ...
};
```

**构造函数** map的所有插入操作都是调用的是`RB-tree`的`insert_unique`, 不允许出现重复的键

```c++
class map {
public:
	...
public:
  // allocation/deallocation
  map() : t(Compare()) {}	// 默认构造函数
  explicit map(const Compare& comp) : t(comp) {}
#ifdef __STL_MEMBER_TEMPLATES
    // 接受两个迭代器
  template <class InputIterator>
  map(InputIterator first, InputIterator last)
    : t(Compare()) { t.insert_unique(first, last); }
  template <class InputIterator>
  map(InputIterator first, InputIterator last, const Compare& comp)
    : t(comp) { t.insert_unique(first, last); }
#else
  map(const value_type* first, const value_type* last)
    : t(Compare()) { t.insert_unique(first, last); }
  map(const value_type* first, const value_type* last, const Compare& comp)
    : t(comp) { t.insert_unique(first, last); }
  map(const_iterator first, const_iterator last)
    : t(Compare()) { t.insert_unique(first, last); }
  map(const_iterator first, const_iterator last, const Compare& comp)
    : t(comp) { t.insert_unique(first, last); }
#endif /* __STL_MEMBER_TEMPLATES */
    ...
};
```



**基本类型属性获取**

```c++
class map {
public:
	...
public:
    // 实际调用的是RB-tree的key_comp函数
  key_compare key_comp() const { return t.key_comp(); }
    // value_comp实际返回的是一个仿函数value_compare
  value_compare value_comp() const { return value_compare(t.key_comp()); }
    // 以下的begin, end等操作都是调用的是RB-tree的接口
  iterator begin() { return t.begin(); }
  const_iterator begin() const { return t.begin(); }
  iterator end() { return t.end(); }
  const_iterator end() const { return t.end(); }
  reverse_iterator rbegin() { return t.rbegin(); }
  const_reverse_iterator rbegin() const { return t.rbegin(); }
  reverse_iterator rend() { return t.rend(); }
  const_reverse_iterator rend() const { return t.rend(); }
  bool empty() const { return t.empty(); }
  size_type size() const { return t.size(); }
  size_type max_size() const { return t.max_size(); }
    // 交换, 调用RB-tree的swap, 实际只交换head和count
  void swap(map<Key, T, Compare, Alloc>& x) { t.swap(x.t); }
    ...
};
template <class Key, class T, class Compare, class Alloc>
inline void swap(map<Key, T, Compare, Alloc>& x, 
                 map<Key, T, Compare, Alloc>& y) {
  x.swap(y);
}
```



**重载**

```c++
class map {
public:
	...
public:
  map(const map<Key, T, Compare, Alloc>& x) : t(x.t) {}
  map<Key, T, Compare, Alloc>& operator=(const map<Key, T, Compare, Alloc>& x)
  {
    t = x.t;
    return *this; 
  }
    ...
};
template <class Key, class T, class Compare, class Alloc>
inline bool operator==(const map<Key, T, Compare, Alloc>& x, 
                       const map<Key, T, Compare, Alloc>& y) {
  return x.t == y.t;
}

template <class Key, class T, class Compare, class Alloc>
inline bool operator<(const map<Key, T, Compare, Alloc>& x, 
                      const map<Key, T, Compare, Alloc>& y) {
  return x.t < y.t;
}
```

重载操作重点分析 []

1.  insert(value_type(k, T()) : 查找是否存在该键值, 如果存在则返回该`pair`, 不存在这重新构造一该键值并且值为空
2.  *((insert(value_type(k, T()))).first) : `pair`的第一个元素表示指向该元素的迭代器, 第二个元素指的是(false与true)是否存在,  `first` 便是取出该迭代器而 ` *` 取出pair.
3.  (*((insert(value_type(k, T()))).first)).second : 取出pair结构中的`second`保存的数据

```c++
class map {
public:
	...
public:
    // 1.  insert(value_type(k, T()) : 查找是否存在该键值, 如果存在则返回该pair, 不存在这重新构造一该键值并且值为空
	// 2.  *((insert(value_type(k, T()))).first) : pair的第一个元素表示指向该元素的迭代器, 第二个元素指的是(false与true)是否存在,  first 便是取出该迭代器而 * 取出pair.
	// 3.  (*((insert(value_type(k, T()))).first)).second : 取出pair结构中的second保存的数据
  T& operator[](const key_type& k) {
    return (*((insert(value_type(k, T()))).first)).second;
  }
    ...
};
```



map的其他insert, erase, find都是直接调用RB-tree的接口函数实现的, 这里就不直接做分析了.



### 总结

实际map也是以RB-tree为底层接口的配接器, 同时map还以pair结构为存储结构, 当这两个都理解后整个map结构分析也就很轻松了.

