# rotate

### 前言

`stl_algo.h`文件中有很多的算法实现, 在这里STL分析中挑选几个函数进行分析, 其他的算法大家有兴趣可以自己看看, 本节分析`stl_algo.h`文件中的`rotate`算法. 该算法实现的功能是将[first, middle),[middle, last)两段区间的元素进行交换.



### rotate分析

```c++
template <class ForwardIterator>
inline void rotate(ForwardIterator first, ForwardIterator middle, ForwardIterator last) 
{
  if (first == middle || middle == last) return;
    // 三个迭代器 : first, middle, last
  __rotate(first, middle, last, distance_type(first), iterator_category(first));
}
```



**forward_iterator_tag** 版本

需要注意 : 

1.  first是前段部分的指针
2.  i 表示的是后段部分的指针

```c++
template <class ForwardIterator, class Distance>
void __rotate(ForwardIterator first, ForwardIterator middle,
              ForwardIterator last, Distance*, forward_iterator_tag) 
{
    // 这是一个死循环, 直到满足必要条件才会退出
  for (ForwardIterator i = middle; ;) {
      // 从middle部分开始, 将两段的元素依次交换, 直到有一段被交换完才进行修改
    iter_swap(first, i);
    ++first;
    ++i;
      // 前段部分先交换完
    if (first == middle) {
        // 直到所有数据都交换完才退出
      if (i == last) return;
        // 重新修改middle的值, 让前段从之前的first = old_middle继续开始交换
      middle = i;
    }
      // 后段部分先交换完
    else if (i == last)
        // 重新修改i的值, 让后段从middle继续开始交换
      i = middle;
  }
}
```

整个交换的时间复杂度为O(n), 没有空间上的开销.



**bidirectional_iterator_tag** 版本

```c++
template <class BidirectionalIterator, class Distance>
void __rotate(BidirectionalIterator first, BidirectionalIterator middle,
              BidirectionalIterator last, Distance*,
              bidirectional_iterator_tag) {
  reverse(first, middle);
  reverse(middle, last);
  reverse(first, last);
}
```



**random_access_iterator_tag** 版本

采用的是最大公因子减少时间复杂度

```c++
template <class RandomAccessIterator, class Distance>
void __rotate(RandomAccessIterator first, RandomAccessIterator middle,
              RandomAccessIterator last, Distance*,
              random_access_iterator_tag) {
  Distance n = __gcd(last - first, middle - first);
  while (n--)
    __rotate_cycle(first, last, first + n, middle - first, value_type(first));
}
template <class RandomAccessIterator, class Distance, class T>
void __rotate_cycle(RandomAccessIterator first, RandomAccessIterator last,
                    RandomAccessIterator initial, Distance shift, T*) {
  T value = *initial;	
  RandomAccessIterator ptr1 = initial;
  RandomAccessIterator ptr2 = ptr1 + shift;
  while (ptr2 != initial) {
    *ptr1 = *ptr2;
    ptr1 = ptr2;
    if (last - ptr2 > shift)
      ptr2 += shift;
    else
      ptr2 = first + (shift - (last - ptr2));
  }
  *ptr1 = value;
}
```

```c++
// 求最大公因子
template <class EuclideanRingElement>
EuclideanRingElement __gcd(EuclideanRingElement m, EuclideanRingElement n)
{
  while (n != 0) {
    EuclideanRingElement t = m % n;
    m = n;
    n = t;
  }
  return m;
}
```



### 总结

本节分析了`rotate`实现两段元素进行交换, 整体实现并不难, 但是需要考虑不同的迭代器类型选择最优的函数实现, 最大的提高效率问题. 下一节我们将继续分析`stl_algo.h`中的其他基本的算法.