# RB-tree

### 前言

上节我们分析了关于RB-tree的迭代器实现, 其中最重要的功能都会在rb-tree结构中调用. 本节我们就来分析`RB-tree`结构.

再来复习一下红黑树的规则:

1.  **每个节点的颜色是黑色或者红色**
2.  **根节点必须是黑色的**
3.  **每个叶节点(NULL)必须是黑色的**
4.  **如果节点是红色的, 则子节点必须是黑色的**
5.  **从节点到每个子孙节点的路径包括的黑色节点数目相同**

```c++
// 红黑定义
typedef bool __rb_tree_color_type;
const __rb_tree_color_type __rb_tree_red = false;
const __rb_tree_color_type __rb_tree_black = true;
```



### RB-tree结构分析

##### rb-tree类型定义

```c++
template <class Key, class Value, class KeyOfValue, class Compare, class Alloc = alloc>
class rb_tree {
protected:
  typedef void* void_pointer;
  typedef __rb_tree_node_base* base_ptr;	// 定义节点指针
  typedef __rb_tree_node<Value> rb_tree_node;	// 定义节点
  typedef simple_alloc<rb_tree_node, Alloc> rb_tree_node_allocator;	// 定义空间配置器
  typedef __rb_tree_color_type color_type;	
public:
	// 满足traits编程
  typedef Key key_type;
  typedef Value value_type;
  typedef value_type* pointer;
  typedef const value_type* const_pointer;
  typedef value_type& reference;
  typedef const value_type& const_reference;
  typedef rb_tree_node* link_type;
  typedef size_t size_type;
  typedef ptrdiff_t difference_type;
protected:
  size_type node_count; // keeps track of size of tree
  link_type header;  	// 头节点, 不是根节点, 头节点是指向根节点
  Compare key_compare;	// 伪函数
public:
	// 定义迭代器
  typedef __rb_tree_iterator<value_type, reference, pointer> iterator;
  typedef __rb_tree_iterator<value_type, const_reference, const_pointer> 
          const_iterator;

#ifdef __STL_CLASS_PARTIAL_SPECIALIZATION
  typedef reverse_iterator<const_iterator> const_reverse_iterator;
  typedef reverse_iterator<iterator> reverse_iterator;
#else /* __STL_CLASS_PARTIAL_SPECIALIZATION */
  typedef reverse_bidirectional_iterator<iterator, value_type, reference,
                                         difference_type>
          reverse_iterator; 
  typedef reverse_bidirectional_iterator<const_iterator, value_type,
                                         const_reference, difference_type>
          const_reverse_iterator;
  ...
};
```



##### 构造析构函数

```c++
template <class Key, class Value, class KeyOfValue, class Compare, class Alloc = alloc>
class rb_tree {
	...
public:                          // allocation/deallocation
  rb_tree(const Compare& comp = Compare())
    : node_count(0), key_compare(comp) { init(); }

  rb_tree(const rb_tree<Key, Value, KeyOfValue, Compare, Alloc>& x) : node_count(0), key_compare(x.key_compare)
  { 
    header = get_node();	// 分配空间
    color(header) = __rb_tree_red;	// 默认节点的颜色设置为红色
    if (x.root() == 0) {	// 如果x为根节点
      root() = 0;
      leftmost() = header;	// 左孩子指向根节点
      rightmost() = header;	// 右孩子指向根节点
    }
    else {
      __STL_TRY {
        root() = __copy(x.root(), header);
      }
      __STL_UNWIND(put_node(header));
      leftmost() = minimum(root());	// 左孩子始终指向最小的节点
      rightmost() = maximum(root());	// 右孩子始终指向最大的节点
    }
    node_count = x.node_count;
  }
  
  // 析构函数
  ~rb_tree() {
    clear();	// 清除或有节点
    put_node(header);	// 释放所有空间
  }
  ...
};
```



