# priority_queue

### 前言

上一节分析`heap`其实就是为`priority_queue`做准备. `priority_queue`是一个优先级队列, 是带权值的. 支持插入和删除操作, 其只能从尾部插入,头部删除, 并且其顺序也并非是根据加入的顺序排列的. `priority_queue`因为也是队列的一种体现, 所以也就跟队列一样不能直接的遍历数组, 也就没有迭代器. `priority_queue`本身也不算是一个容器, 它是以`vector`为容器以`heap`为数据操作的**配置器**.



### 源码分析



#### 类型定义

```c++
#ifndef __STL_LIMITED_DEFAULT_TEMPLATES
template <class T, class Sequence = vector<T>, 
          class Compare = less<typename Sequence::value_type> >
#else
template <class T, class Sequence, class Compare>
#endif
class  priority_queue {
public:
	// 符合traits编程规范
  typedef typename Sequence::value_type value_type;
  typedef typename Sequence::size_type size_type;
  typedef typename Sequence::reference reference;
  typedef typename Sequence::const_reference const_reference;
protected:
  Sequence c;	// 定义vector容器的对象
  Compare comp;	// 定义比较函数(伪函数)
  ...
};
```



#### 构造函数

```c++
class  priority_queue {
	...
public:
  priority_queue() : c() {}	// 默认构造函数
  explicit priority_queue(const Compare& x) :  c(), comp(x) {}	// 设置伪函数

#ifdef __STL_MEMBER_TEMPLATES
    // 接受以迭代器类型的参数
    // 接受两个迭代器以及函数. 传入的迭代器范围内表示的元素以comp定义的方式进行调整
  template <class InputIterator>
  priority_queue(InputIterator first, InputIterator last, const Compare& x)
    : c(first, last), comp(x) { make_heap(c.begin(), c.end(), comp); }
    // 接受两个迭代器. 传入的迭代器范围内表示的元素以默认的大根堆进行调整
  template <class InputIterator>
  priority_queue(InputIterator first, InputIterator last) 
    : c(first, last) { make_heap(c.begin(), c.end(), comp); }
#else /* __STL_MEMBER_TEMPLATES */
     // 接受两个迭代器以及函数. 传入的迭代器范围内表示的元素以comp定义的方式进行调整
  priority_queue(const value_type* first, const value_type* last, 
                 const Compare& x) : c(first, last), comp(x) {
    make_heap(c.begin(), c.end(), comp);
  }
    // 接受两个迭代器. 传入的迭代器范围内表示的元素以默认的大根堆进行调整
  priority_queue(const value_type* first, const value_type* last) 
    : c(first, last) { make_heap(c.begin(), c.end(), comp); }
#endif /* __STL_MEMBER_TEMPLATES */
	...
};
```



#### 属性获取

`priority_queue`只有简单的3个属性获取的函数, 其本身的操作也很简单, 只是实现依赖了`vector`和`heap`就变得比较复杂.

```c++
class  priority_queue {
	...
public:
  bool empty() const { return c.empty(); }
  size_type size() const { return c.size(); }
  const_reference top() const { return c.front(); }
    ...
};
```

#### push和pop实现

push和pop具体都是采用的`heap`算法.

```c++
class  priority_queue {
	...
public:
  void push(const value_type& x) {
    __STL_TRY {
      c.push_back(x); 
        // 间接使用heap算法
      push_heap(c.begin(), c.end(), comp);
    }
    __STL_UNWIND(c.clear());
  }
  void pop() {
    __STL_TRY {
     	// 间接使用heap算法
      pop_heap(c.begin(), c.end(), comp);
      c.pop_back();
    }
    __STL_UNWIND(c.clear());
  }
};
```



### 实际操作

```c
int main()
{
	int a[4] = { 1, 2, 3, 4 };
	priority_queue<int> pq(a, a+4);
	while(!pq.empty())
	{
		cout << pq.top() << " ";	// 4, 3, 2, 1
		pq.pop();
	}

	exit(0);
}
```



### 总结

`priority_queue`本身实现是很复杂的, 但是当我们已经了解过`vector`, `heap`之后再来看, 他其实很简单了, 就是将vector作为容器, heap作为算法来操作的配置器, 这也体现了STL的灵活性是很高的, 通过各个容器与算法的结合就能做出另一种功能的结构.

