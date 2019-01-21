# heap最大堆

###  前言

在分析本节之前你至少应该对堆排序有所了解, 大根堆, 小根堆等. 本节分析的`heap`就是堆排序, 严格意义上来讲heap并不是一个容器, 所以他没有实现自己的迭代器, 也就没有遍历操作, 它只是一种算法. 代码来自stl_heap.h.



### heap分析



#### push插入元素

插入函数是`push_heap`. `heap`只接受`RandomAccessIterator`类型的迭代器. 

注意, 在分析heap的时候最好还是自己画一次, 画一个数组一个二叉树.

```c++
template <class RandomAccessIterator>
inline void push_heap(RandomAccessIterator first, RandomAccessIterator last) {
  __push_heap_aux(first, last, distance_type(first), value_type(first));
}

template <class RandomAccessIterator, class Distance, class T>
inline void __push_heap_aux(RandomAccessIterator first, RandomAccessIterator last, Distance*, T*) 
{
    // 这里传入的是两个迭代器的长度, 0, 还有最后一个数据
  __push_heap(first, Distance((last - first) - 1), Distance(0),  T(*(last - 1)));
}
```

**push的核心代码**

```c++
template <class RandomAccessIterator, class Distance, class T>
void __push_heap(RandomAccessIterator first, Distance holeIndex,Distance topIndex, T value) 
{
    // 这里就行二分, 因为二叉树每一行都是2的倍数
  Distance parent = (holeIndex - 1) / 2;
    // 这里判断的是当前没有达到堆顶并且传入的值大于根节点的值, 那就将根节点下移
  while (holeIndex > topIndex && *(first + parent) < value) {
      // 将根节点下移
    *(first + holeIndex) = *(first + parent);
    holeIndex = parent;
    parent = (holeIndex - 1) / 2;
  }    
    
    // 将数组插入到合适的位置, 可能是根也可能是叶
  *(first + holeIndex) = value;
}
```



#### pop弹出元素

pop操作其实并没有真正意义去删除数据, 而是将数据放在最后, 只是没有指向最后的元素而已, 这里arrary也可以使用, 毕竟没有对数组的大小进行调整.  pop的实现有两种, 这里都罗列了出来, 另一个传入的是cmp伪函数.

```c++
template <class RandomAccessIterator, class Compare>
inline void pop_heap(RandomAccessIterator first, RandomAccessIterator last,
                     Compare comp) {
    __pop_heap_aux(first, last, value_type(first), comp);
}
template <class RandomAccessIterator, class T, class Compare>
inline void __pop_heap_aux(RandomAccessIterator first,
                           RandomAccessIterator last, T*, Compare comp) {
  __pop_heap(first, last - 1, last - 1, T(*(last - 1)), comp,
             distance_type(first));
}
template <class RandomAccessIterator, class T, class Compare, class Distance>
inline void __pop_heap(RandomAccessIterator first, RandomAccessIterator last,
                       RandomAccessIterator result, T value, Compare comp,
                       Distance*) {
  *result = *first;
  __adjust_heap(first, Distance(0), Distance(last - first), value, comp);
}
template <class RandomAccessIterator, class T, class Distance>
inline void __pop_heap(RandomAccessIterator first, RandomAccessIterator last,
                       RandomAccessIterator result, T value, Distance*) {
  *result = *first;	// 因为这里是大根堆, 所以first的值就是最大值, 先将最大值保存.
  __adjust_heap(first, Distance(0), Distance(last - first), value);
}

```

**pop的核心函数**. 这里主要之分析第一个版本.

pop弹出的是二叉树的最下一排的数据. 

```c++
template <class RandomAccessIterator, class Distance, class T>
void __adjust_heap(RandomAccessIterator first, Distance holeIndex, Distance len, T value) 
{
    // holeIndex传入的是0
  Distance topIndex = holeIndex;
    // secondChild是右孩子的一个节点
  Distance secondChild = 2 * holeIndex + 2;
  while (secondChild < len) {
      // 比较左右节点, 根节点较下就将根节点下移, 比较大的节点上移
    if (*(first + secondChild) < *(first + (secondChild - 1)))
      secondChild--;
    *(first + holeIndex) = *(first + secondChild);
    holeIndex = secondChild;
      // 下一个左右节点
    secondChild = 2 * (secondChild + 1);
  }
  if (secondChild == len) {
      // 没有右节点就找左节点并且上移
    *(first + holeIndex) = *(first + (secondChild - 1));
    holeIndex = secondChild - 1;
  }
    // 重新调整堆
  __push_heap(first, holeIndex, topIndex, value);
}
// cmpare版本只将比较修改成用户定义的函数
template <class RandomAccessIterator, class Distance, class T, class Compare>
void __adjust_heap(RandomAccessIterator first, Distance holeIndex,
                   Distance len, T value, Compare comp) {
  Distance topIndex = holeIndex;
  Distance secondChild = 2 * holeIndex + 2;
  while (secondChild < len) {
    if (comp(*(first + secondChild), *(first + (secondChild - 1))))
      secondChild--;
    *(first + holeIndex) = *(first + secondChild);
    holeIndex = secondChild;
    secondChild = 2 * (secondChild + 1);
  }
  if (secondChild == len) {
    *(first + holeIndex) = *(first + (secondChild - 1));
    holeIndex = secondChild - 1;
  }
  __push_heap(first, holeIndex, topIndex, value, comp);
}
```

**make_heap函数, 将数组变为堆存放**.

```c++
template <class RandomAccessIterator>
inline void make_heap(RandomAccessIterator first, RandomAccessIterator last) {
  __make_heap(first, last, value_type(first), distance_type(first));
}
template <class RandomAccessIterator, class T, class Distance>
void __make_heap(RandomAccessIterator first, RandomAccessIterator last, T*,
                 Distance*) {
  if (last - first < 2) return;
    // 计算长度, 并找出中间的根值
  Distance len = last - first;
  Distance parent = (len - 2)/2;
    
  while (true) {
      // 一个个进行调整, 放到后面
    __adjust_heap(first, parent, len, T(*(first + parent)));
    if (parent == 0) return;
    parent--;
  }
}
```

**sort, 堆排序其实就是每次将第一位数据弹出从而实现排序功能**.

```c++
template <class RandomAccessIterator>
void sort_heap(RandomAccessIterator first, RandomAccessIterator last) {
  while (last - first > 1) pop_heap(first, last--);
}
template <class RandomAccessIterator, class Compare>
void sort_heap(RandomAccessIterator first, RandomAccessIterator last,
               Compare comp) {
  while (last - first > 1) pop_heap(first, last--, comp);
}
```



### 总结

`heap`没有自己的迭代器，只要支持`RandomAccessIterator`的容器都可以作为Heap容器. `heap`最重要的函数还是`pop`和`push`的实现.