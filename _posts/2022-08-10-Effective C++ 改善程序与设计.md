---
layout:     post
title:      "Effective C++ 改善程序与设计的方法"
subtitle:   " \"改善程序与设计的方法\""
date:       2022-08-10 12:00:00
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

#### 构造/析构/赋值运算

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

​		同时在其他赋值相关运算中，也最好采用这一方式：

```
Widget& operator+=(const Widget& rhs)
{
	...
	return *this;
}
Widget& operator=(int rhs)
{
	...
	return *this;
}
```



##### 7、在operator= 中处理 ”自我赋值“

​		”自我赋值“ 发生在对象自己给自己赋值时：

```
class Widget {...};

Widget w;

w = w;
```

​		此外也存在难以一眼辨识出来的自我赋值：

```
a[i] = a[j]; //i和j可能相等
*px = *py;	//px和py可能指向同一个地址
```

​		但是这种自我赋值可能带来问题，下面看：

```
class Bitmap {...};
class Widget {
	private:
		Bitmap *pb;
};

//此时有一个operator=的实现代码
Widget& Widget::operator=(const Widget& rhs)
{
	delete pb;		//释放原有的bitmap
	pb = new Bitmap(*rhs.pb);	//new一个新的Bitmap，并使用 rhs.pb的副本
	return *this;
}
```

​		以上这种情况，在自我赋值时存在两种不安全，“自我赋值”的不安全和“异常处理”的不安全，因为在自我赋值时pb和rhs.pb是同一个对象，此时的delete pb导致rhs.pb所指的对象已经被释放，从而导致使用了一块已经被释放掉的内存。所以为了避免这种问题，最好在operator=最前面增加一个"证同测试"已达到自我赋值的检验目的。

```
Widget& Widget::operator=(const Widget& rhs)
{
	if(this == rhs) return *this;
	
	delete pb;
	pb = new Bitmap(*rhs.pb);
	return *this;
}
```

​		这种做法解决“自我赋值安全性”是可行的，但是并没有解决“异常处理的安全性”问题。所以为了同时解决这两个问题，可以引出以下代码：

```
Widget& Widget::operator=(const Widget& rhs)
{
	Bitmap* pOrig = pb;			//先记住原有的pb
	pb = new Bitmap(*rhs.pb);	//令pb指向rhs.pb的一个copy
	delete pOrig;				//释放原有的pb内存
	return *this;
}
```

​		这段代码完成了“自我赋值安全性”和“异常处理安全性”，这里阐述一下“异常处理安全性”的条件：

- 不泄露任何资源。
- 不允许数据败坏。

另外还有种常见并且够好的operator= 的写法，其实现手法大致如下：

```
class Widget {
	...
	void swap(Widget& rhs);	//交换*this和rhs的数据
	...
};

Widget& Widget::operator=(const Widget& rhs)
{
	Widget temp(rhs);	//拷贝一份rhs的复件
	swap(temp);			//交换*this和rhs的复件
	return *this;
}
```

这种方式还有另一种写法，主要利用了：(1)某class的赋值操作符可能被声明为"以值传递的方式接受实参"；（2）以值传递方式传递东西会造成一份复件副本：

```
Widget& Widget::operator=(Widget rhs)	//以值传递，获得的是传递对象的一份复件
{
	swap(rhs);
	return *this;
}
```

以上方式可以在之前学习的Leveldb中可以看到类似写法，例如Status的赋值运算。

```
inline Status& Status::operator=(const Status& rhs) {
  // The following condition catches both aliasing (when this == &rhs),
  // and the common case where both rhs and *this are ok.
  if (state_ != rhs.state_) {	//证同测试
    delete[] state_;
    state_ = (rhs.state_ == nullptr) ? nullptr : CopyState(rhs.state_);
  }
  return *this;
}
inline Status& Status::operator=(Status&& rhs) noexcept {
  std::swap(state_, rhs.state_);	//运用swap
  return *this;
}
```



