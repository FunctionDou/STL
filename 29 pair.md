# pair

### 前言

前面在分析set, RB-tree都有在insert实现中出现`pair`, 下节分析`map`的时候更会经常出现pair, 所以打算在之前先对pair有个认识. 

`pair`是一个有两个变量的结构体, 即谁都可以直接调用它的变量, 毕竟struct默认权限都是public, 将两个变量用`pair`绑定在一起, 这就为map<T1, T2>提供的存储的基础. 



### pair操作

pair结构是在`map`头文件里面, 直接使用pair实例化可以修改任意元素, 而map对`first`设置为了`const`, 所以不能修改键值.

```c++
int main()
{
	pair<string, int> pa[3];
	pa[0].first = "one", pa[0].second = 1;
	pa[1].first = "two", pa[1].second = 2;
	pa[2].first = "three", pa[2].second = 3;

	for(const auto &i : pa)
		cout << i.first << " " << i.second << endl;	
    // one 1
	// two 2
	// three 3

	pa[0].first = "zero", pa[0].second = 0;
	for(const auto &i : pa)
		cout << i.first << " " << i.second << endl;
    // zero 0
	// two 2
	// three 3

	exit(0);
}
```





### pair分析

pair的实现很简单, 结构体封装了两个变量, 重载了==与<运算符, 也提供了构造函数.



**pair结构定义**

```c++
template <class T1, class T2>	// 两个参数类型
struct pair {
  typedef T1 first_type;
  typedef T2 second_type;

    // 定义的两个变量
  T1 first;	
  T2 second;
    
    // 构造函数
  pair() : first(T1()), second(T2()) {}
  pair(const T1& a, const T2& b) : first(a), second(b) {}
#ifdef __STL_MEMBER_TEMPLATES
  template <class U1, class U2>
  pair(const pair<U1, U2>& p) : first(p.first), second(p.second) {}
#endif
};
```



**重载**

因为只有两个变量, 运算符重载也很简单

```c++
template <class T1, class T2>
inline bool operator==(const pair<T1, T2>& x, const pair<T1, T2>& y) { 
  return x.first == y.first && x.second == y.second; 
}
template <class T1, class T2>
inline bool operator<(const pair<T1, T2>& x, const pair<T1, T2>& y) { 
  return x.first < y.first || (!(y.first < x.first) && x.second < y.second); 
}
```

**make_pair**

根据两个数值构造一个pair.

```c++
template <class T1, class T2>
inline pair<T1, T2> make_pair(const T1& x, const T2& y) {
  return pair<T1, T2>(x, y);
}
```



### 总结

整体pair的功能与实现都是很简单的,  这都是为map的实现做准备的, 下节我们就来分析map的实现.