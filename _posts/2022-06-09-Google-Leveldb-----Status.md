---
layout:     post
title:      "Google Leveldb --- Status"
subtitle:   " \"学习谷歌开源项目Leveldb的一些笔记\""
date:       2022-06-09 12:00:00
author:     "Puppetsho"
header-img: "img/post-bg-halting.jpg"
catalog: true
tags:
    - 笔记
    - 分享
    - Google Leveldb

---

> “不适合人类阅读，非常水的自我笔记.”

#### 1. Status类

------

​	首先来看一下Status内的基本介绍。Status内部含有Code枚举值，可以看到Code里面含有0-5个状态。

​	私有成员status_为const char *，其前4个字节代表消息长度，对应[0,3]，第4个字节代表消息码，对应[4]，5个字节之后代表实际的消息体，对应[5,...]。

```
class LEVELDB_EXPORT Status {
private:
	enum Code {
		kOk = 0,
		kNotFound = 1,
		kCorruption = 2,
		kNotSupported = 3,
		kInvalidArgument = 4,
		kIOError = 5
 };
	// OK status has a null state_. Otherwise, state_ is a new[] array
	// of the following form:
	// state_[0..3] == length of message
	// state_[4]   == code
	// state_[5..] == message
  	const char* state_;
};
```

说到这里，私有成员还有：code、Status构造、CopyState。

##### （1）code函数

```
Code code() const {
	return (state_ == nullptr) ? kOk : static_cast<Code>(state_[4]);
}
```

code在返回的时候，根据代码可以知道，当state_ 为nullprt时，返回的是kOk状态，否则从state_中取第四个字节的Code。

##### （2） Status构造

在下面的构造中code被断言不为kOk，传递了两个msg，都为Slince类型。

len1表示msg的大小；len2表示msg2的大小。

size表示len1 + len2 + 2

2表示在msg与msg2中间添加一个冒号与空格（不太理解为什么要用冒号和空格隔开msg和msg2，会有什么好处嘛？如果只是为了区分msg和msg2，为什么要用两个字符呢？）。

最后拼接的结果需要给结果大小加5，5分为4 + 1,4表示前4字节是state_[0...3]大小，1表示 state__[4] 是code类型。

下面三行代码对应的是给result填充内容：

```
std::memcpy(result, &size, sizeof(size)); 
result[4] = static_cast<char>(code); 
std::memcpy(result + 5, msg.data(), len1);
```

memcpy这一行，表示将size内存地址的内容拷贝到result地址中，这里刚好拷贝占据[0...3]，这样就可以把前4个字节的内容填充为result的大小；

result[4]填充为code类型，这里是一个字节所以使用static_cast<char>，做了强转；

最后拷贝真实的msg中的内容到5字节之后，即[5...]。

随后根据msg2的大小是否为空进行后续操作，如果不为空，则继续拷贝，首先在5 + len1的位置填充冒号，在后一位填充空格，最后把len2拷贝进去，这里result + 7 + len1表示的是：5 + len1 + 2 = 7 + len1，指向要拷贝msg2的起始位置。

最终结果就是：“sizecodemsg： msg2”。这里是size + code + msg + 冒号 + 空格 + msg2。

```
Status::Status(Code code, const Slice& msg, const Slice& msg2) {
  assert(code != kOk);
  const uint32_t len1 = static_cast<uint32_t>(msg.size());
  const uint32_t len2 = static_cast<uint32_t>(msg2.size());
  const uint32_t size = len1 + (len2 ? (2 + len2) : 0);
  char* result = new char[size + 5];
  std::memcpy(result, &size, sizeof(size));
  result[4] = static_cast<char>(code);
  std::memcpy(result + 5, msg.data(), len1);
  if (len2) {
    result[5 + len1] = ':';
    result[6 + len1] = ' ';
    std::memcpy(result + 7 + len1, msg2.data(), len2);
  }
  state_ = result;
}
```

这里举个简单的例子：

```
uint32_t sz = 65; 
char* r = new char[5]; 
memcpy(r, &sz, sizeof(sz)); 
cout << r << endl; // A
```

例如：数字65对应的ascii为'A'，那么经过memcpy拷贝之后便是'A'。

同理我们想得到size，可以这样：

```
uint32_t size; 
memcpy(&size, r, sizeof(size)); 
cout << size << endl; // 65
```

那么此时size就是65。

##### （3） CopyState函数

首先通过memcpy拿到state[0...3]所表示的state大小，拿到之后构造了新的result，再加上5，随后拷贝state的实际内容到5字节之后，并返回result。

```
const char* Status::CopyState(const char* state) {
  uint32_t size;
  std::memcpy(&size, state, sizeof(size));
  char* result = new char[size + 5];
  std::memcpy(result, state, size + 5);
  return result;
}
```

#### 2.构造析构与拷贝移动

------



