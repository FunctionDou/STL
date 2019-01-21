# 内存处理工具

[TOC]

----



### 前言

这里将`内存处理工具`放到这里来讲是因为必须对`__type_traits`类型定义有所了解, 对`traits`编程有所了解才行.(如果你还没有了解这些, 希望能去看一下前面4篇的文章). 而且容器实现的很多地方都会用到`uninitialized_copy`的函数, 虽然乍一看功能应该是执行复制操作, 但如果第一次看或者事先没有理解该函数怎么工作的可能会影响到部分人分析代码的效率, 所以在这里就提前对其进行分析. 



### uninitialized_copy函数

`uninitialized_copy`功能 : 从first到last范围内的元素复制到从 result地址开始的内存.

```c++
template <class InputIterator, class ForwardIterator>
inline ForwardIterator uninitialized_copy(InputIterator first, InputIterator last,
                     ForwardIterator result) {
  return __uninitialized_copy(first, last, result, value_type(result));
}
```

很明显这就只是调用另一个函数的接口而已.

`uninitialized_copy`类似于第一篇分析空间配置器的`destory`针对`const char*`和`const wchar*`单独做了特例化.

```c++
inline char* uninitialized_copy(const char* first, const char* last,
                                char* result) {
  memmove(result, first, last - first);
  return result + (last - first);
}

inline wchar_t* uninitialized_copy(const wchar_t* first, const wchar_t* last,
                                   wchar_t* result) {
  memmove(result, first, sizeof(wchar_t) * (last - first));
  return result + (last - first);
}
```

直接调用c++的`memmove`操作, 毕竟这样的效率更加的高效.



#### __uninitialized_copy函数

```c++
template <class InputIterator, class ForwardIterator, class T>
inline ForwardIterator __uninitialized_copy(InputIterator first, InputIterator last,
                     ForwardIterator result, T*) {
  typedef typename __type_traits<T>::is_POD_type is_POD;
  return __uninitialized_copy_aux(first, last, result, is_POD());
}
```

`__uninitialized_copy`使用了`typename`进行萃取, 并且萃取的类型是`POD`, 看来这里准备对`__uninitialized_copy` 进行最优化处理了, 我们接着来分析它是怎么实现优化处理的. 



#### __uninitialized_copy_aux优化处理

```c++
template <class InputIterator, class ForwardIterator> 
inline ForwardIterator __uninitialized_copy_aux(InputIterator first, InputIterator last,
                         ForwardIterator result,
                         __true_type) {
  return copy(first, last, result);
}

template <class InputIterator, class ForwardIterator>
ForwardIterator __uninitialized_copy_aux(InputIterator first, InputIterator last,
                         ForwardIterator result,
                         __false_type) {
  ForwardIterator cur = result;
  __STL_TRY {
    for ( ; first != last; ++first, ++cur)
      construct(&*cur, *first);
    return cur;
  }
  __STL_UNWIND(destroy(result, cur));
}
```

`__uninitialized_copy`针对普通类型(int, double)做了特殊的优化, 以执行更高效的处理, 对类和用户定义类型做了构造处理, 当然用户自定义的不一定是类, 但是编译器为了安全性依然会执行最慢处理.



### uninitialized_copy_n函数

`uninitialized_copy_n`也做了跟`uninitialized_copy`类似的处理, 只是它是采用`tratis`编程里`iterator_category`迭代器的类型来选择最优的处理函数.

````c++
template <class InputIterator, class Size, class ForwardIterator>
inline pair<InputIterator, ForwardIterator>
uninitialized_copy_n(InputIterator first, Size count,
                     ForwardIterator result) {
  return __uninitialized_copy_n(first, count, result, iterator_category(first)); // 根据iterator_category选择最优函数
}

template <class InputIterator, class Size, class ForwardIterator>
pair<InputIterator, ForwardIterator>
__uninitialized_copy_n(InputIterator first, Size count, ForwardIterator result,
                       input_iterator_tag) // input_iterator_tag类型的迭代器
{
  ForwardIterator cur = result;
  __STL_TRY {
    for ( ; count > 0 ; --count, ++first, ++cur) 
      construct(&*cur, *first);
    return pair<InputIterator, ForwardIterator>(first, cur);
  }
  __STL_UNWIND(destroy(result, cur));
}

template <class RandomAccessIterator, class Size, class ForwardIterator>
inline pair<RandomAccessIterator, ForwardIterator>
__uninitialized_copy_n(RandomAccessIterator first, Size count, ForwardIterator result,
                       random_access_iterator_tag) // random_access_iterator_tag类型的迭代器
{
  RandomAccessIterator last = first + count;
  return make_pair(last, uninitialized_copy(first, last, result));
}
````



### uninitialized_fill函数

`uninitialized_fill`功能 : 从first到last范围内的都填充为 x 的值.

`uninitialized_fill`采用了与`uninitialized_copy`一样的处理方法选择最优处理函数, 这里就不过多的分析了.

```c++
template <class ForwardIterator, class T>
inline void uninitialized_fill(ForwardIterator first, ForwardIterator last, const T& x) 
{
  __uninitialized_fill(first, last, x, value_type(first));
}

template <class ForwardIterator, class T, class T1>
inline void __uninitialized_fill(ForwardIterator first, ForwardIterator last,  const T& x, T1*)
{
  typedef typename __type_traits<T1>::is_POD_type is_POD;
  __uninitialized_fill_aux(first, last, x, is_POD());
                   
}

template <class ForwardIterator, class T>
inline void
__uninitialized_fill_aux(ForwardIterator first, ForwardIterator last,  const T& x, __true_type)
{
   fill(first, last, x);
}

template <class ForwardIterator, class T>
void
__uninitialized_fill_aux(ForwardIterator first, ForwardIterator last, const T& x, __false_type)
{
  ForwardIterator cur = first;
  __STL_TRY {
    for ( ; cur != last; ++cur)
      construct(&*cur, x);
  }
  __STL_UNWIND(destroy(first, cur));
}
```



#### uninitialized_fill_n函数

`uninitialized_fill_n`功能 : 从first开始n 个元素填充成 x 值.

```c++
template <class ForwardIterator, class Size, class T>
inline ForwardIterator uninitialized_fill_n(ForwardIterator first, Size n, const T& x) 
{
  return __uninitialized_fill_n(first, n, x, value_type(first));
}

template <class ForwardIterator, class Size, class T, class T1>
inline ForwardIterator __uninitialized_fill_n(ForwardIterator first, Size n, const T& x, T1*) 
{
  typedef typename __type_traits<T1>::is_POD_type is_POD;
  return __uninitialized_fill_n_aux(first, n, x, is_POD());                                
}

template <class ForwardIterator, class Size, class T>
inline ForwardIterator __uninitialized_fill_n_aux(ForwardIterator first, Size n, const T& x, __true_type) 
{
  return fill_n(first, n, x);
}

template <class ForwardIterator, class Size, class T>
ForwardIterator __uninitialized_fill_n_aux(ForwardIterator first, Size n, const T& x, __false_type) 
{
  	ForwardIterator cur = first;
  	__STL_TRY 
  	{
   		for ( ; n > 0; --n, ++cur)
     	 	construct(&*cur, x);
    	return cur;
  	}
  	__STL_UNWIND(destroy(first, cur));
}
```



### 总结

`uninitialized_copy`是为两段内存进行复制的函数, `uninitialized_fill`是为对一段内存进行初始化一个值的函数. 两者都对了`traits`编程中的迭代器类型和`__type_traits`定义的`__false_type`和`__true_type`的不同执行不同的处理函数, 也使效率最优化.