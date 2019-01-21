# queue

### 前言

上一节分析了`stack`实现, `stack`是修改了`deque`的接口而实现的一个功能简单的结构, 本节分析的`queue`也是用`deque`为底层容器封装. 

`queue`数据都是在头部进行操作的, 之允许进行push和pop操作.



### queue分析

**queue结构**

```c++
#ifndef __STL_LIMITED_DEFAULT_TEMPLATES
template <class T, class Sequence = deque<T> >
#else
template <class T, class Sequence>
#endif
class queue {
// 定义友元函数
  friend bool operator== __STL_NULL_TMPL_ARGS (const queue& x, const queue& y);
  friend bool operator< __STL_NULL_TMPL_ARGS (const queue& x, const queue& y);
public:
  typedef typename Sequence::value_type value_type;
  typedef typename Sequence::size_type size_type;
  typedef typename Sequence::reference reference;
  typedef typename Sequence::const_reference const_reference;
protected:
  Sequence c;
public:
  bool empty() const { return c.empty(); }
  size_type size() const { return c.size(); }
  // 调用deque的front函数
  reference front() { return c.front(); }
  const_reference front() const { return c.front(); }
  reference back() { return c.back(); }
  const_reference back() const { return c.back(); }
    // 只封装push_back, pop_front函数
  void push(const value_type& x) { c.push_back(x); }
  void pop() { c.pop_front(); }
};
```



**友元函数**

```c++
// 实现重载
template <class T, class Sequence>
bool operator==(const queue<T, Sequence>& x, const queue<T, Sequence>& y) {
  return x.c == y.c;
}

template <class T, class Sequence>
bool operator<(const queue<T, Sequence>& x, const queue<T, Sequence>& y) {
  return x.c < y.c;
}
```



### queue实际操作

```c++
int main()
{
	std::queue<int> qu;
	qu.push(1);
	qu.push(2);
	qu.push(3);
	qu.size();
	while(!qu.empty())
	{
		std::cout << qu.front()  << " ";	// 1 2 3 
		qu.pop();
	}
	exit(0);
}
```



### 总结

`queue`与`stack`都是使用底层接口封装的结构, 他们是被称为配接器而不是容器.