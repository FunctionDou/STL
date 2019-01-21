# multiset

### 前言

前面也分析过`set`, 并且`set`不能插入相同的键, 本节分析的`multiset`与set不同之处就是他允许插入相同的键.



### multiset操作

```c++
int main()
{
	multiset<string> multi;
    // 允许重复插入键
	multi.insert("zero");
	multi.insert("zero");
	multi.insert("one");
	multi.insert("one");

    // 证明重复插入进行了
	cout << multi.count("zero") << endl;	// 2

    // 将所有键输出
	if(!multi.empty())
		for(const auto &i : multi)
			cout << i << " ";	// one one zero zero

	exit(0);
}
```



### multiset分析



**类型定义**

```c++
#ifndef __STL_LIMITED_DEFAULT_TEMPLATES
template <class Key, class Compare = less<Key>, class Alloc = alloc>
#else
template <class Key, class Compare, class Alloc = alloc>
#endif
class multiset {
public:
  // typedefs:
  typedef Key key_type;
  typedef Key value_type;
  typedef Compare key_compare;
  typedef Compare value_compare;
  // multiset也是以RB-tree为接口
private:
  typedef rb_tree<key_type, value_type, 
                  identity<value_type>, key_compare, Alloc> rep_type;
  rep_type t;  // red-black tree representing multiset
public:
	// 每个都是const类型, 不允许进行修改
  typedef typename rep_type::const_pointer pointer;
  typedef typename rep_type::const_pointer const_pointer;
  typedef typename rep_type::const_reference reference;
  typedef typename rep_type::const_reference const_reference;
  typedef typename rep_type::const_iterator iterator;
  typedef typename rep_type::const_iterator const_iterator;
  typedef typename rep_type::const_reverse_iterator reverse_iterator;
  typedef typename rep_type::const_reverse_iterator const_reverse_iterator;
  typedef typename rep_type::size_type size_type;
  typedef typename rep_type::difference_type difference_type;
  ...
};
```



**构造函数**

与set不同, multiset是以`insert_equal`为接口, 所以允许插入重复的键值.

```c++
class multiset {
	...
public:
  // allocation/deallocation
  multiset() : t(Compare()) {}
  explicit multiset(const Compare& comp) : t(comp) {}
#ifdef __STL_MEMBER_TEMPLATES
  template <class InputIterator>
  multiset(InputIterator first, InputIterator last)
    : t(Compare()) { t.insert_equal(first, last); }
  template <class InputIterator>
  multiset(InputIterator first, InputIterator last, const Compare& comp)
    : t(comp) { t.insert_equal(first, last); }
#else
  multiset(const value_type* first, const value_type* last)
    : t(Compare()) { t.insert_equal(first, last); }
  multiset(const value_type* first, const value_type* last,
           const Compare& comp)
    : t(comp) { t.insert_equal(first, last); }
  multiset(const_iterator first, const_iterator last)
    : t(Compare()) { t.insert_equal(first, last); }
  multiset(const_iterator first, const_iterator last, const Compare& comp)
    : t(comp) { t.insert_equal(first, last); }
#endif /* __STL_MEMBER_TEMPLATES */
    ...
};
```



**基本属性获取**

```c++
class multiset {
	...
public:
  key_compare key_comp() const { return t.key_comp(); }
    // 返回的是仿函数
  value_compare value_comp() const { return t.key_comp(); }
  iterator begin() const { return t.begin(); }
  iterator end() const { return t.end(); }
  reverse_iterator rbegin() const { return t.rbegin(); } 
  reverse_iterator rend() const { return t.rend(); }
  bool empty() const { return t.empty(); }
  size_type size() const { return t.size(); }
  size_type max_size() const { return t.max_size(); }
    // 交换
  void swap(multiset<Key, Compare, Alloc>& x) { t.swap(x.t); }
    ...
};
template <class Key, class Compare, class Alloc>
inline void swap(multiset<Key, Compare, Alloc>& x, 
                 multiset<Key, Compare, Alloc>& y) {
  x.swap(y);
}
```



**重载** 与set一样

```c++
class multiset {
	...
public:
  multiset(const multiset<Key, Compare, Alloc>& x) : t(x.t) {}
  multiset<Key, Compare, Alloc>&
  operator=(const multiset<Key, Compare, Alloc>& x) {
    t = x.t; 
    return *this;
  }
  friend bool operator== __STL_NULL_TMPL_ARGS (const multiset&, const multiset&);
  friend bool operator< __STL_NULL_TMPL_ARGS (const multiset&, const multiset&);
};
template <class Key, class Compare, class Alloc>
inline bool operator==(const multiset<Key, Compare, Alloc>& x, 
                       const multiset<Key, Compare, Alloc>& y) {
  return x.t == y.t;
}
template <class Key, class Compare, class Alloc>
inline bool operator<(const multiset<Key, Compare, Alloc>& x, 
                      const multiset<Key, Compare, Alloc>& y) {
  return x.t < y.t;
}
```



**insert**

以RB-tree的`insert_equal`为接口. 允许重复插入

```c++
class multiset {
	...
public:
  // insert/erase
  iterator insert(const value_type& x) { 
    return t.insert_equal(x);
  }
  iterator insert(iterator position, const value_type& x) {
    typedef typename rep_type::iterator rep_iterator;
    return t.insert_equal((rep_iterator&)position, x);
  }

#ifdef __STL_MEMBER_TEMPLATES  
  template <class InputIterator>
  void insert(InputIterator first, InputIterator last) {
    t.insert_equal(first, last);
  }
#else
  void insert(const value_type* first, const value_type* last) {
    t.insert_equal(first, last);
  }
  void insert(const_iterator first, const_iterator last) {
    t.insert_equal(first, last);
  }
#endif /* __STL_MEMBER_TEMPLATES */
    ...
};
```



**erase** 删除操作, 同样都是以RB-tree为接口

```c++
class multiset {
	...
public:
  void erase(iterator position) { 
    typedef typename rep_type::iterator rep_iterator;
    t.erase((rep_iterator&)position); 
  }
  size_type erase(const key_type& x) { 
    return t.erase(x); 
  }
  void erase(iterator first, iterator last) { 
    typedef typename rep_type::iterator rep_iterator;
    t.erase((rep_iterator&)first, (rep_iterator&)last); 
  }
  void clear() { t.clear(); }
    ...
};
```

**find等函数**

```c++
class multiset {
	...
public:
  iterator find(const key_type& x) const { return t.find(x); }
  size_type count(const key_type& x) const { return t.count(x); }
  iterator lower_bound(const key_type& x) const {
    return t.lower_bound(x);
  }
  iterator upper_bound(const key_type& x) const {
    return t.upper_bound(x); 
  }
  pair<iterator,iterator> equal_range(const key_type& x) const {
    return t.equal_range(x);
  }
};
```



### 总结

multiset与set最大的不同就是可以重复的插入键值, 一个以`insert_equal`为接口, 一个以`insert_uniqual`为接口, 下一节我们分析map与multimap的区别