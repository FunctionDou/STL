# 算法--copy

### 前言

在前面分析顺序容器和关联容器时, 总会遇到`copy`这个函数, 当时并没有去分析这个函数, 毕竟都能知道他是什么功能, 本节就来揭开它面纱.



### copy分析

copy函数源码在`stl_algobase.h`中, 该结构中还有很多其他的算法实现, 我只是从中挑选出了copy, 有兴趣可以自己看看里面的其他算法.

`copy`同`traits`编程, 都对性能做了最优的优化, 能使用的`memmove`函数就使用它, 不能通过迭代器类型再来选择最优处理.

**在偏特化与全特化中分析过, 最适合的函数会优先调用, 普通函数优先级大于模板函数**

```c++
template <class InputIterator, class OutputIterator>
inline OutputIterator copy(InputIterator first, InputIterator last, OutputIterator result)
{
  return __copy_dispatch<InputIterator,OutputIterator>()(first, last, result);
}
// 重载
// 在偏特化与全特化中分析过, 最适合的函数会优先调用, 普通函数优于模板函数
inline char* copy(const char* first, const char* last, char* result) {
    // 直接调用memmove效率最高
  memmove(result, first, last - first);
  return result + (last - first);
}
inline wchar_t* copy(const wchar_t* first, const wchar_t* last, wchar_t* result) {
    // 直接调用memmove效率最高
  memmove(result, first, sizeof(wchar_t) * (last - first));
  return result + (last - first);
}
```

**__copy_dispatch** 通过传入参数的迭代器类型再进行优化处理

```c++
template <class InputIterator, class OutputIterator>
struct __copy_dispatch
{
  OutputIterator operator()(InputIterator first, InputIterator last,
                            OutputIterator result) {
      // iterator_category获取迭代器类型, 不同迭代器选择不同的重载函数
    return __copy(first, last, result, iterator_category(first));
  }
};
```



**输入迭代器类型处理 input_iterator_tag**

```c++
template <class InputIterator, class OutputIterator>
inline OutputIterator __copy(InputIterator first, InputIterator last,
                             OutputIterator result, input_iterator_tag)
{
    // 通过迭代器将一个元素一个元素的复制
  for ( ; first != last; ++result, ++first)
    *result = *first;
  return result;
}
```



**随机访问迭代器类型处理 random_access_iterator_tag**

```c++
template <class RandomAccessIterator, class OutputIterator>
inline OutputIterator 
__copy(RandomAccessIterator first, RandomAccessIterator last, OutputIterator result, random_access_iterator_tag)
{
  return __copy_d(first, last, result, distance_type(first));
}

template <class RandomAccessIterator, class OutputIterator, class Distance>
inline OutputIterator
__copy_d(RandomAccessIterator first, RandomAccessIterator last,
         OutputIterator result, Distance*)
{
    // 通过迭代器之间的元素个数将一个元素一个元素的复制
  for (Distance n = last - first; n > 0; --n, ++result, ++first) 
    *result = *first;
  return result;
}
```

`random_access_iterator_tag`与`input_iterator_tag`不同就在于前者不使用迭代器遍历, 后者使用的是迭代器访问. 使用迭代器的效率要低一点, 毕竟可能是RB-tree的迭代器, 链表的迭代器之类的.



**全特化处理** 针对指针, `const`类型又做了特化处理

```c++
template <class T>
struct __copy_dispatch<const T*, T*>
{
  T* operator()(const T* first, const T* last, T* result) {
    typedef typename __type_traits<T>::has_trivial_assignment_operator t; 
    return __copy_t(first, last, result, t());
  }
};

template <class T>
struct __copy_dispatch<T*, T*>
{
  T* operator()(T* first, T* last, T* result) {
    typedef typename __type_traits<T>::has_trivial_assignment_operator t; 
    return __copy_t(first, last, result, t());
  }
};
```



优化处理

```c++
template <class T>
inline T* __copy_t(const T* first, const T* last, T* result, __false_type) {
  return __copy_d(first, last, result, (ptrdiff_t*) 0);
}
```

优化处理

```c++
template <class T>
inline T* __copy_t(const T* first, const T* last, T* result, __true_type) {
  memmove(result, first, sizeof(T) * (last - first));
  return result + (last - first);
}
```



**针对pair类型**

```c++
template <class InputIterator, class Size, class OutputIterator>
pair<InputIterator, OutputIterator> __copy_n(InputIterator first, Size count,
                                             OutputIterator result,
                                             input_iterator_tag) {
  for ( ; count > 0; --count, ++first, ++result)
    *result = *first;
  return pair<InputIterator, OutputIterator>(first, result);
}

template <class RandomAccessIterator, class Size, class OutputIterator>
inline pair<RandomAccessIterator, OutputIterator>
__copy_n(RandomAccessIterator first, Size count,
         OutputIterator result,
         random_access_iterator_tag) {
  RandomAccessIterator last = first + count;
  return pair<RandomAccessIterator, OutputIterator>(last,
                                                    copy(first, last, result));
}
```



### 总结

本节分析了`copy`对优化的处理, 针对不同的类型, 不同迭代器选择最优的处理函数, 提高程序的效率, STL的强大到处可以体现.	`__copy_backward`我没有做分析, 他同`copy`类似, 只是反向进行复制.

