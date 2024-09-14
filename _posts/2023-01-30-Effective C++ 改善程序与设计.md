---
layout:     post
title:      "Effective C++ 改善程序与设计的方法"
subtitle:   " \"改善程序与设计的方法\""
date:       2023-01-30 12:00:00
author:     "Puppetsho"
header-img: "img/post-bg-jojo.png"
catalog: true
tags:
    - 笔记
    - 分享
    - C++
---

> “不适合人类阅读，非常水的自我笔记. ”

#### 前言

​	最近在读《Effective C++》，学习研究关于如何改善C++的程序与设计，笔者之前是在C语言的基础上，产生了对C++这门面向对象编程语言的浓厚兴趣，为了进一步提升自己的程序设计水平，于是拿出了这本《Effective C++》来作为学习、记录以及反思，这里就主要选取个人认为比较常用或比较有用的改善C++程序与设计的方法。。

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
		
		Empty &operator=(const Empty& rhs)	//赋值操作
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

​		这点并不复杂，主要有两个点需要注意：

- ​	复制一个RAII（或者说资源管理类）对象必须一并复制它所管理的资源，所以资源的copying行为决定RAII对象的copying行为。
- 普遍而常见的RAII class 的copying行为是：
  （1）**抑制copying**：当你的类并不应该被复制时，应该明确拒绝
  （2）**施行引用计数法**：类似std::shared_ptr，我们希望保有资源直到最后一个使用者对象被销毁
  （3）复制底部资源：如果允许复制行为，复制一个资源管理对象时需要进行**深度拷贝**，确保对应的资源都被复制出一个复件
  （4）**转移底部资源的拥有权**：类似std::auto_ptr，某种情况下你希望只有一个RAII对象指向资源，此时就需要在复制过程中把资源的拥有权从被复制物转移到目标物。
  （5）**注意copying行为包括拷贝构造函数和赋值操作符。**



##### 11、在资源管理类中提供对原始资源的访问

​		资源管理类是对抗资源泄露的好办法，但是许多APIs需要直接访问原始资源，例如有以下函数处理：

```
int daysHeld(const Investment *p1);			//返回投资天数
```

​		此时的API接口需要直接提供一个Investment类型的资源，来返回投资天数，这时候你需要一个函数可将RAII class对象（例如trl::shared_ptr）转换为其内含的原始资源（即底部的Investment *）。一般有两种方法可以达到目的：显示转换和隐式转换。

​		例如trl::shared_ptr和auto_ptr都提供了get成员函数，用来执行显示转换，它会返回智能指针内部的原始指针（的复件）：

```
int days = daysHeld(pInv.get());	//通过get显示得到其内部的Investment*
```

​		同时像所有的智能指针一样，trl::shared_ptr和auto_ptr也重载了指针取值操作符（operatpr->和operator*），它们允许隐式转换至底部原始指针：

```
class Investment {
public:
	bool isTaxFree() const;
	...
};

Investment* createInvestment();		//factory函数

std::trl::shared_ptr<Investment> pi1(createInvestment());
bool taxable1 = !(pi1->isTaxFree());	//经由operator-> 访问资源

std::auto_ptr<Investment> pi2(createInvestment());
bool taxable2 = !((*pi2).isTaxFree());	//经由operator* 访问资源


```

总结一下：

​	（1）APIs往往要求访问原始资源，所有每一个RAII class 应该提供一个“取得其所管理的原始资源”的办法。

​	（2）对原始资源的访问可能经由显示转换和隐式转换。一般而言显示转换比较安全，隐式转换比较方便。



##### 12、成对使用new 和 delete时采用相同形式

​	考虑一下行为：

```
std::string *stringArray = new std::string[100];
...
delete stringArray;
```

​	此时的new和delete成对存在，但是内存是否有被完全释放掉？答案自然是不可预知的。最低限度，stringArray所含的100个string对象中有99个不太可能被适当删除，因为它们的析构函数很可能没被调用。

​	当你使用new来生成一个对象时，有两件事情发生：（1）内存被分配出来（通过名为 operator new的函数）；（2）针对此内存会有一个或多个构造函数被调用。
​	当你使用delete来释放内存时，也有两件事情发生：（1）针对此内存会有一个或更多析构函数被调用；（2）内存被释放（通过名为operator delete的函数）。delete最大的问题在于：即将被删除的内存之内究竟存有多少个对象？这个问题的答案也决定了有多少个析构函数必须被调用起来。

