# vector

## 前言

前面两节我们分析了关于`vector`的push, pop, erase, 重载和vector的定义, 通过前面的分析都对vector的实现有所了解了, 这一节我们就来分析vector是怎么实现插入操作. 当数组的大小不足后`insert`又是怎么来处理的.



## 插入实现

`insert`为了接受不同的参数和参数个数, 所以定义了多个重载函数. 现在我们就一个个来进行分析.



**insert(iterator position, const T& x) ** 传入一个迭代器和插入的值.

1.  如果数组还有备用空间, 并且插入的是finish位置, 直接插入即可, 最后调整finish就行了.
2.  以上条件不成立, 调用`insert_aux`函数执行插入操作

```c++
iterator insert(iterator position) { return insert(position, T()); }
iterator insert(iterator position, const T& x) ;

iterator insert(iterator position, const T& x) 
{
      size_type n = position - begin();
    // 如果数组还有备用空间, 并且插入的是finish位置, 直接插入即可, 最后调整finish就行了.
      if (finish != end_of_storage && position == end()) 
      {
        construct(finish, x);
        ++finish;
      }
    // 以上条件不成立, 调用另一个函数执行插入操作
      else
        insert_aux(position, x);
      return begin() + n;
}
```

1.  **如果数组还有备用空间, 就直接移动元素, 再将元素插入过去, 最后调整finish就行了.**
2.  **没有备用空间, <font color=#b20>重新申请空间原始空间的两倍+1的空间</font>后, 再将元素拷贝过去同时执行插入操作**
3.  析构调用原始空间元素以及释放空间, 最后修改3个迭代器的指向.

```c++
void insert_aux(iterator position, const T& x);

template <class T, class Alloc>
void vector<T, Alloc>::insert_aux(iterator position, const T& x) 
{
     // 如果数组还有备用空间, 就直接移动元素, 再将元素插入过去, 最后调整finish就行了.
  if (finish != end_of_storage) 
  {
      // 调用构造, 并将最后一个元素复制过去, 调整finish
    construct(finish, *(finish - 1));
    ++finish;
    T x_copy = x;
      // 将插入元素位置的后面所有元素往后移动, 最后元素插入到位置上.
    copy_backward(position, finish - 2, finish - 1);
    *position = x_copy;
  }
   // 没有备用空间, 重新申请空间再将元素拷贝过去同时执行插入操作
  else {
    const size_type old_size = size();
    const size_type len = old_size != 0 ? 2 * old_size : 1;	// 重新申请空间原始空间的两倍+1的空间

    iterator new_start = data_allocator::allocate(len);
    iterator new_finish = new_start;
    __STL_TRY {
        // 进行分段将原始元素拷贝新的空间中, 这样也就实现了插入操作
      new_finish = uninitialized_copy(start, position, new_start);
      construct(new_finish, x);
      ++new_finish;
      new_finish = uninitialized_copy(position, finish, new_finish);
    }

#       ifdef  __STL_USE_EXCEPTIONS 
    catch(...) {
      destroy(new_start, new_finish); 
      data_allocator::deallocate(new_start, len);
      throw;
    }
#       endif /* __STL_USE_EXCEPTIONS */
      // 释放掉原来的空间, 调整新的3个迭代器的位置
    destroy(begin(), end());
    deallocate();
    start = new_start;
    finish = new_finish;
    end_of_storage = new_start + len;
  }
}
```



**void insert (iterator pos, size_type n, const T& x)** 传入一个迭代器, 插入的个数, 和插入的值. 共3个参数.

其重载函数都调用同一个函数实现插入操作.

```c++
void insert (iterator pos, size_type n, const T& x);
void insert (iterator pos, int n, const T& x) 
{
      insert(pos, (size_type) n, x);
}
void insert (iterator pos, long n, const T& x) 
{
      insert(pos, (size_type) n, x);
}        
```

`insert(iterator position, size_type n, const T& x)`实现步骤:

