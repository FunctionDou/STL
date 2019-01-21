# RB-tree

### 前言

本节将分析STL中难度很高的`RB-tree`, 如果对红黑树有所认识的那么分析起来的难度也就不是很大, 对红黑树没有太多了解的直接来分析的难度就非常的大了, 可以对红黑树有个了解[红黑树之原理和算法详细介绍](https://www.cnblogs.com/skywang12345/p/3245399.html). 红黑树是很类似与`AVL-tree`的, 但是因为`AVL-tree`在插入,删除的操作做的实际操作很多, 综合而言计算机对内存的管理都采用平衡性相对`AVL-tree`较差一点的红黑树. 

`RB-tree`规则:

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

本节就只分析RB-tree的基本结构, 但rb-tree的很多功能都是调用基本结构实现的功能, 像**左旋, 右旋, 删除, 调整红黑树**这些都非常的重要.  



### RB-tree基本结构分析

基本结构与list相似, 将节点与数据分开定义, `__rb_tree_node_base`定义指针; `__rb_tree_node`继承前者, 增加了数据, 就是一个完整的节点.



#### RB-tree基本结构

**__rb_tree_node_base**

```c++
struct __rb_tree_node_base
{
  typedef __rb_tree_color_type color_type;	
  typedef __rb_tree_node_base* base_ptr;	

  color_type color; 	// 定义节点颜色
  base_ptr parent;		// 定义父节点
  base_ptr left;		// 定义左孩子
  base_ptr right;		// 定义右孩子

    // 查找最小节点
  static base_ptr minimum(base_ptr x)
  {
    while (x->left != 0) x = x->left;
    return x;
  }

    // 查找最大节点
  static base_ptr maximum(base_ptr x)
  {
    while (x->right != 0) x = x->right;
    return x;
  }
};
```

**__rb_tree_node** : 完整的节点

```c++
template <class Value>
struct __rb_tree_node : public __rb_tree_node_base	// 继承__rb_tree_node_base
{
  typedef __rb_tree_node<Value>* link_type;
  Value value_field;	// 定义节点数据
};
```



#### RB-tree迭代器

**__rb_tree_base_iterator** 迭代器基本结构

迭代器中`increment`和`decrement`函数是实现++与--的功能的核心.

```c++
struct __rb_tree_base_iterator
{
  typedef __rb_tree_node_base::base_ptr base_ptr;
  typedef bidirectional_iterator_tag iterator_category;	// bidirectional_iterator_tag类型的迭代器
  typedef ptrdiff_t difference_type;
  base_ptr node;	// 指针节点

    // ++核心函数
    // 节点是从node节点出发, 一直寻找右节点的左孩子, 每次找到比上次大的元素
  void increment()
  {
      // 有右节点, 就往右节点走
    if (node->right != 0) {
      node = node->right;
        // 一直往左节点走, 直到走到头
      while (node->left != 0)
        node = node->left;
    }
      // 没有右节点, 就寻找父节点
    else {
      base_ptr y = node->parent;
        // 如果该节点是父节点的右孩子就继续往上找, 直到y节点不是父节点的右孩子
      while (node == y->right) {
        node = y;
        y = y->parent;
      }
      if (node->right != y)
        node = y;
    }
  }
	
    // --核心代码
    // 节点是从node节点出发, 一直寻找左节点的右孩子, 每次找到比上次小的元素
  void decrement()
  {
      // 只有根节点, 每次--都是根节点
    if (node->color == __rb_tree_red && node->parent->parent == node)
      node = node->right;
      // 有左节点
    else if (node->left != 0) {
        // 往左节点走
      base_ptr y = node->left;
        // 只要有右节点就一直往右节点走
      while (y->right != 0)
        y = y->right;
      node = y;
    }
      // 没有左节点
    else {
        // 寻找父节点
      base_ptr y = node->parent;
        // 如果当前节点是父节点的左孩子就继续寻找父节点直到不再是左孩子
      while (node == y->left) {
        node = y;
        y = y->parent;
      }
      node = y;
    }
  }
};
```

**__rb_tree_iterator** 迭代器

```c++
template <class Value, class Ref, class Ptr>
struct __rb_tree_iterator : public __rb_tree_base_iterator	// 继承__rb_tree_base_iterator
{
  typedef Value value_type;
  typedef Ref reference;
  typedef Ptr pointer;
  typedef __rb_tree_iterator<Value, Value&, Value*>             iterator;
  typedef __rb_tree_iterator<Value, const Value&, const Value*> const_iterator;
  typedef __rb_tree_iterator<Value, Ref, Ptr>                   self;
  typedef __rb_tree_node<Value>* link_type;

	// 构造函数
  __rb_tree_iterator() {}
  __rb_tree_iterator(link_type x) { node = x; }	// 初始化node节点
  __rb_tree_iterator(const iterator& it) { node = it.node; }	// 初始化node节点

	// 重载指针
  reference operator*() const { return link_type(node)->value_field; }
#ifndef __SGI_STL_NO_ARROW_OPERATOR
  pointer operator->() const { return &(operator*()); }
#endif /* __SGI_STL_NO_ARROW_OPERATOR */

	// 重载++与--操作, 调用increment和decrement函数
  self& operator++() { increment(); return *this; }
  self operator++(int) {
    self tmp = *this;
    increment();
    return tmp;
  }
    
  self& operator--() { decrement(); return *this; }
  self operator--(int) {
    self tmp = *this;
    decrement();
    return tmp;
  }
};
```

traits编程

```c++
#ifndef __STL_CLASS_PARTIAL_SPECIALIZATION
inline bidirectional_iterator_tag
iterator_category(const __rb_tree_base_iterator&) {
  return bidirectional_iterator_tag();
}

inline __rb_tree_base_iterator::difference_type*
distance_type(const __rb_tree_base_iterator&) {
  return (__rb_tree_base_iterator::difference_type*) 0;
}

template <class Value, class Ref, class Ptr>
inline Value* value_type(const __rb_tree_iterator<Value, Ref, Ptr>&) {
  return (Value*) 0;
}
#endif /* __STL_CLASS_PARTIAL_SPECIALIZATION */
```



重载

```c++
// ==与!= 比较两个tree的node是相同
inline bool operator==(const __rb_tree_base_iterator& x,
                       const __rb_tree_base_iterator& y) {
  return x.node == y.node;
}
inline bool operator!=(const __rb_tree_base_iterator& x,
                       const __rb_tree_base_iterator& y) {
  return x.node != y.node;
}
```



**红黑树的调整最重要的就是用左旋和右旋.**

##### <font color=#b20>左旋</font>

```c++
inline void 
__rb_tree_rotate_left(__rb_tree_node_base* x, __rb_tree_node_base*& root)
{
  __rb_tree_node_base* y = x->right;	// y为x的右孩子
  x->right = y->left;	// x的右孩子为y的左孩子
    // 如果y的左节点不为空
    // 将y的左节点的父节点指向x
  if (y->left != 0)
    y->left->parent = x;	
  // y的父节点指向x的父节点
  y->parent = x->parent;

    // 如果x就是为根节点
  if (x == root)
    root = y;
    // 如果x为父节点的左孩子
    // x的父节点的左节点指向y
  else if (x == x->parent->left)
    x->parent->left = y;
    // 如果x为父节点的右孩子
    // x的父节点的右节点指向y
  else
    x->parent->right = y;
  // y的左孩子指向x
  // x的父节点指向y
  y->left = x;
  x->parent = y;
}
```

##### <font color=#b20>右旋</font>

```c++
inline void 
__rb_tree_rotate_right(__rb_tree_node_base* x, __rb_tree_node_base*& root)
{
  __rb_tree_node_base* y = x->left;	// y为x的左节点
  x->left = y->right;	// x的左节点指向y的右节点
    // y有右孩子
    // y的的右孩子的父节点指向节点x
  if (y->right != 0)
    y->right->parent = x;
  // y的父节点指向x的父节点
  y->parent = x->parent;

    // 如果x为根节点
    // 将y改为根节点
  if (x == root)
    root = y;
    // x为父节点的右孩子
    // x的父节点的左节点指向y
  else if (x == x->parent->right)
    x->parent->right = y;
    // x的父节点的左节点指向y
  else
    x->parent->left = y;
  // y的右节点指向x
  // x的父节点指向y
  y->right = x;
  x->parent = y;
}
```



##### <font color=#b20>调整红黑树 : </font>

1.  (如果) x不是根节点同时x的父节点颜色为红色
    1.  (如果) x的父节点是x的祖节点的左孩子
        1.  有y为x父节点的兄弟
        2.  (如果) y节点存在并且颜色为红色
            1.  将父节点和兄弟节点颜色都改为黑色
            2.  祖节点改为红色
            3.  当前节点(x)为祖节点
        3.  (否则) y节点不存在或颜色为黑色
            1.  x是父节点的右孩子
            2.  当前节点为x的父节点
            3.  左旋
            4.  x父节点颜色改为黑色
            5.  兄弟节点颜色改为红色
            6.  以祖节点为节点右旋
    2.  (否则) x的父节点是x的祖节点的右孩子
        1.  有y为x父节点的兄弟
        2.  (如果) y存在并且颜色为红色
            1.  将父节点和兄弟节点颜色都改为黑色
            2.  祖节点改为红色
            3.  当前节点(x)为祖节点
        3.  (否则) y节点不存在或颜色为黑色
            1.  x是父节点的左孩子
            2.  右旋
            3.  x父节点颜色改为黑色
            4.  兄弟节点颜色改为红色
            5.  以祖节点为节点左旋
2.  (否则) 将根节点调整为黑色, 因为根节点可能会被修改



```c++
// 这里x指的当前节点, 因为后面会修改到x, 描述x就会出问题
inline void 
__rb_tree_rebalance(__rb_tree_node_base* x, __rb_tree_node_base*& root)
{
  x->color = __rb_tree_red;	// 当前节点颜色改为红色
    // x不是根节点同时x的父节点颜色为红色
  while (x != root && x->parent->color == __rb_tree_red) {
      /************ 1 **********/
      // x的父节点是x的祖节点的左孩子
    if (x->parent == x->parent->parent->left) {
        // 有y为x父节点的兄弟
      __rb_tree_node_base* y = x->parent->parent->right;
        /********* a ***********/
        // y节点存在并且颜色为红色
      if (y && y->color == __rb_tree_red) {
        // 将父节点和兄弟节点颜色都改为黑色
        // 祖节点改为红色
        // 当前节点(x)为祖节点
        x->parent->color = __rb_tree_black;
        y->color = __rb_tree_black;
        x->parent->parent->color = __rb_tree_red;
        x = x->parent->parent;
      }
        /********* b ***********/
        // y节点不存在或颜色为黑色
      else {
          // x是父节点的右孩子
        if (x == x->parent->right) {
            // 当前节点为x的父节点
            // 左旋
          x = x->parent;
          __rb_tree_rotate_left(x, root);
        }
          // x父节点颜色改为黑色
          // 兄弟节点颜色改为红色
          // 以祖节点为节点右旋
        x->parent->color = __rb_tree_black;
        x->parent->parent->color = __rb_tree_red;
        __rb_tree_rotate_right(x->parent->parent, root);
      }
    }
     /************ 2 **********/
    // x的父节点是x的祖节点的右孩子
    else {
        // 有y为x父节点的兄弟
      __rb_tree_node_base* y = x->parent->parent->left;
        /********* a ***********/
        // y存在并且颜色为红色
      if (y && y->color == __rb_tree_red) {
          // 将父节点和兄弟节点颜色都改为黑色
          // 祖节点改为红色
          // 当前节点(x)为祖节点
        x->parent->color = __rb_tree_black;
        y->color = __rb_tree_black;
        x->parent->parent->color = __rb_tree_red;
        x = x->parent->parent;
      }
        /********* b ***********/
        // y节点不存在或颜色为黑色
      else {
          // x是父节点的左孩子
        if (x == x->parent->left) {
            // 当前节点为x的父节点
            // 右旋
          x = x->parent;
          __rb_tree_rotate_right(x, root);
        }
          // x父节点颜色改为黑色
          // 兄弟节点颜色改为红色
          // 以祖节点为节点左旋
        x->parent->color = __rb_tree_black;
        x->parent->parent->color = __rb_tree_red;
        __rb_tree_rotate_left(x->parent->parent, root);
      }
    }
  }
    // 将根节点调整为黑色, 因为根节点可能会被修改
  root->color = __rb_tree_black;
}
```



##### <font color=#b20>删除节点</font>

```c++
// 这里x指的当前节点, 因为后面会修改到x, 描述x就会出问题
inline __rb_tree_node_base*
__rb_tree_rebalance_for_erase(__rb_tree_node_base* z,
                              __rb_tree_node_base*& root,
                              __rb_tree_node_base*& leftmost,
                              __rb_tree_node_base*& rightmost)
{
    // y保存z节点指针
  __rb_tree_node_base* y = z;
  __rb_tree_node_base* x = 0;
  __rb_tree_node_base* x_parent = 0;
    /******* 1 *********/
    // y不存在左节点
  if (y->left == 0)             // z has at most one non-null child. y == z.
      // 则当前节点修改为x的右孩子
    x = y->right;               // x might be null.
    //  y存在左节点
    /******* 2 *********/
  else
      /******* a *********/  
    // y不存在右节点
    if (y->right == 0)          // z has exactly one non-null child.  y == z.
        // 则当前节点修改为x的左孩子
      x = y->left;              // x is not null.
      /******* b *********/ 
    // y存在左右节点
    else {                      // z has two non-null children.  Set y to
       // y修改为y的右节点
       // 如果此时y的左节点存在, 就一直往左边走
       // 当前节点为y的右节点
      y = y->right;             //   z's successor.  x might be null.
      while (y->left != 0)
        y = y->left;
      x = y->right;
    }
  /******* 1 *********/ 
  // 以上的操作就是为了找到z的边的最先的那个节点为y
  // y节点被修改过
  if (y != z) {                 // relink y in place of z.  y is z's successor
      // z的左节点的父节点指向y
      // y的左节点指向x的左节点
    z->left->parent = y; 
    y->left = z->left;
      /******* a *********/
      // y不为z的右节点
    if (y != z->right) {
        // 保存y的父节点
      x_parent = y->parent;
        // y的左或右节点存在(x), 则x的父节点指向y的父节点
      if (x) x->parent = y->parent;
       	// y的父节点指向的左孩子指向x
        // y的右孩子指向z的右孩子
        // z的右孩子的父节点指向y
      y->parent->left = x;      // y must be a left child
      y->right = z->right;
      z->right->parent = y;
    }
      // y是z的右节点
    else
      x_parent = y;  
      
      // z是根节点
    if (root == z)
      root = y;
      // z是父节点的左孩子
    else if (z->parent->left == z)
      z->parent->left = y;	// z的父节点的左孩子指向y
      // z是父节点的右孩子
    else 
      z->parent->right = y;	// z的父节点的右孩子指向y
      // y的父节点指向z的父节点
      // 交换y和z的节点颜色
      // y修改为z
    y->parent = z->parent;
    __STD::swap(y->color, z->color);
    y = z;
    // y now points to node to be actually deleted
  }
    /******* 2 *********/
    // y没有被修改过
  else {                        // y == z
    x_parent = y->parent;
    if (x) x->parent = y->parent;   // x的父节点指向y的父节点
    if (root == z)
      root = x;
    else 
      if (z->parent->left == z)
        z->parent->left = x;
      else
        z->parent->right = x;
    if (leftmost == z) 
      if (z->right == 0)        // z->left must be null also
        leftmost = z->parent;
    // makes leftmost == header if z == root
      else
        leftmost = __rb_tree_node_base::minimum(x);
    if (rightmost == z)  
      if (z->left == 0)         // z->right must be null also
        rightmost = z->parent;  
    // makes rightmost == header if z == root
      else                      // x == z->left
        rightmost = __rb_tree_node_base::maximum(x);
  }
   /************ 1 **************/
    // y节点的颜色不为红色
  if (y->color != __rb_tree_red) { 
      // x不为根节点并且x为空或颜色为黑色
      // 下面的分析与上面一样, 这里就不在做详细的分析了, 只要分析的时候画好图就行了
    while (x != root && (x == 0 || x->color == __rb_tree_black))
      if (x == x_parent->left) {
        __rb_tree_node_base* w = x_parent->right;
        if (w->color == __rb_tree_red) {
          w->color = __rb_tree_black;
          x_parent->color = __rb_tree_red;
          __rb_tree_rotate_left(x_parent, root);
          w = x_parent->right;
        }
        if ((w->left == 0 || w->left->color == __rb_tree_black) &&
            (w->right == 0 || w->right->color == __rb_tree_black)) {
          w->color = __rb_tree_red;
          x = x_parent;
          x_parent = x_parent->parent;
        } else {
          if (w->right == 0 || w->right->color == __rb_tree_black) {
            if (w->left) w->left->color = __rb_tree_black;
            w->color = __rb_tree_red;
            __rb_tree_rotate_right(w, root);
            w = x_parent->right;
          }
          w->color = x_parent->color;
          x_parent->color = __rb_tree_black;
          if (w->right) w->right->color = __rb_tree_black;
          __rb_tree_rotate_left(x_parent, root);
          break;
        }
      } else {                  // same as above, with right <-> left.
        __rb_tree_node_base* w = x_parent->left;
        if (w->color == __rb_tree_red) {
          w->color = __rb_tree_black;
          x_parent->color = __rb_tree_red;
          __rb_tree_rotate_right(x_parent, root);
          w = x_parent->left;
        }
        if ((w->right == 0 || w->right->color == __rb_tree_black) &&
            (w->left == 0 || w->left->color == __rb_tree_black)) {
          w->color = __rb_tree_red;
          x = x_parent;
          x_parent = x_parent->parent;
        } else {
          if (w->left == 0 || w->left->color == __rb_tree_black) {
            if (w->right) w->right->color = __rb_tree_black;
            w->color = __rb_tree_red;
            __rb_tree_rotate_left(w, root);
            w = x_parent->left;
          }
          w->color = x_parent->color;
          x_parent->color = __rb_tree_black;
          if (w->left) w->left->color = __rb_tree_black;
          __rb_tree_rotate_right(x_parent, root);
          break;
        }
      }
    if (x) x->color = __rb_tree_black;
  }
  return y;
}
```



### 总结

本节分析了基本结构, 本节重点掌握左旋, 右旋, 删除, 调整红黑树功能的实现, 掌握了这些下节对红黑树的分析就很轻松了.