```c++
template <class Key, class Value, class KeyOfValue, class Compare, class Alloc = alloc>
class rb_tree {
	...
protected:
	// 分配空间
  link_type get_node() { return rb_tree_node_allocator::allocate(); }	
  	// 释放空间
  void put_node(link_type p) { rb_tree_node_allocator::deallocate(p); }
	// 调用构造函数
  link_type create_node(const value_type& x) {
    link_type tmp = get_node();
    __STL_TRY {
      construct(&tmp->value_field, x);
    }
    // 分配失败释放所有空间
    __STL_UNWIND(put_node(tmp));
    return tmp;
  }
	
	// 分配节点, 并初始化
  link_type clone_node(link_type x) {
    link_type tmp = create_node(x->value_field);
    tmp->color = x->color;
    tmp->left = 0;
    tmp->right = 0;
    return tmp;
  }

	// 释放节点
  void destroy_node(link_type p) {
    destroy(&p->value_field);
    put_node(p);
  }
private:
  void init() {
    header = get_node();
    color(header) = __rb_tree_red; // used to distinguish header from 
                                   // root, in iterator.operator++
    root() = 0;
    leftmost() = header;
    rightmost() = header;
  }
public:
  void clear() {
    if (node_count != 0) {
      __erase(root());
      leftmost() = header;
      root() = 0;
      rightmost() = header;
      node_count = 0;
    }
  } 
  ...
};
```



##### 节点属性

```c++
template <class Key, class Value, class KeyOfValue, class Compare, class Alloc = alloc>
class rb_tree {
	...
protected:
  link_type& root() const { return (link_type&) header->parent; }		// 获取根节点
  link_type& leftmost() const { return (link_type&) header->left; }		// 最小节点
  link_type& rightmost() const { return (link_type&) header->right; }	// 最大节点

	// 当前节点的左节点
	// 当前节点的右节点
  static link_type& left(link_type x) { return (link_type&)(x->left); }
  static link_type& right(link_type x) { return (link_type&)(x->right); }
  static link_type& parent(link_type x) { return (link_type&)(x->parent); }
  static reference value(link_type x) { return x->value_field; }	// 当前节点的数据
  static const Key& key(link_type x) { return KeyOfValue()(value(x)); }
  static color_type& color(link_type x) { return (color_type&)(x->color); }

  static link_type& left(base_ptr x) { return (link_type&)(x->left); }
  static link_type& right(base_ptr x) { return (link_type&)(x->right); }
  static link_type& parent(base_ptr x) { return (link_type&)(x->parent); }
  static reference value(base_ptr x) { return ((link_type)x)->value_field; }
  static const Key& key(base_ptr x) { return KeyOfValue()(value(link_type(x)));} 
  static color_type& color(base_ptr x) { return (color_type&)(link_type(x)->color); }

	// 最小节点
  static link_type minimum(link_type x) { 
    return (link_type)  __rb_tree_node_base::minimum(x);
  }
  	// 最大节点
  static link_type maximum(link_type x) {
    return (link_type) __rb_tree_node_base::maximum(x);
  }
  
public:    
                                // accessors:
  Compare key_comp() const { return key_compare; }
  // begin() 获取的是最小节点
  iterator begin() { return leftmost(); }
  const_iterator begin() const { return leftmost(); }
  // end() 头节点
  iterator end() { return header; }
  const_iterator end() const { return header; }
  reverse_iterator rbegin() { return reverse_iterator(end()); }
  const_reverse_iterator rbegin() const { 
    return const_reverse_iterator(end()); 
  }
  reverse_iterator rend() { return reverse_iterator(begin()); }
  const_reverse_iterator rend() const { 
    return const_reverse_iterator(begin());
  } 
  bool empty() const { return node_count == 0; }		// 树为空
  size_type size() const { return node_count; }			// 节点计数
  size_type max_size() const { return size_type(-1); }	
  ...
};
```



##### swap

rb-tree交换也并不是交换所有的节点, 只是交换了头节点, 节点数和比较伪函数.

```c++
template <class Key, class Value, class KeyOfValue, class Compare, class Alloc = alloc>
class rb_tree {
	...
public:    
  void swap(rb_tree<Key, Value, KeyOfValue, Compare, Alloc>& t) {
    __STD::swap(header, t.header);
    __STD::swap(node_count, t.node_count);
    __STD::swap(key_compare, t.key_compare);
  }
  ...
};
template <class Key, class Value, class KeyOfValue, class Compare, class Alloc>
inline void swap(rb_tree<Key, Value, KeyOfValue, Compare, Alloc>& x, 
                 rb_tree<Key, Value, KeyOfValue, Compare, Alloc>& y) {
  x.swap(y);
}
```