1.   如果备用空间足够大, 先保存插入位置到end的距离

     1.   从插入的位置到end的距离大于要插入的个数n
          1.   先构造出finish-n个大小的空间, 再移动finish - n个元素的数据
          2.   在将从插入位置后的n个元素移动
          3.   最后元素从插入位置开始进行填充
     2.   从插入的位置到end的距离小于了要插入的个数
          1.   先构造出n - elems_after个大小的空间, 再从finish位置初始化n - elems_after为x
          2.   从插入位置开始到原来的finish位置结束全部复制到新的结束位置后面
          3.   从插入位置进行填充x

     -   上面分成两种情况就只是为了插入后的元素移动方便. 其实我自己考虑过为什么直接分配出n个空间, 然后从position+n处开始把原数据依次拷贝再将数据插入过去就行了, 没有必要分成两种情况, 可能这里面有什么我没有考虑到的因素吧.

2.   备用空间不足的时候执行

     1.   重新申请一个当前两倍的空间或者当前大小+插入的空间, 选择两者最大的方案.
     2.   进行分段复制到新的数组中, 从而实现插入
     3.   将当前数组的元素进行析构, 最后释放空间
     4.   修改3个迭代器

```c++
template <class T, class Alloc>
void vector<T, Alloc>::insert(iterator position, size_type n, const T& x) 
{
	if (n != 0) 
  	{
        // ******** 1 ***********
        // 备用空间足够大
    	if (size_type(end_of_storage - finish) >= n) 
    	{
      		T x_copy = x;
            // 保存插入位置到end的距离
      		const size_type elems_after = finish - position;
      		iterator old_finish = finish;
            // ******* a **********
            // 从插入的位置到数据结束的距离大于了要插入的个数n
      		if (elems_after > n) 
      		{
                // 先构造出finish-n个大小的空间, 再移动finish - n个元素的数据
        		uninitialized_copy(finish - n, finish, finish);
        		finish += n;
                // 在将从插入位置后的n个元素移动
        		copy_backward(position, old_finish - n, old_finish);
                // 元素从插入位置开始进行填充即可
        		fill(position, position + n, x_copy);
      		}
            // ********* b *********
            // 从插入的位置到end的距离小于了要插入的个数
      		else 
            {
                // 先构造出n - elems_after个大小的空间, 再从finish位置初始化n - elems_after为x
        		uninitialized_fill_n(finish, n - elems_after, x_copy);
        		finish += n - elems_after;
                // 从插入位置开始到原来的finish位置结束全部复制到新的结束位置后面
        		uninitialized_copy(position, old_finish, finish);
        		finish += elems_after;
                // 从插入位置进行填充x
        		fill(position, old_finish, x_copy);
      		}
    	}
        // ******* 2 ***********
        // 空间不足处理
    	else 
    	{
            // 重新申请一个当前两倍的空间或者当前大小+插入的空间, 选择两者最大的方案.
	      	const size_type old_size = size();        
	      	const size_type len = old_size + max(old_size, n);
	      	iterator new_start = data_allocator::allocate(len);
	      	iterator new_finish = new_start;
	      	__STL_TRY 
	        {
                // 同样进行分段复制到新的数组中, 从而实现插入
		        new_finish = uninitialized_copy(start, position, new_start);
	        	new_finish = uninitialized_fill_n(new_finish, n, x);
	        	new_finish = uninitialized_copy(position, finish, new_finish);
    	  	}
#         ifdef  __STL_USE_EXCEPTIONS 
      		catch(...) 
      		{
	        	destroy(new_start, new_finish);
    	    	data_allocator::deallocate(new_start, len);
        		throw;
      		}
#         endif /* __STL_USE_EXCEPTIONS */
            // 将当前数组的元素进行析构, 最后释放空间
      		destroy(start, finish);
      		deallocate();
            // 修改3个迭代器
      		start = new_start;
      		finish = new_finish;
      		end_of_storage = new_start + len;
    	}
	}
}
```



**insert(iterator position, const_iterator first, const_iterator last)** 传入3个迭代器, 进行范围插入.

