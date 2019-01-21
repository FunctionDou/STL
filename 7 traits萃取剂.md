# traits萃取器

### 前言

前面我们分析了迭代器的五类, 而迭代器所指向对象的型别被称为`value type`. 传入参数的类型可以通过编译器自行推断出来, 但是如果是函数的返回值的话, 就无法通过`value type`让编译器自行推断出来了. 而`traits`就解决了函数返回值类型. 同样原生指针不能内嵌型别声明，所以内嵌型别在这里不适用, 迭代器无法表示原生指针(int *, char *等称为原生指针). 这个问题就通过`traits`偏特化技术解决的. 这一篇我们就主要探讨`traits`是怎么实现这些没有能解决的问题.



### iterator_traits结构

`iterator_traits`结构体就是使用`typename`对参数类型的提取(萃取), 并且对参数类型在进行一次命名, 看上去对参数类型的使用有了一层间接性. 以下就是它的定义.

```c++
template <class Iterator>
struct iterator_traits {
  typedef typename Iterator::iterator_category iterator_category;	//迭代器类型
  typedef typename Iterator::value_type        value_type;			// 迭代器所指对象的类型
  typedef typename Iterator::difference_type   difference_type;		// 两个迭代器之间的距离
  typedef typename Iterator::pointer           pointer;				// 迭代器所指对象的类型指针
  typedef typename Iterator::reference         reference;			// 迭代器所指对象的类型引用
};
```

在五类迭代器对模板对象的类型重新定义一次. 这里提取(萃取)出来的参数类型名都是统一的, 也就说明每个要使用`traits`编程的类必须以此类型名为标准, 而且需要自己对类定义这些类型名.

上面的`traits`结构体并没有对原生指针做处理, 所以还要<font color=#b20>为特化, 偏特化版本(即原生指针)做统一.</font>  以下便是iterator_traits 的特化和偏特化实现

```c++
// 针对原生指针 T* 生成的 traits 偏特化
template <class T>
struct iterator_traits<T*> {
  typedef random_access_iterator_tag iterator_category;
  typedef T                          value_type;
  typedef ptrdiff_t                  difference_type;
  typedef T*                         pointer;
  typedef T&                         reference;
};
// 针对原生指针 const T* 生成的 traits 偏特化
template <class T>
struct iterator_traits<const T*> {
  typedef random_access_iterator_tag iterator_category;
  typedef T                          value_type;
  typedef ptrdiff_t                  difference_type;
  typedef const T*                   pointer;
  typedef const T&                   reference;
};
```

这样不管是函数返回值类型还是原生指针都能通过萃取器萃取出来, `typename I::类型`进行类型萃取.



前面也分析了一下`iterator_category`函数, 现在再来看一下就能明白, 该函数是通过`iterator_traits`萃取的类型的`iterrator_category`确定该迭代器的类型的, 五类迭代器都设置了不同的`iterator_category`的值, 最后调用`category()`函数确定传入参数的类型. 

```c++
template <class Iterator>
inline typename iterator_traits<Iterator>::iterator_category
iterator_category(const Iterator&) {
  typedef typename iterator_traits<Iterator>::iterator_category category;
  return category();
}
// category的五类类型
struct input_iterator_tag {};
struct output_iterator_tag {};
struct forward_iterator_tag : public input_iterator_tag {};
struct bidirectional_iterator_tag : public forward_iterator_tag {};
struct random_access_iterator_tag : public bidirectional_iterator_tag {};
```

继续看`distance`函数, `__distance`接受的前三个参数都是一样, 唯一不一样的就是最后一个参数, 通过`iterator_category`函数萃取出迭代器的类型从而根据类型而执行其对应的`__distance`函数.

```c++
// 根据第三个参数的类型调用相应的重载函数
template <class InputIterator, class Distance>
inline void distance(InputIterator first, InputIterator last, Distance& n) 
{
  	__distance(first, last, n, iterator_category(first));
}
	
template <class InputIterator, class Distance>
inline void __distance(InputIterator first, InputIterator last, Distance& n, 
                       input_iterator_tag) 
{plate <class InputIterator, class Distance>
inline void __distance(InputIterator first, InputIterator last, Distance& n, 
                       input_iterator_tag) 
{
  	while (first != 
  	while (first != last) 
    { ++first; ++n; }
}

template <class RandomAccessIterator, class Distance>
inline void __distance(RandomAccessIterator first, RandomAccessIterator last, 
                       Distance& n, random_access_iterator_tag) 
{
  	n += last - first;
}
```

这里又列出了两个类型的实现, **这里用到了0可以转换成指针的性质, 相当于返回一个空指针, 但是可以通过它们确定不同的参数类型.** 
```c++
template <class Iterator>
inline typename iterator_traits<Iterator>::difference_type*
distance_type(const Iterator&) {
  return static_cast<typename iterator_traits<Iterator>::difference_type*>(0);
}

template <class Iterator>
inline typename iterator_traits<Iterator>::value_type*
value_type(const Iterator&) {
  return static_cast<typename iterator_traits<Iterator>::value_type*>(0);
}
```

`value_type`在空间配置器的时有提过, 就是关于`destory`的第二个版本.

````c++
// 第二个版本的, 接受两个迭代器, 并设法找出元素的类型. 通过__type_trais<> 找出最佳措施
template <class ForwardIterator>
inline void destroy(ForwardIterator first, ForwardIterator last) 
{
  __destroy(first, last, value_type(first));
}
````



### 总结

`traits`编程使用`typename`和特化, 偏特化将迭代器没能支持原生指针, 不能推导出函数返回值的问题完善了.  同时`traits`编程技法对迭代器加以规范, 提前知道了对象的类型相关信息, 从而选择最优的函数执行, 少了类型转化, 提高了执行效率. 



