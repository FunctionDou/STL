# stack

### 前言

上面几节我们分析了`deque`是一个双向开口, 并且每部分数据也是存放在连续空间中, 而`stack`是栈, 只允许在尾进行操作, 而且`stack`的功能`deque`都已经实现了, 只需要对`deque`的功能进行部分封装就行了, 也就提取出pop_back, push_back就行了. 

`stack`严格他并不是容器, 它是一底部容器完成其所有的工作, 它只修改了容器的接口, 准确是叫**配接器**.



### stack源码

`stack`也满足`straits`编程规范.

```c++
#ifndef __STL_LIMITED_DEFAULT_TEMPLATES
template <class T, class Sequence = deque<T> >
#else
template <class T, class Sequence>
#endif
class stack {
// 定义友元函数
  friend bool operator== __STL_NULL_TMPL_ARGS (const stack&, const stack&);
  friend bool operator< __STL_NULL_TMPL_ARGS (const stack&, const stack&);
public:
  typedef typename Sequence::value_type value_type;
  typedef typename Sequence::size_type size_type;
  typedef typename Sequence::reference reference;
  typedef typename Sequence::const_reference const_reference;
protected:
  Sequence c;
public:
  bool empty() const { return c.empty(); }	// 调用deque的empty函数
  size_type size() const { return c.size(); }	
  // 调用deque的back函数
  reference top() { return c.back(); }
  const_reference top() const { return c.back(); }
  // 只封装了push_back和pop_back函数, 只对尾进行操作
  void push(const value_type& x) { c.push_back(x); }
  void pop() { c.pop_back(); }
};

// 这里调用的deque的==与!=重载
template <class T, class Sequence>
bool operator==(const stack<T, Sequence>& x, const stack<T, Sequence>& y) {
  return x.c == y.c;
}
template <class T, class Sequence>
bool operator<(const stack<T, Sequence>& x, const stack<T, Sequence>& y) {
  return x.c < y.c;
}
```



### 总结

这里stack严格来说是配接器, 就是修改其他容器的接口实现简单的功能. 同样下节分析的`queue`也是以`deque`为接口的配接器.

