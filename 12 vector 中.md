# vector

## 前言

上一节我们分析了`vector`的构造, 析构, back, front等基本操作, 这一节我们就来分析`vector`实现插入, 删除等直接对数组具体操作的实现.



## vector实例

与上节一样, 我们将待会会用到的部分常用的操作先执行一次, 进行一次快速的回忆.

```c++
int main()
{
	vector<int> v1;
	vector<int> v2(4);
	vector<int> v3(4, 1);

	v1.push_back(1);
	v1.push_back(2);
	v1.push_back(3);
	v1.push_back(4);
	v1.push_back(5);
	
    // 这里清除的是一个[v1.begin()+2, v1.end()-1) , 左闭右开的区间
	v1.erase(v1.begin()+2, v1.end()-1);
	v1.pop_back();
	
	v2 = v1;
	if(!v2.empty())	// 不为空
	{
		for (const auto i : v2)
		{
			cout << i << " ";
		}
		cout << endl;
		cout << "size = " << v2.size() << endl;
		v1.~vector();
	}

	exit(0);
}
```



## vector实现

### push和pop操作

**`vector`的push和pop操作都只是对尾进行操作.** 这里说的尾部是指数据的尾部, 尾部时数组大小的尾部.

`push_back`从尾部插入数据.  当数组还有备用空间的时候就直接插入尾部就行了, 当没有备用空间后就重新寻找更大的空间再将数据全部复制过去.

```c++
// 如果可用空间还有就调用对象的构造函数并使用空间的尾增加
// 没有空间就重新申请一个更大的空间, 然后进行插入
void push_back(const T& x) 
{
      // 如果还没有到填满整个数组, 就在数据尾部插入
      if (finish != end_of_storage) 
      {
        	construct(finish, x);
        	++finish;
      }
    // 数组被填充满, 调用insert_aux必须重新寻找新的更大的连续空间, 再进行插入
      else
          insert_aux(end(), x);
}
```

`pop_back`从尾部进行删除

```c++
// 使用空间的尾自减并调用其析构函数. 但是并没有释放内存
void pop_back() 
{
     --finish;
     destroy(finish);
}
```

push和pop也保证 了`finish`始终都指向最后一个元素的后一个位置的地址.



### 删除元素

#### erase删除一个元素

`erase`清除指定位置的元素, 其重载函数用于清除一个范围内的所有元素. 实际实现就是将删除元素后面所有元素往前移动, 对于vector来说这样的操作花费的时间还是很大的, 毕竟他是一个数组.

```c++
// 清除指定位置的元素. 实际就是将指定位置后面的所有元素向前移动, 最后析构掉最后一个元素
iterator erase(iterator position) 
{
    if (position + 1 != end())
    	copy(position + 1, finish, position);
    --finish;
    destroy(finish);
    return position;
}
 
// 清除一个指定范围的元素, 同样将指定范围后面的所有元素向前移动, 最后析构掉整个范围的元素
// 清除的是左闭右开的区间 [ )
iterator erase(iterator first, iterator last) 
{
    iterator i = copy(last, finish, first);
    destroy(i, finish);
    finish = finish - (last - first);
    return first;
}
```

#### clear

`clear`清除一个范围的数据.

```c++
void clear() { erase(begin(), end()); }
```



### 重载运算符

`vector`为实现给用户最直接最方便的用途, 重载了`[]`, `=`等运算符, 用户更加容易的能操作迭代器, 使用起来更像是直接操作数组一样. 



#### 重载

`[]`返回的是元素的**引用, 即一个左值**, 毕竟可能会对元素值进行修改. 

```c++
reference operator[](size_type n) { return *(begin() + n); }
const_reference operator[](size_type n) const { return *(begin() + n); }
```

针对右值和const类型选择不同的重载方法.

vector的`[]`重载很有意思, 是`begin() + n`实现, 也就是说**n可以为负数**. 