##### 重载

```c++
// 相等比较
template <class Key, class Value, class KeyOfValue, class Compare, class Alloc>
inline bool operator==(const rb_tree<Key, Value, KeyOfValue, Compare, Alloc>& x, 
                       const rb_tree<Key, Value, KeyOfValue, Compare, Alloc>& y) {
  return x.size() == y.size() && equal(x.begin(), x.end(), y.begin());
}

template <class Key, class Value, class KeyOfValue, class Compare, class Alloc>
inline bool operator<(const rb_tree<Key, Value, KeyOfValue, Compare, Alloc>& x, 
                      const rb_tree<Key, Value, KeyOfValue, Compare, Alloc>& y) {
  return lexicographical_compare(x.begin(), x.end(), y.begin(), y.end());
}

// 重载=运算符
template <class Key, class Value, class KeyOfValue, class Compare, class Alloc>
rb_tree<Key, Value, KeyOfValue, Compare, Alloc>& 
rb_tree<Key, Value, KeyOfValue, Compare, Alloc>::
operator=(const rb_tree<Key, Value, KeyOfValue, Compare, Alloc>& x) {
  if (this != &x) {
                                // Note that Key may be a constant type.
    clear();	// 清除所有节点
    node_count = 0;
    key_compare = x.key_compare;        
      // 将每个节点进行赋值
    if (x.root() == 0) {
      root() = 0;
      leftmost() = header;
      rightmost() = header;
    }
    else {
      root() = __copy(x.root(), header);
      leftmost() = minimum(root());
      rightmost() = maximum(root());
      node_count = x.node_count;
    }
  }
  return *this;
}
```



```c++
template <class Key, class Value, class KeyOfValue, class Compare, class Alloc = alloc>
class rb_tree {
	...
private:
  iterator __insert(base_ptr x, base_ptr y, const value_type& v);
  link_type __copy(link_type x, link_type p);
  void __erase(link_type x);
public:
  rb_tree<Key, Value, KeyOfValue, Compare, Alloc>& 
  operator=(const rb_tree<Key, Value, KeyOfValue, Compare, Alloc>& x);
    
public:
                                // insert/erase
  pair<iterator,bool> insert_unique(const value_type& x);
  iterator insert_equal(const value_type& x);

  iterator insert_unique(iterator position, const value_type& x);
  iterator insert_equal(iterator position, const value_type& x);

#ifdef __STL_MEMBER_TEMPLATES  
  template <class InputIterator>
  void insert_unique(InputIterator first, InputIterator last);
  template <class InputIterator>
  void insert_equal(InputIterator first, InputIterator last);
#else /* __STL_MEMBER_TEMPLATES */
  void insert_unique(const_iterator first, const_iterator last);
  void insert_unique(const value_type* first, const value_type* last);
  void insert_equal(const_iterator first, const_iterator last);
  void insert_equal(const value_type* first, const value_type* last);
#endif /* __STL_MEMBER_TEMPLATES */

  void erase(iterator position);
  size_type erase(const key_type& x);
  void erase(iterator first, iterator last);
  void erase(const key_type* first, const key_type* last);
     

public:
                                // set operations:
  iterator find(const key_type& x);
  const_iterator find(const key_type& x) const;
  size_type count(const key_type& x) const;
  iterator lower_bound(const key_type& x);
  const_iterator lower_bound(const key_type& x) const;
  iterator upper_bound(const key_type& x);
  const_iterator upper_bound(const key_type& x) const;
  pair<iterator,iterator> equal_range(const key_type& x);
  pair<const_iterator, const_iterator> equal_range(const key_type& x) const;

public:
                                // Debugging.
  bool __rb_verify() const;
};
```



##### <font color=#b20>插入</font>

