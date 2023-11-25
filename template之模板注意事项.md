# template之模板注意事项

## 前言

在分析`STL`之前, 我们需要先对`template`做一个回忆, 可能我总结的内容你都会了, 也可能你没有了印象了, 但是我还是希望你先浏览一下template的用法. 毕竟STL全部都涉及到了模板, 而template是学习STL的基础. 我将template暂时分成5节来讲, 这里就先分析三节, 后面.两节准备放在STL中来分析.



## template使用

`template`的使用大大提高了代码的复用性, 抽象性.

1.  **类模板实例化时并不是每个成员函数都实例化了, 而是使用到了哪个成员函数, 那个成员函数才实例化**. 

```c++
/* ***** 1 *******/
template<class T>
class point
{
	public:
		point() : x(0), y(0) {}
		point(T x, T y) : x(x), y(y) {}
		T getX() const   { x = y;  return x; }	// 一般是无法通过编译的, x不允许被修改, 但是这里并没有报错
	private:
		T x;
		T y;
};
/* ***** 2 *******/
#define T int
class point
{
	public:
		point() : x(0), y(0) {}
		point(T x, T y) : x(x), y(y) {}
		T getX() const   { x = y;  return x; }
	private:
		T x; T y;
};
```

成员函数getX()应该无法通过编译, 就像实例2一样, 但是因为模板中没有使用到函数getX(), 也就没有实例化getX, 所以就没有出现编译报错. 实例2必须在编译的时候就要检查所有的成员即函数.

**不是所有的模板都只能在运行时才会被实例化, 比如非类型模板参数就能在编译时期就已经实例化了, 这里主要说的是类模板, 不要混淆了.** 

2.  **可以把类模板和函数模板结合起来, 定义一个含有成员函数模板的类模板**.

```c++
template<class T>
class point
{
	public:
		point() : x(0), y(0) {}
		point(T x, T y) : x(x), y(y) {}
		template<class U>	// 定义了另一个的模板参数类型
			void print(U x);
	private:
		T x;
		T y;
};

// 这里两个都要写出来
template<class T>
template<class U>
void point<T>::print(U x)
{
	std::cout << this->x + x;
}

int main()
{
	point<int> p;
	p.print(3.14);	// 因为是模板函数, 所以交给编译器自行推导

	exit(0);
}
```



3.  **类模板中可以声明static成员, 在类外定义的时候要增加template相关的关键词, 并且需要注意<font color=#b20>每个不同的模板实例都会有一个独有的static成员对象</font>**.

```c++
template<class T>
class tmp
{
	public:
		static T t;	//静态
};

template<class T>
T tmp<T>::t = 0;

int main()
{
	tmp<int> t1;
	tmp<int> t2;
	tmp<double> t3;
	t1.t = 1;
	std::cout << "t1.t = " << t1.t << endl;
	std::cout << "t2.t = " << t2.t << endl;
	cout << "t3.t = " << t3.t << endl;

	exit(0);
}
```

输出结果:

```c++
t1.t = 1
t2.t = 1
t3.t = 0
```

**模板中的`static`是在每个不同的类型实例化一个, 相同类型的实例化对象共享同一个参数.** 所以这里的t1, t2中哦t都是同一个实例化变量, 是共享的.



## 总结

本节只是简单讲了`template`在实例化时期和使用static, 嵌套模板的使用和注意, 下一节着重分析template的非类型参数.
