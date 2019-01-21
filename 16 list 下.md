# list

## 前言

前两节对`list`的push, pop, insert等操作做了分析, 本节准备探讨`list`怎么实现`sort`功能.

`list`是一个循环双向链表, 不是一个连续地址空间, 所以`sort`功能需要特殊的算法单独实现, 而不能用算法中的sort. 当然还可以将`list`的元素插入到vector中最后在将vector排序好的数据拷贝回来, 不过这种做法很费时, 费效率. 



## list操作实现

在分析`sort`之前先来分析`transfer`, `reverse`, `merge`这几个会被调用的函数.



### transfer函数

<font color=#b20>**`transfer`函数功能是将一段链表插入到我们指定的位置之前**</font>. 该函数一定要理解, 后面分析的所有函数都是该基础上进行修改的.

`transfer`函数接受3个迭代器. 第一个迭代器表示链表要插入的位置, `first`到`last`最闭右开区间插入到`position`之前.



从`if`下面开始分析(*这里我将源码的执行的先后顺序进行的部分调整, 下面我分析的都是调整顺序过后的代码. 当然我也会把源码顺序写下来, 以便参考*)

-   **为了避免待会解释起来太绕口, 这里先统一一下部分名字**

1.  `last`的前一个节点叫`last_but_one`
2.  `first`的前一个节点叫`zero`

-   好, 现在我们开始分析`transfer`的每一步(*最好在分析的时候在纸上画出两个链表一步步来画*)

1.  第一行.  `last_but_one`的`next`指向插入的`position`节点
2.  第二行. `position`的`next`指向`last_but_one`
3.  第三行. 临时变量`tmp`保存`position`的前一个节点
4.  第四行. `first`的`prev`指向`tmp`
5.  第五行. `position`的前一个节点的`next`指向`first`节点
6.  第六行. `zero`的`next`指向`last`节点
7.  第七行. `last`的`prev`指向`zero`

```c++
template <class T, class Alloc = alloc>
class list 
{
    ...
protected:
    void transfer(iterator position, iterator first, iterator last) 
    {
      if (position != last) 
      {
          (*(link_type((*last.node).prev))).next = position.node;
          (*position.node).prev = (*last.node).prev;
          link_type tmp = link_type((*position.node).prev);
          (*first.node).prev = tmp;
          (*(link_type((*position.node).prev))).next = first.node; 
          (*(link_type((*first.node).prev))).next = last.node;
          (*last.node).prev = (*first.node).prev; 
      }
    }
    /*
    void transfer(iterator position, iterator first, iterator last) 
    {
      if (position != last) 
      {
        (*(link_type((*last.node).prev))).next = position.node;
        (*(link_type((*first.node).prev))).next = last.node;
        (*(link_type((*position.node).prev))).next = first.node;  
        link_type tmp = link_type((*position.node).prev);
        (*position.node).prev = (*last.node).prev;
        (*last.node).prev = (*first.node).prev; 
        (*first.node).prev = tmp;
      }
    }
    */
    ...
};
```

**splice** 将两个链表进行合并.

```c++
template <class T, class Alloc = alloc>
class list 
{
    ...
public:
    void splice(iterator position, list& x) {
      if (!x.empty()) 
        transfer(position, x.begin(), x.end());
    }
    void splice(iterator position, list&, iterator i) {
      iterator j = i;
      ++j;
      if (position == i || position == j) return;
      transfer(position, i, j);
    }
    void splice(iterator position, list&, iterator first, iterator last) {
      if (first != last) 
        transfer(position, first, last);
    }
    ...
};
```



### merge函数

`merge`函数接受一个`list`参数. 

**`merge`函数是将传入的`list`链表x与原链表按从小到大合并到原链表中(前提是两个链表都是已经从小到大排序了)**. 这里`merge`的核心就是`transfer`函数.