```c++
// 插入
template <class Key, class Value, class KeyOfValue, class Compare, class Alloc>
typename rb_tree<Key, Value, KeyOfValue, Compare, Alloc>::iterator
rb_tree<Key, Value, KeyOfValue, Compare, Alloc>::
__insert(base_ptr x_, base_ptr y_, const Value& v) 
{
    // x为要插入的节点
    // y : 插入节点的父节点
    // v : 插入节点的数据
  link_type x = (link_type) x_;
  link_type y = (link_type) y_;
  link_type z;

    // 如果y是头节点, 将插入的节点设置为根节点
  if (y == header || x != 0 || key_compare(KeyOfValue()(v), key(y))) {
    z = create_node(v);	
      // 左节点指向z
    left(y) = z;                // also makes leftmost() = z when y == header
      // y是头节点, 根节点就是z
    if (y == header) {
      root() = z;
      rightmost() = z;
    }
      // 如果y是最小节点, 则把z修改为最小节点
    else if (y == leftmost())
      leftmost() = z;           // maintain leftmost() pointing to min node
  }
  else {
    z = create_node(v);
    right(y) = z;
    if (y == rightmost())
      rightmost() = z;          // maintain rightmost() pointing to max node
  }
  parent(z) = y;
  left(z) = 0;
  right(z) = 0;
  __rb_tree_rebalance(z, header->parent);
  ++node_count;	// 节点数++
  return iterator(z);	// 返回节点z迭代器
}

// 找到值应该插入的地方
template <class Key, class Value, class KeyOfValue, class Compare, class Alloc>
typename rb_tree<Key, Value, KeyOfValue, Compare, Alloc>::iterator
rb_tree<Key, Value, KeyOfValue, Compare, Alloc>::insert_equal(const Value& v)
{
  link_type y = header;
  link_type x = root();
  while (x != 0) {
    y = x;
    x = key_compare(KeyOfValue()(v), key(x)) ? left(x) : right(x);
  }
  return __insert(x, y, v);
}
```

**不能重复的插入**. 这里出现的`pair`结构下节分析, 这里只要知道这个它( ) true表示插入成功, false表示已经存在了.

```c++
template <class Key, class Value, class KeyOfValue, class Compare, class Alloc>
pair<typename rb_tree<Key, Value, KeyOfValue, Compare, Alloc>::iterator, bool>
rb_tree<Key, Value, KeyOfValue, Compare, Alloc>::insert_unique(const Value& v)
{
  link_type y = header;
  link_type x = root();
  bool comp = true;
    // 找到合适的节点
  while (x != 0) {
    y = x;
    comp = key_compare(KeyOfValue()(v), key(x));
    x = comp ? left(x) : right(x);
  }
  iterator j = iterator(y);   
  if (comp)
    if (j == begin())     
      return pair<iterator,bool>(__insert(x, y, v), true);
    else
      --j;
    // 找到插入的节点是否已经存在了
  if (key_compare(key(j.node), KeyOfValue()(v)))
    return pair<iterator,bool>(__insert(x, y, v), true);
  return pair<iterator,bool>(j, false);
}

template <class Key, class Val, class KeyOfValue, class Compare, class Alloc>
typename rb_tree<Key, Val, KeyOfValue, Compare, Alloc>::iterator 
rb_tree<Key, Val, KeyOfValue, Compare, Alloc>::insert_unique(iterator position,const Val& v) 
{
  if (position.node == header->left) // begin()
    if (size() > 0 && key_compare(KeyOfValue()(v), key(position.node)))
      return __insert(position.node, position.node, v);
  // first argument just needs to be non-null 
    else
      return insert_unique(v).first;
  else if (position.node == header) // end()
    if (key_compare(key(rightmost()), KeyOfValue()(v)))
      return __insert(0, rightmost(), v);
    else
      return insert_unique(v).first;
  else {
    iterator before = position;
    --before;
    if (key_compare(key(before.node), KeyOfValue()(v))
        && key_compare(KeyOfValue()(v), key(position.node)))
      if (right(before.node) == 0)
        return __insert(0, before.node, v); 
      else
        return __insert(position.node, position.node, v);
    // first argument just needs to be non-null 
    else
      return insert_unique(v).first;
  }
}
```



