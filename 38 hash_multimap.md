# hash_multimap

### 前言

上节分析了`hash_map`知道它是不允许存在相同的键存在, 本节的`hash_multimap`就可以允许存在多个相同的键, 以`hashtable`为接口, 这里我就只分析与hash_map的不同点吧, 毕竟很多都是一样的了, 没有必要重复的分析相同的代码.



### insert

最大的区别就在这里, hash_multimap 是以`insert_equal`为接口, 所以支持插入重复的键.

```c++
#ifndef __STL_LIMITED_DEFAULT_TEMPLATES
template <class Key, class T, class HashFcn = hash<Key>,
          class EqualKey = equal_to<Key>,
          class Alloc = alloc>
#else
template <class Key, class T, class HashFcn, class EqualKey,
          class Alloc = alloc>
#endif
class hash_multimap
{
public:
	...
  iterator insert(const value_type& obj) { return rep.insert_equal(obj); }
#ifdef __STL_MEMBER_TEMPLATES
  template <class InputIterator>
  void insert(InputIterator f, InputIterator l) { rep.insert_equal(f,l); }
#else
  void insert(const value_type* f, const value_type* l) {
    rep.insert_equal(f,l);
  }
  void insert(const_iterator f, const_iterator l) { rep.insert_equal(f, l); }
#endif /*__STL_MEMBER_TEMPLATES */
  iterator insert_noresize(const value_type& obj)
    { return rep.insert_equal_noresize(obj); }    
    ...
};
```



### 总结

本节很快的分析了`hash_multimap` ,  主要实现区别在`insert`接口不同, 其他的功能都与hash_map一样. 从下一节我们就开始分析算法, 当然我只是选择了几个比较有意思的进行分析, 毕竟很多都直接就能理解.