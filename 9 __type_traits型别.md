# __type_traits型别

### 前言

上一篇探讨的是`traits`是为了将迭代器没能完善的原生指针, `traits`用特化和偏特化编程来完善. 这一篇准备探讨`__type_traits`, 为了将我们在空间配置器里面的提过的`__true_type`和`false_type`进行解答. 而`type_traits`型别对我们STL的效率又有什么影响, 有什么好处?

### __type_traits介绍

前面介绍的Traits技术在STL中弥补了C++模板的不足，但是Traits技术只是用来规范迭代器，对于迭代器之外的东西没有加以规范。因此，SGI将该技术扩展到迭代器之外，称为`__type_traits`。iterator_traits是萃取迭代器的特性，而__type_traits是萃取型别的特性。萃取的型别如下：

-   是否具备non-trivial default ctor?
-   是否具备non-trivial copy ctor?
-   是否具备non-trivial assignment operator?
-   是否具备non-trivial dtor?
-   是否为POD（plain old data）型别？

其中non-trivial意指非默认的相应函数，编译器会为每个类构造以上四种默认的函数，如果没有定义自己的，就会用编译器默认函数，如果使用默认的函数，我们可以使用memcpy(),memmove(),malloc()等函数来加快速度，提高效率. 

且`__iterator_traits`允许针对不同的型别属性在编译期间决定执行哪个重载函数而不是在运行时才处理, 这大大提升了运行效率. 这就需要STL提前做好选择的准备. 是否为**POD, non-trivial型别**用`__true_type`和`__false_type` 来区分.



### 空间配置器的例子

关于`__false_type`也在空间配置器提过. 现在再来看一看. 

```c++
// 接受两个迭代器, 以__type_trais<> 判断是否有traival destructor 
template <class ForwardIterator, class T>
inline void __destroy(ForwardIterator first, ForwardIterator last, T*) 
{
  typedef typename __type_traits<T>::has_trivial_destructor trivial_destructor;
  __destroy_aux(first, last, trivial_destructor());
}
// non-travial destructor 
template <class ForwardIterator>
inline void __destroy_aux(ForwardIterator first, ForwardIterator last, __false_type) 
{
  for ( ; first < last; ++first)
    destroy(&*first);
}
// travial destructor
template <class ForwardIterator> 
inline void __destroy_aux(ForwardIterator, ForwardIterator, __true_type) {}
```

通过函数`trivial_destructor()`确定型别来执行更高效的函数. 而且重载函数的选择实在编译时期就确定了. 马上就来探讨`__false_type`和`__true_type` 参数推导.



### 两个参数推导

```c++
struct __true_type {};
struct __false_type {};
```

我们不能将参数设为bool值, 因为需要在编译期就决定该使用哪个函数, 所以需要利用函数模板的参数推导机制, 将`__true_type`和`__false_type`表现为一个空类, 就不会带来额外的负担, 又能表示真假, 还能在编译时类型推导就确定执行相应的函数.



### __type_traits源码

```c++
__STL_TEMPLATE_NULL struct __type_traits<char> {
   typedef __true_type    has_trivial_default_constructor;
   typedef __true_type    has_trivial_copy_constructor;
   typedef __true_type    has_trivial_assignment_operator;
   typedef __true_type    has_trivial_destructor;
   typedef __true_type    is_POD_type;
};

__STL_TEMPLATE_NULL struct __type_traits<signed char> {
   typedef __true_type    has_trivial_default_constructor;
   typedef __true_type    has_trivial_copy_constructor;
   typedef __true_type    has_trivial_assignment_operator;
   typedef __true_type    has_trivial_destructor;
   typedef __true_type    is_POD_type;
};

...
```

以上是将基础的类型都设置为`__true_type`型别. 

```c++
#ifdef __STL_CLASS_PARTIAL_SPECIALIZATION

template <class T>
struct __type_traits<T*> {
   typedef __true_type    has_trivial_default_constructor;
   typedef __true_type    has_trivial_copy_constructor;
   typedef __true_type    has_trivial_assignment_operator;
   typedef __true_type    has_trivial_destructor;
   typedef __true_type    is_POD_type;
};

#else /* __STL_CLASS_PARTIAL_SPECIALIZATION */

struct __type_traits<char*> {
   typedef __true_type    has_trivial_default_constructor;
   typedef __true_type    has_trivial_copy_constructor;
   typedef __true_type    has_trivial_assignment_operator;
   typedef __true_type    has_trivial_destructor;
   typedef __true_type    is_POD_type;
};

struct __type_traits<signed char*> {
   typedef __true_type    has_trivial_default_constructor;
   typedef __true_type    has_trivial_copy_constructor;
   typedef __true_type    has_trivial_assignment_operator;
   typedef __true_type    has_trivial_destructor;
   typedef __true_type    is_POD_type;
};

struct __type_traits<unsigned char*> {
   typedef __true_type    has_trivial_default_constructor;
   typedef __true_type    has_trivial_copy_constructor;
   typedef __true_type    has_trivial_assignment_operator;
   typedef __true_type    has_trivial_destructor;
   typedef __true_type    is_POD_type;
};
```

这里将指针进行特化处理, 同样是`__true_type`型别.

SGI将所有的内嵌型别都定义为`false_type`, 这是对所有的定义最保守的值.

```c++
template <class type>
struct __type_traits { 
   typedef __true_type     this_dummy_member_must_be_first;
   typedef __false_type    has_trivial_default_constructor;
   typedef __false_type    has_trivial_copy_constructor;
   typedef __false_type    has_trivial_assignment_operator;
   typedef __false_type    has_trivial_destructor;
   typedef __false_type    is_POD_type;
};
```

从上面的源码可以明白, 所有的基本类型都是设置的为`__true_type`型别, 而所有的对象都会被特化为`__false_type`型别, 这是为了保守而不得这样做. 也因为`__true_type`型别让我们在执行普通类型时能够以最大效率进行对其的copy, 析构等操作.



### 总结

SGI对`traits`进行扩展，使得所有类型都满足`traits`编程规范, 这样SGI STL算法可以通过`__type_traits`获取类型信息在编译期间就能决定出使用哪一个重载函数, 解决了`template`是在运行时决定重载选择的问题. 并且通过`true`和`false`来确定POD和travial destructor, 让程序能选择更加符合其类型的处理函数, 大大提高了对基本类型的快速处理能力并保证了效率最高.