​	简单来说，即将被删除的那个指针，所指的是单一对象或对象数组？这是必须搞清楚的问题，因为单一对象的内存布局一般而言不同于数组的内存布局，数组所用的内存通常会包括**“数组大小”**这段记录，以便delete知道需要调用多少次析构函数，而单一对象的内存则不会有这段记录，你可以理解成以下模型。

![image.png](https://s2.loli.net/2022/09/06/a6TPlMeuvS3t7IR.png)

​	当你对着一个指针使用delete，唯一能够让delete知道内存中是否存在一个“数组大小”记录的办法就是你来明确告诉它。如果你使用delete时加上方括号，delete便认定指针指向一个数组，否则认定指向单一对象。再考虑一下情况：

```
std::string *stringArray = new std::string;
...
delete []stringArray;
```

​	可能会发生什么？结果难以预期，假设内存布局如上述单一对象和对象数组模型，delete会读取若干内存并将它们解释为**“数组大小”**，然后调用多次析构函数，浑然不知它所处理的那块内存不但不是个数组，也或许并未持有它忙着销毁的那种类型的对象。

总结一下：

​	（1）如果在new表达式中使用了[]，必须在相应delete表达式中也使用[]；如果new表达式没有使用[]，一定不要在相应的delete表达式中使用[]。



##### 13、以独立语句将newed对象置入智能指针

​	考虑以下调用情况：

```
//有以下两个接口
int priority();
void processWidget(std::trl::shared_ptr<Widget>pw, int priority);

//现在考虑调用processWidget
processWidget(std::trl::shared_ptr<Widget>(new Widget),priority());
```

​	我们现在分析这段调用，编译器产出一个processWidget调用码之前，必须首先确定即将被传入的各个实参。上述第二实参只是简单的priority函数调用，但是第一实参由两部分组成：

​	（1）new Widget表达式；

​	（2）trl::shared_ptr构造函数。

​	于是在调用processWidget之前，编译器需要做三件事：

​	（1）调用priority

​	（2）执行new Widget

​	（3）调用trl::shared_ptr构造函数

​	C++编译器以什么样的顺序完成这些事情呢？弹性很大，不确定。这和其他语言如Java和C#总以特定顺序完成函数参数的核算不同。不过可以确定的是 "new Widget"一定在 “trl::shared_ptr”之前（需要new Widget才能执行shared_ptr的构造函数），但是priority的调用难以确定。如果编译器选择第二顺位执行它，我们来看结果：
​	（1）执行new Widget；

​	（2）调用priority；

​	（3）调用trl::shared_ptr构造函数；

​	现在请想想，万一对priority的调用导致了异常，可能会导致 "new Widget"返回的指针遗失，因为它没有被置入trl::shared_ptr中，于是可能就会发生资源泄露，因为在“资源被创建”和 “资源被转换为资源管理对象” 两个时间点之前发生了异常。

​	避免这类问题其实很简单：使用分离语句，分别写出（1）创建Widget，（2）将它置入一个智能指针内，然后再把智能指针传給processWidget：

```
std::trl::shared_ptr<Widget> pw(new Widget);
processWidget(pw,priority());
```

总结：

​	（1）以独立语句将 newed 对象存储于智能指针内。如果不这样做，一旦异常被抛出，很可能导致难以察觉的资源泄露。



#### 设计与声明

##### 所谓软件设计，是“令软件做出你希望它做的事情”的步骤和做法，通常以颇为一般性的构想开始，最终演变成十足的细节，以允许特殊接口的开发。这里用最重要、合适任何接口设计的一个准则作为开端：“让接口容易被正确使用，不容易被误用”。

##### 14、让接口容易被正确使用，不易被误用

​	C++在接口之海漂浮。function接口、class接口、template接口......每一种接口都是客户与你的代码互动的手段。假设你面对的是一群“讲道理的人”，那些客户企图把事情做好。他们想要正确使用你的接口，这种情况下如果他们对任何其中一个接口的用法不正确，你可能也需要承担一部分责任。

​	欲开发一个“容易被正确使用，不容易被误用”的接口，首先必须考虑客户可能做出什么样的错误。假设你为一个用来表现日期的class设计构造函数：

```
class Date {
public:
	Date(int month, int day, int year);
};
```

​	乍见之下这个接口通情达理，但它的使用者很容易犯下至少两个错误。第一，他们也许会以错误的次序传递参数：

```
Date d(30,3,1995);//次序错误，应该是3,30,1995
```

第二，传递一个无效的月份或天数：

```
Date d(2,30,1995); //2月并没有30天
```

许多错误可以通过导入新类型而预防。例如这里可以导入简单的外覆类型来区别天数，月份和年份，然后于Date构造函数中使用这些类型。

```
struct Day{
 explicit Day(int d): val(d){
 }
 int val;
};

struct Month {
 explicit Month(int m):val(m){}
 int val;
};

struct Year{
 explicit Year(int y):val(y){}
 int val;
};

class Date{
public:
	Date(const Month &m, const Day& d, const Year& y);
	...
};

Date d(Month(3), Day(30), Year(1995));
```

​	令Day，Month和Year成为成熟且经充分锻炼的class并封装其内数据，比简单使用上述的structs好。



##### 15、设计class如同设计type

​	我们在设计一个新的class时，也就相当于定义了一个新的type，也就意味着你不仅仅是个class设计者，还是个type设计者（类比int、float这些内置类型）。重载函数、操作符、控制内存的分配和归还、定义对象的初始化和终结等一系列操作，都在我们手中，因为我们在设计class时应该带着和“语言设计者当初设计内置类型时”一样的谨慎来设计class。

​	那么在设计class时，我们必须了解我们可能要面临的问题，因此当面对以下一些提问时，你的回答往往导致你的设计规范：

- **新的type(类型)对象应该如何被创建和销毁？**这会影响到class的构造函数和析构函数以及内存分配函数和释放函数（operator new，operator new[]，operator delete和operator delete[]）的设计，不过前提是你的class打算拥有它们。

- **对象的初始化和对象的赋值该有什么样的差别？**这个答案决定你的构造函数和赋值操作符的行为以及期间的差异。很重要的是不能混淆了“初始化”和"赋值"，它们对应的是不同的函数调用。

- **新type的对象如果被passed by value（以值传递），意味着什么？**记住，copy构造函数用来定义一个type的passed by value（以值传递）该如何实现。

- **什么是新type的合法值？**对class的成员变量而言，通常只有某些数值集是有效的。那些数值集决定了你的class必须维护的约束条件，也就决定了你的成员函数（特别是构造函数、赋值操作符和所谓的"setter"函数（setter函数又叫存值函数，一般指对某个属性的赋值、存值操作)）必须进行的错误检查工作。它也影响函数的抛出异常。