```c++
// 验证为n可以为负数
int main()
{
	vector<int> a(10);
	iota(a.begin(), a.end(), 1);
	for_each(a.begin(), a.end(), [](int a){ cout << a <<  " ";}); // 1 2 3 4 5 6 7 8 9 10 
	auto i = a.begin() + 3;
	cout << i[-1] << " " << a.end()[-1];	// 3 10

	exit(0);
}
```

*end()[-1]可以这样的操作来获取最后一个元素, 当然这样的操作一般也不会这样干*.



`vector`之间能相互的复制主要它也重载了`=`, 使相互传递更加的便利.

```c++
vector<T, Alloc>& operator=(const vector<T, Alloc>& x);
template <class T, class Alloc>
vector<T, Alloc>& vector<T, Alloc>::operator=(const vector<T, Alloc>& x) 
{
  	if (&x != this) 
  	{
        // 判断x的数据大小跟赋值的数组大小
		if (x.size() > capacity()) 	// 数组大小过小
    	{
            // 进行范围的复制, 并销毁掉原始的数据.
	      	iterator tmp = allocate_and_copy(x.end() - x.begin(), x.begin(), x.end());
      		destroy(start, finish);
      		deallocate();
            // 修改偏移
      		start = tmp;
      		end_of_storage = start + (x.end() - x.begin());
    	}
        // 数组的元素大小够大, 直接将赋值的数据内容拷贝到新数组中. 并将后面的元素析构掉
    	else if (size() >= x.size()) 
        {
      		iterator i = copy(x.begin(), x.end(), begin());
      		destroy(i, finish);
    	}
        // 数组的元素大小不够, 装不完x的数据, 但是数组本身的大小够大
    	else 
    	{
            // 先将x的元素填满原数据大小
	      	copy(x.begin(), x.begin() + size(), start);
            // 再将x后面的数据全部填充到后面
	      	uninitialized_copy(x.begin() + size(), x.end(), finish);
	    }
	    finish = start + x.size();
  	}
  	return *this;
}
```

我不清楚为什么最后一个实现需要分两步走, 既然原数组的大小够大, 元素小, 所以可以直接覆盖掉原数据就行了, 不清相分两步. 



### 容器的调整

`reserver`修改容器实际的大小

```c++
void reserve(size_type n) 
{
    // 修改的容器大小要大于原始数组大小才行
      if (capacity() < n) 
      {
        const size_type old_size = size();
          // 重新拷贝数据, 并将原来的空间释放掉
        iterator tmp = allocate_and_copy(n, start, finish);
        destroy(start, finish);
        deallocate();
          // 重新修改3个迭代器位置
        start = tmp;
        finish = tmp + old_size;
        end_of_storage = start + n;
      }
}
```

`resize`重新修改数组元素的容量. 这里是修改容纳元素的大小, 不是数组的大小.

```c++
void resize(size_type new_size) { resize(new_size, T()); }
void resize(size_type new_size, const T& x) 
{
    // 元素大小大于了要修改的大小, 则释放掉超过的元素
      if (new_size < size()) 
        erase(begin() + new_size, end());
    // 元素不够, 就从end开始到要求的大小为止都初始化x
      else
        insert(end(), new_size - size(), x);
}
```



### swap交换

vector实现`swap`就只是将3个迭代器进行交换即可, 并不用将整个数组进行交换.

```c++
void swap(vector<T, Alloc>& x) 
{
      __STD::swap(start, x.start);
      __STD::swap(finish, x.finish);
      __STD::swap(end_of_storage, x.end_of_storage);
}
```



## 总结

本节将`vector`的删除, 交换, 重载等操作进行的分析. 学到关于交换数组可以修改头尾指针即可, 并不实际交换整个元素. 同时要注意`erase`清除是一个**左闭右开的区间**.  因为insert的代码很多, 所以我将插入操作放到下节进行分析. 
