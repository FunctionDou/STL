# template之类相关

## 前言

前面两节主要分析了使用`template`的注意点以及非类型参数如何使用, 本节来探讨虚模板函数和模板拷贝构造函数.



## 虚函数模板

在我们使用模板从来都没有将虚函数与模板进行套用, 那么这两者能不能同时连用呢?

这个直接来写代码验证才知道.

```c++
class point
{
	public:
		template<class T>
			virtual T getX()
			{
				return x;
			}
	private:
		int x;
};

int main()
{
	exit(0);
}
```

分别用了`g++`和`clang`编译

```c++
// g++
virtual_template.cpp:17:4: error: templates may not be ‘virtual’
    virtual T getX()
// clang
virtual_template.cpp:17:4: error: 'virtual' cannot be specified on member function templates
                        virtual T getX()
```

可以看出来`clang`更加准确的指出了<font color=#b20>`virtual`不能是member function templates. </font> 

*为什么虚函数不能是模板函数?*

-   <font color=#b20>编译器在编译类的定义的时候就必须确定该类的虚表大小.</font> 
-   模板只有在运行调用时才能确认其大小, 两者冲突. 结果显而易见.



## 模板拷贝构造函数

模板与不同模板之间不能直接的赋值(强制转换), 毕竟模板一般都是类和函数都不是普通的类型. 但是类有拷贝构造函数, 所以我们可以对类的构造函数进行修改, 也就成了模板构造函数.

模板拷贝构造函数与一般的拷贝构造函数没有什么区别, 仅仅实在加了一个模板作为返回值类型

```c++
template<class T>
class tmp
{
	public:
		tmp() : x(0) {}
		template<class U>
			tmp(const tmp<U>& t)
			{
				x = t.x;
			}
	private:
		T x;
};

int main()
{
	tmp<int> t1;
	tmp<char> t2;
	t1 = t2;

	exit(0);
}
```



## 总结

本节分析了类的虚表大小要在编译时知道其大小所以虚函数不能为模板函数, 模板构造函数与普通的构造函数写法无异, 并且可以实现强制转换的效果.