##### 8、复制对象时勿忘其每一个成分                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    

​		**一般在程序设计时，我们会把对象内部封装起来，只留下拷贝构造函数和赋值操作符，统称为copying函数。当你自己声明copying函数时，应该要确保复制"对象内的所有成员变量"以及”所有基类部分“。**

​		例如在一个类的copying函数中，请确保你完成了以下两点：（1）复制所有local成员变量；（2）调用所有基类中适当的copying函数，确保基类的成员变量也完成了复制。例如你可以参考以下代码：

```
void logCall(const std::string& funName);	//日志记录函数
class Customer{
	public:
		...
		Customer(const std::Customer& rhs);
		Customer& operator=(const Customer& rhs);
		...
	private:
		std::string name;
};
Customer::Customer(const Customer& rhs)
:name(rhs.name)
{
	logCall("Customer copy constructor");
}
Customer& Customer::operator=(const Customer& rhs)
{
	logCall("Customer copy assignment operator");
	name = rhs.name;
	return *this;
}

class PriorityCustomer::public Customer{
	public:
		...
		PriorityCustomer(const PriorityCustomer& rhs);
		PriorityCustomer& operator=(const PriorityCustomer& rhs);
	private:
		int priority;
};

PriorityCustomer::PriorityCustomer(const PriorityCustomer& rhs)
:Customer(rhs),		//调用基类拷贝构造函数
 priority(rhs.priority)
{
	logCall("PriorityCustomer copy constructor");
}
PriorityCustomer& PriorityCustomer::operator=(const PriorityCustomer& rhs)
{
	logCall("PriorityCustomer copy assignment operator");
	Customer::operator=(rhs);	//调用基类的赋值函数，对基类部分的成员变量进行赋值
	priority = rhs.priority;	//对自己的成员变量进行赋值
	return *this;
}
```

​		另外不要尝试以某个copying函数实现另一个copying函数，尽管它们之间有相同或相似的部分，你应该把共同机能部分放入第三个函数中（例如再写一个init函数），供copying函数调用。



#### 资源管理

##### 常见的系统资源包括：内存、文件描述器、互斥锁、图形界面中的字型和笔刷、数据库连接以及网络sockets，无论哪种资源，当你不再使用它时，必须将它归还给系统。



##### 9、以对象管理资源

​		**方法结论：**

- **为防止资源泄漏，请使用RAII对象，它们在构造函数中获得资源并在析构函数中释放资源。**
- **两个常被使用的RAII classes 分别是trl::shared_ptr和auto_ptr。前者通常是较佳选择，因为其copy行为比较直观。若选择auto_ptr，复制动作会使它（被复制物）指向null。**

​		现在考虑一个基类，并有各种各样的子类继承自这个基类,并由一个工厂函数提供给资源请求者某种特定的对象

```
enum InvestmentType{TypeA, TypeB, TypeC};

class Investment{...};

Investment* createInvestment(InvestmentType type);
```

​		现有函数f()调用了createInvestment()来请求资源，得到了返回后的对象，并且有责任在使用完成后删除之。

```
void fun()
{
	Investment* pInvA = createInvestment(TypeA);
	...
	delete pInv;
}
```

​		目前看起来没问题，但是考虑在"..."的实现中，可能存在return或者提前退出的语句，导致控制流最后不会调用到delete，从而导致pInv所指向的资源没有被释放，这不仅导致pInv指向的内存被泄漏，甚至还包括pInv对象保存的资源。

​		为了确保createInvestment返回的资源总能得到释放，我们可以将资源放进对象内进行管理，让对象的析构函数自动释放那些资源。我们同时也可以通过**“智能指针”**（例如std::auto_ptr、std::trl::shared_ptr），其析构函数自动对其所指向对象调用delete来避免资源泄漏。下面来改进以上代码：

```
void f()
{
	std::auto_ptr<Investment> pInvA(createInvestment(TypeA));	//调用createInvestment，在																  //结束使用经由auto_ptr的析构																 //函数自动删除pInv
	...
}
```