- 构造与析构

  这个很简单，就是设置nullptr与delete[]。

  ```
  // Create a success status. 
  Status() noexcept : state_(nullptr) {} 
  ~Status() { delete[] state_; }
  ```

- 拷贝操作

  拷贝构造与拷贝赋值

  ```
  Status(const Status& rhs); 
  Status& operator=(const Status& rhs);
  ```

拷贝构造：内部实际就是调用了CopyState函数。

```
inline Status::Status(const Status& rhs) { 
	state_ = (rhs.state_ == nullptr) ? nullptr : CopyState(rhs.state_); 
}
```

拷贝赋值：判断是否是自身，不是自身删除原先的state_ ，和拷贝构造操作一样。

```
inline Status& Status::operator=(const Status& rhs) {
  // The following condition catches both aliasing (when this == &rhs),
  // and the common case where both rhs and *this are ok.
  if (state_ != rhs.state_) {
    delete[] state_;
    state_ = (rhs.state_ == nullptr) ? nullptr : CopyState(rhs.state_);
  }
  return *this;
}
```

- 移动操作

  ```
  Status(Status&& rhs) noexcept : state_(rhs.state_) { rhs.state_ = nullptr; } 
  Status& operator=(Status&& rhs) noexcept;
  ```

移动构造：浅拷贝

```
Status& operator=(Status&& rhs) noexcept;
```

移动赋值：直接交换

```
inline Status& Status::operator=(Status&& rhs) noexcept { 
	std::swap(state_, rhs.state_); 
	return *this; 
}
```

#### 3.一些函数

------

Status还提供了构造出不同Status的函数

```
  // Return a success status.
  static Status OK() { return Status(); }

  // Return error status of an appropriate type.
  static Status NotFound(const Slice& msg, const Slice& msg2 = Slice()) {
    return Status(kNotFound, msg, msg2);
  }
  static Status Corruption(const Slice& msg, const Slice& msg2 = Slice()) {
    return Status(kCorruption, msg, msg2);
  }
  static Status NotSupported(const Slice& msg, const Slice& msg2 = Slice()) {
    return Status(kNotSupported, msg, msg2);
  }
  static Status InvalidArgument(const Slice& msg, const Slice& msg2 = Slice()) {
    return Status(kInvalidArgument, msg, msg2);
  }
  static Status IOError(const Slice& msg, const Slice& msg2 = Slice()) {
    return Status(kIOError, msg, msg2);
  }
```

随后也给出了判断当前返回Code是哪个enum。

```
  // Returns true iff the status indicates success.
  bool ok() const { return (state_ == nullptr); }

  // Returns true iff the status indicates a NotFound error.
  bool IsNotFound() const { return code() == kNotFound; }

  // Returns true iff the status indicates a Corruption error.
  bool IsCorruption() const { return code() == kCorruption; }

  // Returns true iff the status indicates an IOError.
  bool IsIOError() const { return code() == kIOError; }

  // Returns true iff the status indicates a NotSupportedError.
  bool IsNotSupportedError() const { return code() == kNotSupported; }

  // Returns true iff the status indicates an InvalidArgument.
  bool IsInvalidArgument() const { return code() == kInvalidArgument; }
```

最后给出了ToString函数，将当前Status转换为对应的String表示的Status。

```
// Return a string representation of this status suitable for printing.
// Returns the string "OK" for success.
std::string Status::ToString() const {
  if (state_ == nullptr) {
    return "OK";
  } else {
    char tmp[30];
    const char* type;
    switch (code()) {
      case kOk:
        type = "OK";
        break;
      case kNotFound:
        type = "NotFound: ";
        break;
      case kCorruption:
        type = "Corruption: ";
        break;
      case kNotSupported:
        type = "Not implemented: ";
        break;
      case kInvalidArgument:
        type = "Invalid argument: ";
        break;
      case kIOError:
        type = "IO error: ";
        break;
      default:
        std::snprintf(tmp, sizeof(tmp),
                      "Unknown code(%d): ", static_cast<int>(code()));
        type = tmp;
        break;
    }
    std::string result(type);
    uint32_t length;
    std::memcpy(&length, state_, sizeof(length));
    result.append(state_ + 5, length);
    return result;
  }
```

#### 4.测试

下面是测试函数：

```
#include <iostream> 
#include <cstring> 
#include <cassert> 
#include "leveldb/status.h" 
using namespace std; 
using namespace leveldb; 
int main() { 
	uint32_t sz = 65; 
	char* r = new char[5]; 
	memcpy(r, &sz, sizeof(sz)); 
	cout << r << endl; // A uint32_t size; 
	memcpy(&size, r, sizeof(size)); 
	cout << size << endl; // 65 
	delete [] r; 
	
	Status status = Status::NotFound("custom NotFound status message"); 
	Status status2 = std::move(status); 
	assert(status2.IsNotFound() == true); 
	assert(status2.IsIOError() == false); 
	assert("NotFound: custom NotFound status message" == status2.ToString()); 
	return 0; 
}
```