```c++
template <class T, class Alloc>
void list<T, Alloc>::merge(list<T, Alloc>& x) {
  iterator first1 = begin();
  iterator last1 = end();
  iterator first2 = x.begin();
  iterator last2 = x.end();
  while (first1 != last1 && first2 != last2)
    if (*first2 < *first1) {
      iterator next = first2;
      // 将first2到first+1的左闭右开区间插入到first1的前面
      // 这就是将first2合并到first1链表中
      transfer(first1, first2, ++next);
      first2 = next;
    }
    else
      ++first1;
      // 如果链表x还有元素则全部插入到first1链表的尾部
  if (first2 != last2) transfer(last1, first2, last2);
}
```



### reverse函数

**`reverse`函数是实现将链表翻转的功能.** 主要是`list`的迭代器基本不会改变的特点, 将每一个元素一个个插入到`begin`之前. 这里注意迭代器不会变, 但是`begin`会改变, 它始终指向第一个元素的地址.

```c++
template <class T, class Alloc>
void list<T, Alloc>::reverse() 
{
  if (node->next == node || link_type(node->next)->next == node) 
  	return;
  iterator first = begin();
  ++first;
  while (first != end()) {
    iterator old = first;
    ++first;
      // 将元素插入到begin()之前
    transfer(begin(), old, first);
  }
} 
```



### sort

`list`实现`sort` 功能本身就不容易, 当我分析了之后就对其表示佩服. 严格的说`list`排序的时间复杂度应为`nlog(n)`, 其实现用了归并排序的思想, 将所有元素分成n分, 总共2^n个元素.

这个sort的分析 : 

-   这里将每个重要的参数列出来解释其含义

    1.  `fill` : 当前可以处理的元素个数为2^fill个

    2.  `counter[fill]` : 可以容纳2^(fill+1)个元素
    3.  `carry` : 一个临时中转站, 每次将一元素插入到counter[i]链表中.


在处理的元素个数不足2^fill个时，在`counter[i](0<i<fill)`之前转移元素

具体是显示步骤是：

1.  每次读一个数据到`carry`中，并将carry的数据转移到`counter[0]`中
    1.  当`counter[0]`中的数据个数少于2时，持续转移数据到counter[0]中
    2.  当counter[0]的数据个数等于2时，将counter[0]中的数据转移到counter[1]...从counter[i]转移到counter[i+1],直到counter[fill]中数据个数达到2^(fill+1)个。
2.  ++fill, 重复步骤1

```c++
//list 不能使用sort函数，因为list的迭代器是bidirectional_iterator, 而sort
//sort函数要求random_access_iterator
template<class T,class Alloc>
void list<T,Alloc>::sort()
{
    //如果元素个数小于等于1，直接返回
    if(node->next==node||node->next->next==node)
    return ;
    list<T,Alloc> carry; //中转站
    list<T,Alloc> counter[64];
    int fill=0;
    while(!empty())
    {
        carry.splice(carry.begin(),*this,begin());  //每次取出一个元素
        int i=0;    
        while(i<fill&&!counter[i].empty())
        {
            counter[i].merge(carry);  //将carry中的元素合并到counter[i]中
            carry.swap(counter[i++]);  //交换之后counter[i-1]为空
        }
        carry.swap(counter[i]);
        if(i==fill) 
            ++fill;
    }
    // 将counter数组链表的所有节点按从小到大的顺序排列存储在counter[fill-1]的链表中
    for(int i=1;i<fill;++i)
    {
        counter[i].merge(counter[i-1]);
    }
    // 最后将couter与carry交换, 实现排序
    swap(counter[fill-1]);
}
```

`sort`用了一个数组链表用来存储2^i个元素, 当上一个元素存储满了之后继续往下一个链表存储, 最后将所有的链表进行`merge`归并(合并), 从而实现了链表的排序.



## 总结

本节我们分析了`list`最难的`transfer`和`sort`实现, 当然`transfer`函数是整个实现的核心. 我在将本节分析的函数在进行一个归纳.

1.  `transfer` : 将两个*段*链表进行合并(两段可以是来自同一个链表, 但不交叉).
2.  `merge` : 前提两个段链表都已经排好序. 将两段链表按从小到大的顺序进行合并, 主要是`sort`实现调用.
3.  `reverse` : 调用`transfer`函数将元素一个个调整到`begin`之前, 实现链表的转置.
4.  `sort` : 运用归并思想将链表分段排序.