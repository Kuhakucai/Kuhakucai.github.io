---
layout:     post
title:      "Effective C++ 改善程序与设计的方法"
subtitle:   " \"改善程序与设计的方法\""
date:       2022-08-03 12:00:00
author:     "Puppetsho"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - 笔记
    - 分享
    - C++
---

> “不适合人类阅读，非常水的自我笔记. ”

#### 前言

​	最近在读《Effective C++》，学习研究关于如何改善C++的程序与设计，笔者之前是在C语言的基础上，产生了对C++这门面向对象编程语言的浓厚兴趣，为了进一步提升自己的程序设计水平，于是拿出了这本买了很久，但是一直没仔细看过的《Effective C++》来作为学习、记录以及反思，这里就主要选取个人认为比较常用或比较有用的改善C++程序与设计的方法。。

#### 正文

##### 1、C++中的默认函数

​		封装作为C++面向对象的三大特征之一，其Class是关键字，在我们构造一个类的时候，需要清楚的知道编译器会帮你声明出哪些默认函数，并且这些默认函数只有在被调用（被需要）的时候，才会被编译器创建出来，另外所以这些函数都是public且inline。

​		其声明的默认函数如下：

```
class Empty{
	public:
		Empty() {...}	//默认构造函数
		Empty(const Empty& rhs) {...}	//拷贝构造函数
		~Empty() {...}	//析构函数
		
		Empty &operator()	//赋值操作
}
```

需要注意的是，当类中内涵const成员或引用，不是简单的内置类型的成员变量时，编译器会拒绝生成赋值操作（copy assignment操作）

##### 2、不想编译器自动生成默认函数，就需要明确拒绝

​		我们知道类中有默认函数，它会被默认生成，但是如果我们不需要某些默认函数，并且也不想要编译器去自动生成，我们可以将对应的成员函数声明为private并且只声明不定义。

##### 3、多态基类应该声明virtual析构函数

​		如果一个基类带有多态性质，那么我们应该为其声明一个virtual的析构函数。因为在C++中明确指出，当一个继承类对象经由一个基类指针被删除，而基类中带有的是non-virtual析构函数，那么可能导致继承类中的成员变量没有被销毁，从而可能导致内存泄漏。例如看以下代码：

```
class TimeKeeper{		//计时方式
	public：
		TimeKeeper();
		~TimeKeeper();
};

class AtomicClock : public TimeKeeper{...};		//原子钟
class WaterClock ： public TimeKeeper{...};		//水钟

//如果此时有一个接口，可以返回指向一个Parent派生类的动态分配对象
TimeKeeper *getTimeKeeper();
//假如调用该接口
TimeKeeper *p = getTimeKeeper();
//再释放它
delete p;
```

​		此时释放后，由于基类的析构函数是non-virtual，如果getTimeKeeper()返回了一个AtomicClock对象，而AtomicClock中的成员变量就很可能没被销毁，从而导致内存泄漏。

​		所以如果一个基类带有virtual函数，那么它应该同样带有virtual的析构函数。另外当一个类设计的目的不是因为基类或者没有带多态性质时，就不该为其声明virtual析构函数，因为virtual函数是需要一个额外的虚函数指针（virtual table point）指出的，这代表对象的体积得到了增加，但是这显然是没有必要的。

##### 4、别让异常逃离析构函数

​		析构函数绝对不要吐出异常，因为它可能导致在释放内存的时候被异常中止，从而导致内存泄漏。因此如果一个被析构函数调用的函数可能抛出异常，那么析构函数应该要捕捉任何异常，然后吞下它们（不传播）或结束程序。

##### 5、绝不在构造和析构过程中调用virtual函数

​		当你在编写代码时，在构造函数和析构函数中，你一定不能调用virtual函数，因为此类调用从不下降至派生类（derived class），例如当你有一个基类，并同时又一个派生类：

```
class Transaction {	//一个基类
public：
	Transaction();
	virtual void log(); //一个因类型不同而不同的日志记录函数
	...
};

Transaction::Transaction()	//基类的构造函数实现
{
	...
	log(); //操作完成后调用日志记录
}

class BuyTransaction()
{
public:
	virtual void log();
	...
};
```

此时，如果执行以下语句，会发生什么？

```
BuyTransaction b;
```

​		此时一定会有BuyTransaction的构造函数被调用，并在此之前Transaction构造函数更早被调用（派生类对象内的基类成分会在派生类自身成分被构造之前先构造妥当）。那此时Transaction构造函数里最后一行的virtual函数log()的调用，会是Transaction内的版本，而不是BuyTranacction版本——即使即将建立的对象类似是BuyTransaction。但是为什么呢？

​		很简单，由于基类构造函数的执行更早于派生类构造函数，当基类构造函数执行时，派生类的成员变量一定没有初始化，如果此时的调用virtual函数下降至派生类阶层，那么使用没有初始化的成员变量，这一行为将会是一个不明确行为，使用对象内部尚未初始化的成分是危险的，C++肯定不会让你这样去使用。

​		还有个更为直观的解释：在派生类对象的基类构造期间，对象的类型是基类而不是派生类。

​		但是如何保证在基类的继承体系上的对象被创建时，就会有适当版本的log被调用呢？有一种做法，例如将class Transaction内将log()函数改为非虚函数，然后要求派生类的构造函数传递必要信息给Transaction构造函数，而后那个构造函数就可以安全地调用log()。

```
class Transaction{
	public：
		explicit Transaction(const std::string& logInfo); //单参构造函数最好显示转换，避免不符合预期的转换
		void log(const std::string& logInfo); //non-virtual
		...
};

Transaction::Transaction(const std::string& logInfo)
{
	...
	log(const std::string& logInfo);
}

class BuyTransaction: public Transaction {
	public:
		BuyTransaction(parameters)
			:Transaction(createLogString(parameters))	//通过参数将log信息传給基类构造函数
			{...}
		...
    private:
    	static std::string createLogString(parameters);
};
```

​		这里通过在派生类中将必要信息向上传递至基类来达到调用log()，生成不同log的结果。

##### 6、令operator= 返回一个reference to *this

​		在赋值中，你可以将赋值写成连锁形式:

```
int a,b,c;
a = b = c = 15;
```

​		然而为了实现连锁赋值，赋值操作符必须返回一个referen 指向操作符的左侧实参。例如：

```
Widget& operator=(const Widget& rhs)
{
	...
	return *this;
}
```

...未完待续