- **你的新type需要配合某个继承图系嘛？**如果你继承自某些既存有的classes，你就受到那些classes的设计束缚，特别是受到“它们的函数是“virtual”或"non-virtual"的影响。如果你允许其他classes继承你的class，那会影响你所声明的函数——尤其是析构函数——是否为virtual。

- **你的新type需要什么样的转换？**你的type生存于其他一系列的type之间，因此彼此该有转换行为嘛？如果你允许类型T1转换为类型T2，就必须在class T1内写一个类型转换函数（类比QString的toInt）或在class T2内写一个可被单一实参调用的构造函数。如果你不想在这两个class内写格外的转换函数，那么你必须写一个专门负责转换的函数，来处理这种情况。

- **什么样的标准函数应该驳回？**那些正是你必须声明为private的函数。

- **谁该取用新的type的成员？**这个提问可以帮助你决定哪些成员为public、protected、private。它也可以帮助你决定哪一个classes或functions应该是friends，以及将它们嵌套于另一个之内是否合理。

- 是否真的需要一个新的type？如果只是定义新的衍生类以便为既有的class添加机能，那么说不定单纯定义一或多个非成员函数或templates（模板），更能够达到目标。

  在设计一个class时，这些问题都不好回答。当我们在设计时，应该向C++内置类型看齐，设计一个好的用户自定义classes是非常值得的。

  

  未完待续...
