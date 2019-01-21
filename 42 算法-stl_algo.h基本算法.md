# stl_algo.h中的基本算法

### 前言

上一节分析了`stl_algo.h`中的rotate函数实现, 本节我们继续分析该文件的中的其他基本算法, 这些算法功能实现看似很简单, 但是这些算法都能进行衍生, 用户自定义, 简单化了我们的部分编程, 直接使用无需再定义.



### 基本算法



#### for_each

将[first, last) 范围的元素由传入仿函数(函数)进行处理, 最后返回仿函数(函数).

```c++
template <class InputIterator, class Function>
Function for_each(InputIterator first, InputIterator last, Function f) {
  for ( ; first != last; ++first)
    f(*first);	// 传入的是右值
  return f;
}
```

实例:

```c++
void For_each(int i)  { i = 1; }

void For_func(int i) { cout << i << " "; }

int main()
{
	vector<int> a(10);
	for_each(a.begin(), a.end(), For_each);	// 尝试初始化值
	for(const auto &i : a)
		cout << i <<  " ";	// 0 0 0 0 0 0 0 0 0 0
	for_each(a.begin(), a.end(), For_func);	// 0 0 0 0 0 0 0 0 0 0

	return 0;
}
```

很明显尝试初始化值是有问题的, 因为`for_each`不会修改传入的值, 传入的是右值, 所以一般使用`for_each`都是实现迭代器的输出, 这样就不用写for循环再来输出了.



#### find

两个版本.

```c++
// 版本一
template <class InputIterator, class T>
InputIterator find(InputIterator first, InputIterator last, const T& value) {
  while (first != last && *first != value) ++first;
  return first;
}
// 版本二
template <class InputIterator, class Predicate>
InputIterator find_if(InputIterator first, InputIterator last,
                      Predicate pred) {	
  while (first != last && !pred(*first)) ++first;	// 一元操作, 自定义判断条件
  return first;
}
```



#### adjacent_find

两个版本.

第一个版本找出相邻元素间第一个出现元素相同的情况, 返回第一次出现的迭代器

第二个版本找出相邻元素间第一个满足仿函数条件的情况, 返回第一次满足情况的迭代器

```c++
// 第一个版本找出相邻元素间第一个出现元素相同的情况, 返回第一次出现的迭代器
template <class ForwardIterator>
ForwardIterator adjacent_find(ForwardIterator first, ForwardIterator last) {
  if (first == last) return last;
  ForwardIterator next = first;
  while(++next != last) {
    if (*first == *next) return first;
    first = next;
  }
  return last;
}
// 第二个版本找出相邻元素间第一个满足仿函数条件的情况, 返回第一次满足情况的迭代器
template <class ForwardIterator, class BinaryPredicate>
ForwardIterator adjacent_find(ForwardIterator first, ForwardIterator last,
                              BinaryPredicate binary_pred) {
  if (first == last) return last;
  ForwardIterator next = first;
  while(++next != last) {
    if (binary_pred(*first, *next)) return first;
    first = next;
  }
  return last;
}
```



#### count

两个版本

版本一 : 统计[first, last)范围出现的指定元素的次数

版本二 : 统计[first, last)范围满足条件的元素的次数

```c++
// 版本一 : 统计[first, last)范围出现的指定元素的次数
template <class InputIterator, class T, class Size>
void count(InputIterator first, InputIterator last, const T& value, Size& n) {
  for ( ; first != last; ++first)
    if (*first == value)
      ++n;
}
// 版本二 : 统计[first, last)范围满足条件的元素的次数
template <class InputIterator, class Predicate, class Size>
void count_if(InputIterator first, InputIterator last, Predicate pred, Size& n) {
  for ( ; first != last; ++first)
    if (pred(*first))
      ++n;
}

#ifdef __STL_CLASS_PARTIAL_SPECIALIZATION
// 版本一 : 统计[first, last)范围出现的指定元素的次数
template <class InputIterator, class T>
typename iterator_traits<InputIterator>::difference_type
count(InputIterator first, InputIterator last, const T& value) {
  typename iterator_traits<InputIterator>::difference_type n = 0;
  for ( ; first != last; ++first)
    if (*first == value)
      ++n;
  return n;
}
// 版本二 : 统计[first, last)范围满足条件的元素的次数
template <class InputIterator, class Predicate>
typename iterator_traits<InputIterator>::difference_type
count_if(InputIterator first, InputIterator last, Predicate pred) {
  typename iterator_traits<InputIterator>::difference_type n = 0;
  for ( ; first != last; ++first)
    if (pred(*first))
      ++n;
  return n;
}
```



#### replace

两个版本. 

版本一 : 将所有是old_value的数据换成new_value

版本二 : 将所有是满足仿函数(函数)的数据换成new_value

```c++
// 版本一 : 将所有是old_value的数据换成new_value
template <class ForwardIterator, class T>
void replace(ForwardIterator first, ForwardIterator last, const T& old_value,
             const T& new_value) {
  for ( ; first != last; ++first)
    if (*first == old_value) *first = new_value;
}
// 版本二 : 将所有是满足仿函数的数据换成new_value
template <class ForwardIterator, class Predicate, class T>
void replace_if(ForwardIterator first, ForwardIterator last, Predicate pred,
                const T& new_value) {
  for ( ; first != last; ++first)
    if (pred(*first)) *first = new_value;
}
```



#### replace_copy

两个版本. 两个版本都不会修改原容器的数据

版本一 : 将所有元素保存在另个容器中, 并将所有是old_value的数据换成new_value

版本二 : 将所有元素保存在另个容器中, 并将所有满足仿函数的数据换成new_value

```c++
// 将所有元素保存在另个容器中, 并将所有是old_value的数据换成new_value
template <class InputIterator, class OutputIterator, class T>
OutputIterator replace_copy(InputIterator first, InputIterator last,
                            OutputIterator result, const T& old_value,
                            const T& new_value) {
  for ( ; first != last; ++first, ++result)
    *result = *first == old_value ? new_value : *first;
  return result;
}
// 将所有元素保存在另个容器中, 并将所有满足仿函数的数据换成new_value
template <class Iterator, class OutputIterator, class Predicate, class T>
OutputIterator replace_copy_if(Iterator first, Iterator last,
                               OutputIterator result, Predicate pred,
                               const T& new_value) {
  for ( ; first != last; ++first, ++result)
    *result = pred(*first) ? new_value : *first;
  return result;
}
```



### 总结

本节分析了`stl_algo.h`的基本算法的实现, 可以简化平时程序的代码量.

