---
layout:     post
title:      "Google Leveldb --- Slice"
subtitle:   " \"学习谷歌开源项目Leveldb的一些笔记\""
date:       2022-05-11 12:00:00
author:     "Puppetsho"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - 笔记
    - 分享




---

> “不适合人类阅读，非常水的自我笔记. ”

#### 1.0 什么是Slice？

​		切片是leveldb中的数据结构，包括指向某些外部存储的指针和大小，非常类似于string，比较奇怪的是，为什么谷歌不直接C++自带的string这样的结构呢？可能有如下一些原因

> - 相对于拷贝`string`，拷贝`Slice`会轻量级很多，只需要拷贝指针和长度；
> - `char []`无法保存二进制字节，而`Slice`则没有这个问题；
> - `Slice`的底层可以是`char`，也可以是`string`;
> - 多个`Slice`可以指向同一个字符串。

​		所以采用Slice而不是string，避免了大量的字符串拷贝，提高了效率。

#### Slince源码分析

##### （1）类结构

```
class LEVELDB_EXPORT Slice {
 public:
  // 一些成员函数
  
  private:
  	const char* data_;
 	size_t size_;
};
```

​		其成员变量非常简单，包含char*的字符串指针和size大小。

##### （2）构造方式

​		在Slice中，一共有四种构造方法



```
class LEVELDB_EXPORT Slice {
 public:
  // 创建空字符串.
  Slice() : data_(""), size_(0) {}
  
  // 通过字符串指针和size初始化一个Slice.
  Slice(const char* d, size_t n) : data_(d), size_(n) {}
  
  // 通过String初始化Slice
  Slice(const std::string& s) : data_(s.data()), size_(s.size()) {}
  
  // 通过字符串指针初始化Sline
  Slice(const char* s) : data_(s), size_(strlen(s)) {}
}
```

##### （3）拷贝构造与拷贝赋值

​		这里的拷贝构造和拷贝赋值，采用了C++11 =default 特性来实现。

```
  // Intentionally copyable.
  Slice(const Slice&) = default;
  Slice& operator=(const Slice&) = default;
```

​		这里简单介绍一下 =default特性

> ​		首先明确一个概念：在C++中，声明自定义的类型之后，编译器会默认生成一些成员函数，这些函数被称为默认函数（这些默认函数的效率通常都比手写的要高）。但是如果一个类中自定义了带参数的构造函数，那么编译器就不会再自动生成默认构造函数，也就是说该类将不能默认创建对象，只能携带参数进行创建一个对象。
>
> ​		但有时候需要创建一个默认的对象但是类中编译器又没有自动生成一个默认构造函数，那么为了让编译器生成这个默认构造函数就需要default这个属性，让编译器生成一个合成版本的构造/析构函数（包括拷贝构造，赋值构造，移动构造，移动赋值构造）。



##### （4）基础函数

```
  // Return a pointer to the beginning of the referenced data
  const char* data() const { return data_; }

  // Return the length (in bytes) of the referenced data
  size_t size() const { return size_; }

  // Return true iff the length of the referenced data is zero
  bool empty() const { return size_ == 0; }
  
  // Change this slice to refer to an empty array
  void clear() {
    data_ = "";
    size_ = 0;
  }
  
  // Return a string that contains the copy of the referenced data.
  std::string ToString() const { return std::string(data_, size_); }
```



##### （5）特殊操作函数

​		starts_with 是去判断当前切片指针指向的存储区和另外一个切片存储区的前n个字节是否一致。

在这里使用memcmp函数进行比较同时检查当前的切片长度是否小于另一个切片长度。

​		remove_prefix是对当前切片进行修改，修改前n个字节。

​		compare是比较两个切片。比较规则跟memcmp函数类似，但是当相等的时候要根据两个切片长度进行判断一次。例如：“abc”与“ab”比较，"abc"长度大于"ab"则会返回+1，"abc"与"abc"长度相等才返回0。

```
  // Return true iff "x" is a prefix of "*this"
  bool starts_with(const Slice& x) const {
    return ((size_ >= x.size_) && (memcmp(data_, x.data_, x.size_) == 0));
  }
  
  // Drop the first "n" bytes from this slice.
  void remove_prefix(size_t n) {
    assert(n <= size());
    data_ += n;
    size_ -= n;
  }
  
inline int Slice::compare(const Slice& b) const {
	const size_t min_len = (size_ < b.size_) ? size_ : b.size_;
	int r = memcmp(data_, b.data_, min_len);
	if (r == 0) {
	if (size_ < b.size_)
		r = -1;
	else if (size_ > b.size_)
		r = +1;
	}
	return r;
}
```



##### （6）操作符重载

​		操作符重载有内部的[]操作符重载，全局== 与 != 操作符重载。

```
// Return the ith byte in the referenced data.
// REQUIRES: n < size()
char operator[](size_t n) const {
    assert(n < size());
    return data_[n];
  }
  
inline bool operator==(const Slice& x, const Slice& y) {
	return ((x.size() == y.size()) &&
          (memcmp(x.data(), y.data(), x.size()) == 0));
}

inline bool operator!=(const Slice& x, const Slice& y) { return !(x == y); }
```

##### （7）memcmp原理

​		在Slice中有大量memcmp，在这里阐述一下内部实现。

```
#include <ansidecl.h>
#include <stddef.h> int memcmp (const PTR str1, const PTR str2, size_t count) 
{ 
	register const unsigned char *s1 = (const unsigned char*)str1; 
	register const unsigned char *s2 = (const unsigned char*)str2; 
	while (count-- > 0) 
	{ 
		if (*s1++ != *s2++) 
		return s1[-1] < s2[-1] ? -1 : 1; 
	} 
	return 0;
}
```

其中PTR表示一个宏：

```
#define PTR char *
```

另外，最核心的在于while循环，在return这里有个数组访问-1操作，大家会觉得不可思议，竟然可以-1操作。实际上是允许这样操作的，在redis源码中，也可以找到类似的操作，假设数组a=[1,2,3]，int* ptr = a + 2，那么ptr[-1]是多少呢？

答案是：2，那如果int *ptr = a，那ptr[-1]是多少呢？此时就是不确定了，有可能直接gg。

也就是说-1在数组有效范围内是可靠的，但是一旦超出了[0, size-1]范围便不可靠。

而上述if语句每次都会++操作，那-1便是访问前一个字符，例如：

```
char *a = "abcd"; 
while (*a++!='d') { 
	cout << a[-1] << endl; 
}
```

```
a
b
c
```

完~