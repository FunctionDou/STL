# typename与class

### 前言

在分析`traits`编程之前, 我们需要对模板参数类型`tempname`和`class`有一定的了解, 要明白他们在哪些方面不同, 哪些方面相同, 这样才能对体会到`traits`编程的核心. 如果你已经明白了两者, 那么你可以直接看下一篇了.



### 相同之处

一般对模板参数类型`typename`和`class`认为是一样的. 这两者在参数类型中确实是一样的. 你可以写成

```c++
template<class T> 
class point {};
```

 也可以写成

```c++
template<typename T>
class point {};
```

这两者都是一样的, 没有区别. <font color=#b20>两者`typename`和`class`在**参数类型**中没有不同</font> 

*既然相同又为什么定义这两个符号呢?*

1.  最开始定义定义模板的方法就是template\<class T> , 但是class毕竟都认为是一个类, 在使用时难免会有些点混淆, 也就定义了typename来标志参数类型
2.  最重要关于 <font color=#b20>typename可以使用嵌套依赖类型, 也就是类型可以嵌套使用.</font> 这也是两个的不同之处.



### 不同之处

`typename`可以用在嵌套依赖中, 并且表示其类型, 而`class`并没有这样的功能. 

*什么是嵌套依赖?* 我们以一个简单的实例来看

```c++
template<class T>
class people
{
	public:
		typedef T	value_type;
		typedef T*	pointer;
		typedef T&	reference;
};

template<class T>
struct man 
{
	public:
		typedef typename T::value_type	value_type;
		typedef typename T::pointer		pointer;
		typedef typename T::reference	reference;
		void print()
		{
			cout << "man" << endl;
		}
};

int main()
{
	man<people<int>> Man;
	Man.print();

	exit(0);
}
```

以上就是`typename`的嵌套使用. `typename`告诉编译器这不是一个函数, 也不是一个变量而是一个类型. 这里使用typedef又将参数类型重新定义一次, 1. 增加了一层间接性, 2. 使用的时候也不需要在写很长的代码.

这里`typename`是对people类中定义的类型进行了一次提取, 这里将`typename`改为`class`就会出错. 

`typename`主要的作用:

-   **对于模板参数是类的时候, `typename`能够提取出该类所定义的参数类型.**

并不是所有的嵌套依赖类型都要加上`typename`, 有一个例外 : <font color=#b20>**当继承列表或成员初始化列表中对基类进行初始化的时候, 可以去掉`typename`关键字**</font>

```c++
man(int x) : T::value_type(x) {}
```



### 总结

这里对`typename`做了一个浅显的分析, 这也足够我们可以分析`traits`编程的基础了. 我再将以上的分析做一个归纳.

1.  `typename`和`class`在作为参数类型时用法一样, 没有区别
2.  `typename`主要用于对嵌套依赖类型进行提取(萃取). 而`class`没有这样的功能.
3.  `typename`提取的一个例外是在继承或成员初始化列表中对基类进行初始化时不用加`typename`关键字