​		其实你也可以修改工厂函数，使其调用createInvestment()后返回的就是一个**"智能指针"**，来提醒后来的程序员，让他们意识到释放资源。例如你可以这样写createInvestment()：

```
auto_ptr<Investment> createInvestment(InvestmentType type){
	switch(type){
		case TypeA: return auto_ptr<InvestmentType>(new InvestmentTypeA);
		case TypeB: return auto_ptr<InvestmentType>(new InvestmentTypeB);
		case TypeC: return auto_ptr<InvestmentType>(new InvestmentTypeC);
	}
}

void f()
{
	std::auto_ptr<Investment> pInvA = createInvestment(TypeA);
	...
}
```

​		以上例子都示范了"以对象管理资源"的两个关键想法：

- **获得资源后立刻放进管理对象内。**以上代码中createInvestment()返回的资源都被当做其管理者auto_ptr的初值，用auto_ptr来管理。实际上“以对象来管理资源”的观念常被称为“资源取的时机便是初始化时机”（RAII），因为我们几乎总是在获得一笔资源后与同一语句内以它初始化某个管理对象例如：

  ```
  std::auto_ptr<Investment> pInvA(createInvestment(TypeA))
  ```

  有时候获得的资源被拿来赋值，而非初始化，例如

  ```
  std::auto_ptr<Investment> pInvA = createInvestment(TypeA)
  ```

  但无论哪一种，每一笔资源都在获得的同时立刻被放进了管理对象中。

- **管理对象运用析构函数确保资源被释放。**不论控制流如何离开区块，一旦对象被销毁（例如当对象离开作用域）其析构函数自然会被自动调用，于是资源被释放。如果资源释放动作可能导致抛出异常，就可以采用之前提到的第四点。

​		同时RAII对资源的所有权分为常性类型和变性类型，代表者分别是shared_ptr<>和auto_ptr<>。**常性类型**指获取资源的地点是构造函数，释放点是析构函数，并且在这两点之间，任何对该RAII类型实力的操作都不会从它手中夺走资源的所有权（例如shared_ptr<>）；变性类型是指可以中途被设置为接管另一个资源，或者干脆被置为null，不拥有任何资源（例如auto_ptr<>）。

​		由于auto_ptr被销毁时会自动删除它所指之物，所以一定不要让多个auto_ptr同时指向同一对象。同时auto_ptr有个性质：若通过copy构造函数或copy assignment操作符复制它们，它们会变成null，而复制所得的指针将获取资源的唯一拥有权。

```
std::auto_ptr<Investment> pInv(createInvestment());
std::auto_ptr<Investment> pInv2(pInv1);	//拷贝构造，会导致pInv1设为null，pInv2指向对象
pInv1 = pInv2;		//赋值操作符会导致pInv1指向对象，pInv2设为null
```

​		由于以上auto_ptr<>的性质，auto_ptr管理的资源绝对没有一个以上的auto_ptr指向它，意味着auto_ptr并非管理动态分配资源的神兵利器。例如STL容器要求元素发挥正常复制行为，因此这时候用不了auto_ptr，可以采用trl::shared_ptr。

```
void f()
{
	std::trl::shared_ptr<Investment> pInv1(createInvestment());
	std::trl::shared_ptr<Investment> pInv2(pInv1);	//拷贝构造，使pInv1和pInv2指向同一个对象
	pInv1 = pInv2;									//同上
	...
}												//pInv1和pInv2被销毁，它们所指对象也就被自动销毁
```

​		另外需要提一点，auto_ptr和trl::shared_ptr两者都在其析构函数内做delete而不是delete[]，于是如果动态分配得到的是array，并不适合用auto_ptr和trl::shared_ptr（要用的话，可以参考看看boost::scoped_array和boost::shared_array）。



##### 10、在资源管理类中小心copying行为

...未完待续
