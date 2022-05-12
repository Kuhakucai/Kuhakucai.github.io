---
layout:     post
title:      "Google Leveldb"
subtitle:   " \"学习谷歌开源项目Leveldb的一些笔记\""
date:       2022-05-09 12:00:00
author:     "Puppetsho"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - 笔记
    - 分享
    - Google Leveldb


---

> “不适合人类阅读，非常水的自我笔记. ”

# 什么是LeveDB?

​		LevelDB是google的一个开源项目，是实现的非常高效的key-value数据库，版本1.2能够支持billion级别的数据量了。 在这个数量级别下还有着非常高的性能，主要归功于它的良好的设计。特别是LSM算法。但是LevelDB只是个C/C++编程语言的库，不包含网络封装，所以无法像一般意义的存储服务器(如 MySQL)那样, 用客户端来连接它. LevelDB 自己也声明, 使用者应该封装自己的网络服务器.

​																																		------参照自百度百科

​		于是我准备拿这个C++开源项目作为练手，去学习学习源码。

# 1、编译源码库

​		首先提一句，笔者的部分学习记录参考至Light-City，另外笔者是直接在Windows下编译LevelDB的，其次使用的方式跟github上提供的不太一样，不需要用Visual Studio，大家自行辨别。

##### （1）下载源码

```
git clone https://github.com/google/leveldb.git
```

##### （2）需要先安装好gcc/g++

​		下载对应版本的gcc/g++源码即可，这里采用的版本是gcc8.1.0,同时把对应的bin目录配置到path（环境变量）中就好，下载时候大家可能会比较疑惑版本后面带的SJLJ、DWARF、SEH代表什么意思，这里贴一下解释。

> **SJLJ** （setjmp / longjmp）： – 可用于32位和64位 – 不是“零成本”：即使不抛出exception，也会造成较小的性能损失（在exception大的代码中约为15％） – 允许exception遍历例如窗口callback
>
> **DWARF** （DW2，dwarf-2） – 只有32位可用 – 没有永久的运行时间开销 – 需要整个调用堆栈被启用，这意味着exception不能被抛出，例如Windows系统DLL。
>
> **SEH** （零开销exception） – 将可用于64位GCC 4.8。

> ​	https://sourceforge.net/projects/mingw-w64/files/

这里笔者下载的是sjlj版本。

##### （3）make

​		在\mingw64\bin目录下面找到mingw32-make.exe，make文件默认是以mingw32-make.exe文件，我们需要复制一份并重命名为make.exe，这样做的目的是让cmake可以识别。

##### （4）cmake

​		然后就是下载cmake，并且配置环境变量。

> https://cmake.org/download/

##### （5）配置CC和CXX环境变量

​		目的是让cmake编译时知道gcc与g++的目录，按照下面图进行配置。

![image-20220509234223128.png](https://s2.loli.net/2022/05/09/LBUNwufMFS5VHpo.png)

##### （6）编译

​		然后在源码根目录创建build目录，并进去执行编译即可，笔者用的git bash界面进行编译的。

```
cmake -DCMAKE_BUILD_TYPE=Release -G "Unix Makefiles" .. && cmake --build .
```

​		最后等编译完成即可。

##### 	（7）测试

​		在源码根目录创建test目录，添加helloworld.cpp来进行测试。

```
#include <cassert>
#include <iostream>
#include "leveldb/db.h"

using namespace std;
using namespace leveldb;

int main() {
	leveldb::DB* db;
	leveldb::Options options;
	options.create_if_missing = true;
	leveldb::Status status = leveldb::DB::Open(options, "testdb", &db);
	assert(status.ok());
	status = db->Put(WriteOptions(), "first", "hello world!");
	assert(status.ok());
	string res;
	status = db->Get(ReadOptions(), "first", &res);
	assert(status.ok());
	cout << res << endl;
	delete db; 
	return 0;
}
```

​		然后编译目标文件采用cmake比较方便。

```
PROJECT(Test)
cmake_minimum_required(VERSION 3.20)

set(BASE_INCLUDES E:/my_project/leveldb/leveldb/include/)
set(BASE_LIBS E:/my_project/leveldb/leveldb/build/)


include_directories(${BASE_INCLUDES})
link_directories(${BASE_LIBS})

add_executable(hw helloworld.cpp)
target_link_libraries(hw leveldb)
```

​		把对应的头文件和库文件目录修改一下即可。

​		最后cmake运行

```
cmake -G "Unix Makefiles" .. && cmake --build .
```

​		即可得到可执行文件hw.exe，最后运行输出了hello world即代表成功。