整个流程与上面的操作基本一样, 所以这里也不详细分析了.

```c++
template <class T, class Alloc>
void vector<T, Alloc>::insert(iterator position, const_iterator first, const_iterator last) {
// 插入不为空
  if (first != last) {
    size_type n = 0;
    // 计算插入的长度, 并保存在n中
    distance(first, last, n);
    // 如果是剩余的空间大于插入的长度.
    if (size_type(end_of_storage - finish) >= n) {
    	// 保存插入点到end的距离
      const size_type elems_after = finish - position;
      iterator old_finish = finish;
      	// 同样比较n与插入点到end的距离, 下面的过程与上面描述的基本一致.
      	// 从插入的位置到end的距离大于要插入的个数n
      if (elems_after > n) {
        uninitialized_copy(finish - n, finish, finish);
        finish += n;
        copy_backward(position, old_finish - n, old_finish);
        copy(first, last, position);
      }
      else {
        uninitialized_copy(first + elems_after, last, finish);
        finish += n - elems_after;
        uninitialized_copy(position, old_finish, finish);
        finish += elems_after;
        copy(first, first + elems_after, position);
      }
    }
    else {
      const size_type old_size = size();
      const size_type len = old_size + max(old_size, n);
      iterator new_start = data_allocator::allocate(len);
      iterator new_finish = new_start;
      __STL_TRY {
        new_finish = uninitialized_copy(start, position, new_start);
        new_finish = uninitialized_copy(first, last, new_finish);
        new_finish = uninitialized_copy(position, finish, new_finish);
      }
#         ifdef __STL_USE_EXCEPTIONS
      catch(...) {
        destroy(new_start, new_finish);
        data_allocator::deallocate(new_start, len);
        throw;
      }
#         endif /* __STL_USE_EXCEPTIONS */
      destroy(start, finish);
      deallocate();
      start = new_start;
      finish = new_finish;
      end_of_storage = new_start + len;
    }
  }
}
```



## vector实例

我将vector的操作放在最后, 想这insert的实现有多种, 不一定都用过, 将insert分析完后再来实现印象. 

这里我主要为了验证`vector`在插入时备用空间不足的情况下的例子, 同时用上了3种插入方法.

```c++
class T
{
	public:
		T() { cout << "construct" << endl;}
		~T() { cout << "destruct" << endl; }
};

int main()
{
	vector<T> v1;
	cout << "array size = " << v1.capacity() << endl;
	cout << endl;

	v1.insert(v1.begin(), T());
	cout << "array size = " << v1.capacity() << endl;
	cout << endl;

	v1.insert(v1.begin(), v1.begin(), v1.end());
	cout << "array size = " << v1.capacity() << endl;
	cout << endl;

	v1.insert(v1.begin(), v1.size(), T());
	cout << "array size = " << v1.capacity() << endl;
	cout << endl;

	v1.insert(v1.begin(), v1.size(), T());
	cout << "array size = " << v1.capacity() << endl;
	cout << endl;

	exit(0);
}
```

输出结果:

```c++
array size = 0

construct
destruct
array size = 1

destruct
array size = 2

construct
destruct
destruct
destruct
array size = 4

construct
destruct
destruct
destruct
destruct
destruct
array size = 8
```

明显可以看出每次插入是空间不足都会重新寻找新的空间然后析构释放掉原始空间. **所以`vector`插入操作并不是非常的高效, 效率很低, 代价高.**



## 总结

本节分析了`insert`和删除的操作, 可能都清楚了为什么建议一个经常调用随机插入和删除操作就尽量不要用`vector`, 因为开销有时真的很大, 如果经常插入不如用`list`, 因为`list`对随机插入和删除的开销比`vector`小的多, `list`的用法我们下节再来分析, 

`vector`适合用来做随机访问很快, 也支持适合频率较低的插入和删除操作, 在每次插入和删除后迭代器都会失效, 因为不能保证操作之后迭代器的位置没有改变.