```c++
template <class Key, class Val, class KeyOfValue, class Compare, class Alloc>
typename rb_tree<Key, Val, KeyOfValue, Compare, Alloc>::iterator 
rb_tree<Key, Val, KeyOfValue, Compare, Alloc>::insert_equal(iterator position,
                                                            const Val& v) {
  if (position.node == header->left) // begin()
    if (size() > 0 && key_compare(KeyOfValue()(v), key(position.node)))
      return __insert(position.node, position.node, v);
  // first argument just needs to be non-null 
    else
      return insert_equal(v);
  else if (position.node == header) // end()
    if (!key_compare(KeyOfValue()(v), key(rightmost())))
      return __insert(0, rightmost(), v);
    else
      return insert_equal(v);
  else {
    iterator before = position;
    --before;
    if (!key_compare(KeyOfValue()(v), key(before.node))
        && !key_compare(key(position.node), KeyOfValue()(v)))
      if (right(before.node) == 0)
        return __insert(0, before.node, v); 
      else
        return __insert(position.node, position.node, v);
    // first argument just needs to be non-null 
    else
      return insert_equal(v);
  }
}
```

这里只选择了两个, 其中还有几个重载函数没有列出

```c++
// 范围的插入(可以重复)
template <class K, class V, class KoV, class Cmp, class Al> template<class II>
void rb_tree<K, V, KoV, Cmp, Al>::insert_equal(II first, II last) {
  for ( ; first != last; ++first)
    insert_equal(*first);
}

// 范围插入(不可重复)
template <class K, class V, class KoV, class Cmp, class Al> template<class II>
void rb_tree<K, V, KoV, Cmp, Al>::insert_unique(II first, II last) {
  for ( ; first != last; ++first)
    insert_unique(*first);
}
```



##### <font color=#b20>删除</font>

```c++
template <class Key, class Value, class KeyOfValue, class Compare, class Alloc>
inline void
rb_tree<Key, Value, KeyOfValue, Compare, Alloc>::erase(iterator position) {
  link_type y = (link_type) __rb_tree_rebalance_for_erase(position.node,
                                                          header->parent,
                                                          header->left,
                                                          header->right);
  destroy_node(y);	// 释放空间
  --node_count;		// 节点数--
}

template <class Key, class Value, class KeyOfValue, class Compare, class Alloc>
void rb_tree<Key, Value, KeyOfValue, Compare, Alloc>::__erase(link_type x) {
                                // erase without rebalancing
   // 递归删除x指向的所有节点
  while (x != 0) {
    __erase(right(x));
    link_type y = left(x);
    destroy_node(x);
    x = y;
  }
}

// 范围删除节点
// 调用erase函数
template <class Key, class Value, class KeyOfValue, class Compare, class Alloc>
void rb_tree<Key, Value, KeyOfValue, Compare, Alloc>::erase(iterator first, 
                                                            iterator last) {
  if (first == begin() && last == end())
    clear();
  else
    while (first != last) erase(first++);
}
template <class Key, class Value, class KeyOfValue, class Compare, class Alloc>
void rb_tree<Key, Value, KeyOfValue, Compare, Alloc>::erase(const Key* first, 
                                                            const Key* last) {
  while (first != last) erase(*first++);
}

template <class Key, class Value, class KeyOfValue, class Compare, class Alloc>
typename rb_tree<Key, Value, KeyOfValue, Compare, Alloc>::size_type 
rb_tree<Key, Value, KeyOfValue, Compare, Alloc>::erase(const Key& x) {
  pair<iterator,iterator> p = equal_range(x);
  size_type n = 0;
	// 计算长度, erase进行删除
  distance(p.first, p.second, n);
  erase(p.first, p.second);
  return n;	// 返回删除的个数
}
```



##### 复制

```c++
template <class K, class V, class KeyOfValue, class Compare, class Alloc>
typename rb_tree<K, V, KeyOfValue, Compare, Alloc>::link_type 
rb_tree<K, V, KeyOfValue, Compare, Alloc>::__copy(link_type x, link_type p) {
                                // structural copy.  x and p must be non-null.
  // 复制x的节点为top
  // top的父节点指向p
  link_type top = clone_node(x);
  top->parent = p;
 
  __STL_TRY {
  	// x的右节点存在
    if (x->right)
    	// 递归复制右节点
      top->right = __copy(right(x), top);
    p = top;
    // x指向左节点
    x = left(x);

	// 只要x节点存在
    while (x != 0) {
    	// y为复制当前x的节点
    	// p的左孩子指向y, y的父节点指向p
      link_type y = clone_node(x);
      p->left = y;
      y->parent = p;
      // x的右节点存在
      if (x->right)
      	// 递归复制右节点
        y->right = __copy(right(x), y);
      p = y;
      x = left(x);
    }
  }
  // 一个分配失败就销毁所有的空间
  __STL_UNWIND(__erase(top));

  return top;
}
```



