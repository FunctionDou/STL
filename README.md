# SIG STL源码分析

## 前言

本专栏主要以STL源码剖析分析路线来分析SIGSTL3.0源码.

整个模块准备对学习`STL源码剖析`之后做一个系统的总结, 这些都是我个人的理解, 如果分析有什么问题欢迎各位大佬们指出. 也很感谢作者以及网络中各个大佬的总结, 让我也能更容易更深刻的理解到`STL`强大和方便, 也让我对`template`感受深刻.

以下是我自己对STL版块进行分析. 

**总共分为六个版块 : 空间配置器, 迭代器, 容器(序列容器, 关联容器),  算法, 仿函数, 配接器.**



## STL前期准备

在学习STL源码之前需要对`template`有一个认识或者回忆.

[template(一)](https://github.com/FunctionDou/STL/blob/master/template%E4%B9%8B%E6%A8%A1%E6%9D%BF%E6%B3%A8%E6%84%8F%E4%BA%8B%E9%A1%B9.md)

[template(二)](https://github.com/FunctionDou/STL/blob/master/template%E4%B9%8B%E9%9D%9E%E7%B1%BB%E5%9E%8B%E6%A8%A1%E6%9D%BF%E5%8F%82%E6%95%B0.md)

[template(三)](https://github.com/FunctionDou/STL/blob/master/template%E4%B9%8B%E7%B1%BB%E7%9B%B8%E5%85%B3.md)

## STL分析



### 空间配置器

c/c++都需要手动的管理内存, 而封装实现一个能申请空间又能自己释放空间不再让我们自己管理. 而STL就实现了这样的一个功能, 它就是`空间配置器`. 而空间配置器一般是隐藏在各个版块的组件, 实现中我们都看不到它的存在, 但是它确实是非常重要的部分, 因为它, 所有的版块操作内存时都直接调用它就行了, 而不需要再实现内存的分配. 

[0. new实现](https://blog.csdn.net/Function_Dou/article/details/84526761)

[1. 空间配置器](https://github.com/FunctionDou/STL/blob/master/1%20%E5%88%9D%E6%AC%A1%E6%8E%A5%E8%A7%A6%E7%A9%BA%E9%97%B4%E9%85%8D%E7%BD%AE%E5%99%A8.md)

[2. 第一级配置器](https://github.com/FunctionDou/STL/blob/master/2%20第一级配置器.md)

[3. 第二级配置器](https://github.com/FunctionDou/STL/blob/master/3%20%E7%AC%AC%E4%BA%8C%E7%BA%A7%E9%85%8D%E7%BD%AE%E5%99%A8.md)

[4. 内存池](https://github.com/FunctionDou/STL/blob/master/4%20%E5%86%85%E5%AD%98%E6%B1%A0.md)



### 迭代器

每个组件都可能会涉及到对元素的简单操作, 比如 : 访问, 自增, 自减等. 每个组件都的数据类型可能不同, 所以每个组件可能都要自己设计对自己的操作. 但将每个组件的实现成统一的接口就是迭代器, 

它的优点也很明显: 

1.  它是能屏蔽掉底层数据类型差异的. 
2.  迭代器将容器和算法粘合在一起, 使版块之间更加的紧凑, 同时提高了执行效率, 让算法更加的得到优化.

这些实现大都通过`traits`编程实现的.  它的定义了一个类型名规则, 满足`traits`编程规则就可以自己实现对`STL`的扩展, 也体现了`STL`的灵活性. 同时`straits`编程让程序根据不同的参数类型选择执行更加合适参数类型的处理函数, 也就提高了`STL`的执行效率. 可见迭代器对`STL`的重要性.

[1. 迭代器](https://github.com/FunctionDou/STL/blob/master/5%20%E8%BF%AD%E4%BB%A3%E5%99%A8.md)

[2. template(四)](https://github.com/FunctionDou/STL/blob/master/6%20%E6%A8%A1%E6%9D%BF%E4%B8%ADclass%E4%B8%8Etypename%E5%8C%BA%E5%88%AB.md)

[3. traits萃取剂](https://github.com/FunctionDou/STL/blob/master/7%20traits%E8%90%83%E5%8F%96%E5%89%82.md)

[4. template(五)](https://github.com/FunctionDou/STL/blob/master/8%20%E5%85%A8%E7%89%B9%E5%8C%96%E5%92%8C%E5%81%8F%E7%89%B9%E5%8C%96.md)

[5. __type_traits型别](https://github.com/FunctionDou/STL/blob/master/9%20__type_traits%E5%9E%8B%E5%88%AB.md)



### 容器

容器是封装了大量常用的数据结构, 因为容器, 凸显出STL的方便, 操作简单. 毕竟它将常用的但是实现比较麻烦的数据结构封装之后就可以直接的调用, 不再让用户重写一长串的代码实现. 

容器根据排列分为了**序列式和关联式**. 

1.  序列式包括`vector`, `list`, `deque`等. 序列容器有头或尾, 甚至有头有尾.
2.  关联式包括`map`, `set`, `hashtable`等. 关联容器没有所谓的头尾, 只有最大值, 最小值.

学习容器的时候要注意`end`返回的是**最后一元素的后一个地址, 这个地址并没有储存实际的值.** 

[0. uninitialized系列函数](https://github.com/FunctionDou/STL/blob/master/10%20uninitialized.md)

##### 序列容器

[1. vector序列容器(一)](https://github.com/FunctionDou/STL/blob/master/11%20vector%20%E4%B8%8A.md)

[2. vector序列容器(二)](https://github.com/FunctionDou/STL/blob/master/12%20vector%20%E4%B8%AD.md)

[3. vector序列容器(三)](https://github.com/FunctionDou/STL/blob/master/13%20vector%20%E4%B8%8B.md)

[4. list有序容器(一)](https://github.com/FunctionDou/STL/blob/master/14%20list%20%E4%B8%8A.md)

[5. list有序容器(二)](https://github.com/FunctionDou/STL/blob/master/15%20list%20%E4%B8%AD.md)

[6. list有序容器(三)](https://github.com/FunctionDou/STL/blob/master/16%20list%20%E4%B8%8B.md)

[7. deque有序容器(一)](https://github.com/FunctionDou/STL/blob/master/17%20deque%20%E4%B8%8A.md)

[8. deque有序容器(二)](https://github.com/FunctionDou/STL/blob/master/18%20deque%20%E4%B8%AD.md)

[9. deque有序容器(三)](https://github.com/FunctionDou/STL/blob/master/19%20deque%20%E4%B8%8B.md)

[10. stack配接器](https://github.com/FunctionDou/STL/blob/master/20%20stack.md)

[11. queue配接器](https://github.com/FunctionDou/STL/blob/master/21%20queue.md)

[12. heap大根堆](https://github.com/FunctionDou/STL/blob/master/22%20heap.md)

[13. 优先级队列](https://github.com/FunctionDou/STL/blob/master/23%20priority_queue.md)

[14. slist有序容器(一)](https://github.com/FunctionDou/STL/blob/master/24%20slist%20%E4%B8%8A.md)

[15. slist有序容器(二)](https://github.com/FunctionDou/STL/blob/master/25%20slist%20下.md)



##### 关联容器

[1. RB-tree关联容器(一)](https://github.com/FunctionDou/STL/blob/master/26%20RB-tree%20%E4%B8%8A.md)

[2. RB-tree关联容器(二)](https://github.com/FunctionDou/STL/blob/master/27%20RB-tree%20%E4%B8%8B.md)

[3. set配接器](https://github.com/FunctionDou/STL/blob/master/28%20set.md)

[4. pair结构体](https://github.com/FunctionDou/STL/blob/master/29%20pair.md)

[5. map配接器](https://github.com/FunctionDou/STL/blob/master/30%20map.md)

[6. multiset配接器](https://github.com/FunctionDou/STL/blob/master/31%20multiset.md)

[7. multimap配接器](https://github.com/FunctionDou/STL/blob/master/32%20multimap.md)

[8. hashtable关联容器(一)](https://github.com/FunctionDou/STL/blob/master/33%20hashtable%20%E4%B8%8A.md)

[9. hashtable关联容器(二)](https://github.com/FunctionDou/STL/blob/master/34%20hashtable%20%E4%B8%8B.md)

[10. hash_set配接器](https://github.com/FunctionDou/STL/blob/master/35%20hash_set.md)

[11. hash_multiset配接器](https://github.com/FunctionDou/STL/blob/master/36%20hash_multiset.md)

[12. hash_map配接器](https://github.com/FunctionDou/STL/blob/master/37%20hash_map.md)

[13. hash_multimap配接器](https://github.com/FunctionDou/STL/blob/master/38%20hash_multimap.md)



### 算法

STL的算法有很多, 有简单也有一些复杂的, 这里我就以`STL3.0`源码为例, 挑选出来几个常用的算法来进行分析. 

[1. copy算法](https://github.com/FunctionDou/STL/blob/master/39%20%E7%AE%97%E6%B3%95--copy.md)

### 仿函数

所谓仿函数也就是函数对象, 以前是这样称呼它的, 只是一直沿用至今了. **仿函数就是一种具有函数特质的对象**. STL因为仿函数, 大大在增加了灵活性, 而且可以将部分操作由用户自己来定义然后传入自定义的函数名就可以被调用. 但是你会发现仿函数实现的功能都是相当简单的, 而且都要通过配接器再封装才能够使用.

STL的仿函数根据参数个数可以分为 : 一元仿函数和二元仿函数. 根据功能可分为 : 算术运算, 关系运算和逻辑运算.

[1. 仿函数](https://github.com/FunctionDou/STL/blob/master/44%20%E4%BB%BF%E5%87%BD%E6%95%B0.md)



### 配接器

主要用来修改接口. 修改迭代器的接口, 修改容器的接口, 修改仿函数的接口.

[1. 配接器](https://github.com/FunctionDou/STL/blob/master/45%20%E9%85%8D%E6%8E%A5%E5%99%A8.md)



## 总结

STL这本书对我这个菜鸟收获还是挺有帮助的, 了解了`template`的强大, STL对程序做到了最大的优化. STL对每个功能都尽可能的单一, 然后可能通过多次的函数调用才调用真正执行的函数.

让我对c++有了一个更深的认识, 真的很难, 自己还差的太远, 努力加油!