##### <font color=#b20>find</font>

```c++
template <class Key, class Value, class KeyOfValue, class Compare, class Alloc>
typename rb_tree<Key, Value, KeyOfValue, Compare, Alloc>::iterator 
rb_tree<Key, Value, KeyOfValue, Compare, Alloc>::find(const Key& k) {
  link_type y = header;        // Last node which is not less than k. 
  link_type x = root();        // Current node. 

	// 像二叉树一样通过节点比较
  while (x != 0) 
    if (!key_compare(key(x), k))
      y = x, x = left(x);
    else
      x = right(x);

  iterator j = iterator(y);   
  return (j == end() || key_compare(k, key(j.node))) ? end() : j;
}

template <class Key, class Value, class KeyOfValue, class Compare, class Alloc>
typename rb_tree<Key, Value, KeyOfValue, Compare, Alloc>::const_iterator 
rb_tree<Key, Value, KeyOfValue, Compare, Alloc>::find(const Key& k) const {
  link_type y = header; /* Last node which is not less than k. */
  link_type x = root(); /* Current node. */

  while (x != 0) {
    if (!key_compare(key(x), k))
      y = x, x = left(x);
    else
      x = right(x);
  }
  const_iterator j = const_iterator(y);   
  return (j == end() || key_compare(k, key(j.node))) ? end() : j;
}
```



##### count

```c++
// 计算RB-tree中k出现的次数
template <class Key, class Value, class KeyOfValue, class Compare, class Alloc>
typename rb_tree<Key, Value, KeyOfValue, Compare, Alloc>::size_type 
rb_tree<Key, Value, KeyOfValue, Compare, Alloc>::count(const Key& k) const {
  pair<const_iterator, const_iterator> p = equal_range(k);
  size_type n = 0;
  distance(p.first, p.second, n);
  return n;
}
```



```c++
// 计算红黑树有多少个黑节点
inline int __black_count(__rb_tree_node_base* node, __rb_tree_node_base* root)
{
  if (node == 0)
    return 0;
  else {
    int bc = node->color == __rb_tree_black ? 1 : 0;
    if (node == root)
      return bc;
    else
      return bc + __black_count(node->parent, root);
  }
}
```



```c++
// 检查是否符合rb-tree
template <class Key, class Value, class KeyOfValue, class Compare, class Alloc>
bool 
rb_tree<Key, Value, KeyOfValue, Compare, Alloc>::__rb_verify() const
{
    // 空树
  if (node_count == 0 || begin() == end())
    return node_count == 0 && begin() == end() &&
      header->left == header && header->right == header;
  
    // 最左节点到根节点的黑色节点数量为 len
  int len = __black_count(leftmost(), root());
    // 遍历每个节点
  for (const_iterator it = begin(); it != end(); ++it) {
    link_type x = (link_type) it.node;
    link_type L = left(x);
    link_type R = right(x);
	
      // 节点是红色如果子节点也是红色就不满足
    if (x->color == __rb_tree_red)
      if ((L && L->color == __rb_tree_red) ||
          (R && R->color == __rb_tree_red))
        return false;

    if (L && key_compare(key(x), key(L)))
      return false;
    if (R && key_compare(key(R), key(x)))
      return false;

    if (!L && !R && __black_count(x, root()) != len)
      return false;
  }

  if (leftmost() != __rb_tree_node_base::minimum(root()))
    return false;
  if (rightmost() != __rb_tree_node_base::maximum(root()))
    return false;

  return true;
}
```



### 总结

本节很粗略的分析了`RB-tree`的结构和功能实现, 最主要要掌握插入, 删除操作, 因为执行插入和删除都可能会造成不满足rb-tree, 就要进行调整, 左旋, 右旋以及颜色调整我们在上一节就已经分析过了. 在分析插入时有可重复和不可重复的插入, 这两种函数在后面几节分析具体用在哪里.