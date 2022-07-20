---
layout:     post
title:      "Google Leveldb --- Coding"
subtitle:   " \"学习谷歌开源项目Leveldb的一些笔记\""
date:       2022-07-20 12:00:00
author:     "Puppetsho"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - 笔记
    - 分享
    - Google Leveldb

---

> “不适合人类阅读，非常水的自我笔记. ”

#### 前言

​	coding里面包含了变长（Varint，Google的Protocol Buffer就是采用的这种编码方式）编码与定长（Fixed）编码格式。

#### 1.定长编码

涉及的函数包括：

```
void PutFixed32(std::string* dst, uint32_t value);
void PutFixed64(std::string* dst, uint64_t value)
void EncodeFixed32(char* dst, uint32_t value);
void EncodeFixed64(char* dst, uint64_t value);
uint32_t DecodeFixed32(const char* ptr);
uint64_t DecodeFixed64(const char* ptr);
```

​	上述函数从函数名就可以看出分为32位和64位，首先来看Encode，这里的encode比较简单（这里的编码方式让我想起来kiwibuffer）。

##### （1）Encode

​	利用位运算将value处理成小端模式存储到dst中，32与64位函数实现是一致的。

```
// Lower-level versions of Put... that write directly into a character buffer
// REQUIRES: dst has enough space for the value being written

inline void EncodeFixed32(char* dst, uint32_t value) {
  uint8_t* const buffer = reinterpret_cast<uint8_t*>(dst);

  // Recent clang and gcc optimize this to a single mov / str instruction.
  buffer[0] = static_cast<uint8_t>(value);
  buffer[1] = static_cast<uint8_t>(value >> 8);
  buffer[2] = static_cast<uint8_t>(value >> 16);
  buffer[3] = static_cast<uint8_t>(value >> 24);
}

inline void EncodeFixed64(char* dst, uint64_t value) {
  uint8_t* const buffer = reinterpret_cast<uint8_t*>(dst);

  // Recent clang and gcc optimize this to a single mov / str instruction.
  buffer[0] = static_cast<uint8_t>(value);
  buffer[1] = static_cast<uint8_t>(value >> 8);
  buffer[2] = static_cast<uint8_t>(value >> 16);
  buffer[3] = static_cast<uint8_t>(value >> 24);
  buffer[4] = static_cast<uint8_t>(value >> 32);
  buffer[5] = static_cast<uint8_t>(value >> 40);
  buffer[6] = static_cast<uint8_t>(value >> 48);
  buffer[7] = static_cast<uint8_t>(value >> 56);
}
```



##### （2）Decode

​	decode也有两个，对应32和64位，为上述的逆操作。可以这样理解：encode是将value从右往左依次塞入buffer。decode操作则是将buffer从左往右塞入value。

```
// Lower-level versions of Get... that read directly from a character buffer
// without any bounds checking.

inline uint32_t DecodeFixed32(const char* ptr) {
  const uint8_t* const buffer = reinterpret_cast<const uint8_t*>(ptr);

  // Recent clang and gcc optimize this to a single mov / ldr instruction.
  return (static_cast<uint32_t>(buffer[0])) |
         (static_cast<uint32_t>(buffer[1]) << 8) |
         (static_cast<uint32_t>(buffer[2]) << 16) |
         (static_cast<uint32_t>(buffer[3]) << 24);
}

inline uint64_t DecodeFixed64(const char* ptr) {
  const uint8_t* const buffer = reinterpret_cast<const uint8_t*>(ptr);

  // Recent clang and gcc optimize this to a single mov / ldr instruction.
  return (static_cast<uint64_t>(buffer[0])) |
         (static_cast<uint64_t>(buffer[1]) << 8) |
         (static_cast<uint64_t>(buffer[2]) << 16) |
         (static_cast<uint64_t>(buffer[3]) << 24) |
         (static_cast<uint64_t>(buffer[4]) << 32) |
         (static_cast<uint64_t>(buffer[5]) << 40) |
         (static_cast<uint64_t>(buffer[6]) << 48) |
         (static_cast<uint64_t>(buffer[7]) << 56);
}
```

##### （3）Put

​	定义buf，调用Encode，实现将value的Fixed编码塞入dst中。

```
void PutFixed32(std::string* dst, uint32_t value) {
  char buf[sizeof(value)];
  EncodeFixed32(buf, value);
  dst->append(buf, sizeof(buf));
}

void PutFixed64(std::string* dst, uint64_t value) {
  char buf[sizeof(value)];
  EncodeFixed64(buf, value);
  dst->append(buf, sizeof(buf));
}
```

##### 测试：

```
void TestFixedCode() { 
	const int num = 8; 
	uint32_t val = 1365; // 1365 101 01010101 
	string dst; PutFixed32(&dst, val); 
	uint8_t* ptr = reinterpret_cast<uint8_t*>(const_cast<char*>(dst.c_str())); 
	cout << "encode is " << (bitset<num>)(*ptr++) << " " << (bitset<num>)(*ptr++) << " " << endl; // 01010101 00000101 
	uint32_t target = DecodeFixed32(const_cast<char*>(dst.c_str())); 
	cout << "decode is " << target << endl; 
}
```

​	输出，1365二进制表示为：101 0101 0101

```
encode is 01010101 00000101 
decode is 1365
```



#### 2.变长编码

​	变长编码相关函数就比较多了，实现起来也是比较复杂，下面来看一下有哪些函数，以及之间的关联。

```
void PutVarint32(std::string* dst, uint32_t value); 
void PutVarint64(std::string* dst, uint64_t value); void PutLengthPrefixedSlice(std::string* dst, const Slice& value); 
bool GetVarint32(Slice* input, uint32_t* value); 
bool GetVarint64(Slice* input, uint64_t* value); 
ool GetLengthPrefixedSlice(Slice* input, Slice* result); 
const char* GetVarint32Ptr(const char* p, const char* limit, uint32_t* v); 
const char* GetVarint64Ptr(const char* p, const char* limit, uint64_t* v); 
int VarintLength(uint64_t v); 
har* EncodeVarint32(char* dst, uint32_t value); 
char* EncodeVarint64(char* dst, uint64_t value); 
onst char* GetVarint32PtrFallback(const char* p, const char* limit, uint32_t* value);
```

Varint基础：varint是一种使用一个或多个字节序列化整数的方法，会把整数编码位变成字节。对于32位整数来说，varint编码会编码成1~5字节，不超过5字节，64位不超过10字节，会编码位1~10字节。根据统计学来说，小数字使用频率大于大数字，所以利用varint编码能够起到压缩空间的作用。

Varint编码结构分为msb+实际有效位。

msb表示最高有效位，取值可以为0或1,0表示当前字节是该数据的最后一个字节，为1表示后面的字节还是属于当前数据。实际有效位为7位。

例如：8645二进制表示为  100 0011 100 0101，经过编码后的串为：1100010101000011。粉红色标记为msb。

![Snipaste_2022-07-18_15-28-30.png](https://s2.loli.net/2022/07/18/Fha1A5DH8lTn3uK.png)



##### （1）Encode

​	32位编码，将一个整数编码并存入dst中，返回的指针指向末尾位置。

​	1000 0000等于128,127则是0111 1111。后面14、21等依次类推。

​	下面操作便是判断当前给定的32位整数在下面5种if中的哪个if，例如上述8645，则会进入第二个if，也就是1<<14 这个if中，首先执行 | 操作，对应上图蓝色位置。

```
11000101 // 8645二进制后8位 
10000000 // 128二进制表示 
11000101 // 返回本身
```

​	v右移7位得到：100 0011，对ptr继续赋值，得到：0100 0011，对应上图绿色。

​	最终dst存储的是：1100 0101 0100 0011。

```
char* EncodeVarint32(char* dst, uint32_t v) {
  // Operate on characters as unsigneds
  uint8_t* ptr = reinterpret_cast<uint8_t*>(dst);
  static const int B = 128;
  if (v < (1 << 7)) {
    *(ptr++) = v;
  } else if (v < (1 << 14)) {
    *(ptr++) = v | B;
    *(ptr++) = v >> 7;
  } else if (v < (1 << 21)) {
    *(ptr++) = v | B;
    *(ptr++) = (v >> 7) | B;
    *(ptr++) = v >> 14;
  } else if (v < (1 << 28)) {
    *(ptr++) = v | B;
    *(ptr++) = (v >> 7) | B;
    *(ptr++) = (v >> 14) | B;
    *(ptr++) = v >> 21;
  } else {
    *(ptr++) = v | B;
    *(ptr++) = (v >> 7) | B;
    *(ptr++) = (v >> 14) | B;
    *(ptr++) = (v >> 21) | B;
    *(ptr++) = v >> 28;
  }
  return reinterpret_cast<char*>(ptr);
}
```

32位整数最高编码位5位，而64位则可达10位，所以在64位中不会去写10个if，而是采用while循环解决，核心实现同32位。

```
char* EncodeVarint64(char* dst, uint64_t v) {
  static const int B = 128;
  uint8_t* ptr = reinterpret_cast<uint8_t*>(dst);
  while (v >= B) {
    *(ptr++) = v | B;
    v >>= 7;
  }
  *(ptr++) = static_cast<uint8_t>(v);
  return reinterpret_cast<char*>(ptr);
}
```

##### （2）Decode

​	Decode对应是 GetVarint32 与 GetVarint64 。这个操作是上述Encode的逆操作，简言之，从dst中取

出整数放进一个变量中，而在实现的时候是从Slice中取。

```
bool GetVarint32(Slice* input, uint32_t* value) {
  const char* p = input->data();
  const char* limit = p + input->size();
  const char* q = GetVarint32Ptr(p, limit, value);
  if (q == nullptr) {
    return false;
  } else {
  	// limit - q 一般为0
    *input = Slice(q, limit - q);
    return true;
  }
}
```

这一段比较简单，核心在于GetVarint32Ptr 实现。

```
inline const char* GetVarint32Ptr(const char* p, const char* limit,
                                  uint32_t* value) {
  if (p < limit) { // < 128
    uint32_t result = *(reinterpret_cast<const uint8_t*>(p));
    if ((result & 128) == 0) {
      *value = result;
      return p + 1;
    }
  }
  return GetVarint32PtrFallback(p, limit, value);
}
```

在该函数中，分为两块：

- ​	小于128的数

直接取出来放进value中，并返回下一个字节。

- 大于等于128的数

还是以上述8645二进制表示，我们编码后的p为：11 000101 010000 1，第一次for循环会进入下面第一个if，此时得1000011，第二次for循环会进入else，else里面的代码等价于1000101 + ((100011) << 7)，得到还原后的8645二进制表示：1000101 1000101

```
// 11000101 01000011
const char* GetVarint32PtrFallback(const char* p, const char* limit,
                                   uint32_t* value) {
  uint32_t result = 0;
  for (uint32_t shift = 0; shift <= 28 && p < limit; shift += 7) {
    uint32_t byte = *(reinterpret_cast<const uint8_t*>(p));
    p++;
    if (byte & 128) {
      // More bytes are present
      result |= ((byte & 127) << shift); // 1000011
    } else {
      result |= (byte << shift); // 1000101 + ((1000011) << 7) = 1000101 1000101
      *value = result;
      return reinterpret_cast<const char*>(p);
    }
  }
  return nullptr;
}
```

其中for循环有个细节就是28，这个是对32位，最多5字节，首次1字节，还需要4次左移操作，因此是4 * 7 = 28，同理对于64位处理时便是 9 * 7 = 63。

对于64位没有fallback函数，而是直接集成在GetVarint64Ptr中：

```
const char* GetVarint64Ptr(const char* p, const char* limit, uint64_t* value) {
  uint64_t result = 0;
  for (uint32_t shift = 0; shift <= 63 && p < limit; shift += 7) {
    uint64_t byte = *(reinterpret_cast<const uint8_t*>(p));
    p++;
    if (byte & 128) {
      // More bytes are present
      result |= ((byte & 127) << shift);
    } else {
      result |= (byte << shift);
      *value = result;
      return reinterpret_cast<const char*>(p);
    }
  }
  return nullptr;
}
```

其余实现同32位：

```
bool GetVarint64(Slice* input, uint64_t* value) {
  const char* p = input->data();
  const char* limit = p + input->size();
  const char* q = GetVarint64Ptr(p, limit, value);
  if (q == nullptr) {
    return false;
  } else {
    *input = Slice(q, limit - q);
    return true;
  }
}
```

##### 3.Put

Put操作就是给一个整数和字符串指针，将整数进行varint编码，并存入字符串中。

```
void PutVarint32(std::string* dst, uint32_t v) {
  char buf[5];
  char* ptr = EncodeVarint32(buf, v);
  dst->append(buf, ptr - buf);
}

void PutVarint64(std::string* dst, uint64_t v) {
  char buf[10];
  char* ptr = EncodeVarint64(buf, v);
  dst->append(buf, ptr - buf);
}
```

##### 4.PrefixedSlice

PrefixedSlice分为三个函数，分别为PutLengthPrefixedSlice、GetLengthPrefixedSlice。Put的作用是先对传入的Slice大小进行编码，再在后面追加Slice内容，一起存入传入的字符串中。例如：当传入的切片是abcd，那么最后结果就是：4abcde，4表示的是编码后的结果，由于不到128，只需要1个字节即可，还是原值。

Get的作用是从输入的切片，去掉编码的切片大小，返回切片内容。bool返回是否获取成功。

```
void PutLengthPrefixedSlice(std::string* dst, const Slice& value) {
  PutVarint32(dst, value.size());
  dst->append(value.data(), value.size());
}

bool GetLengthPrefixedSlice(Slice* input, Slice* result) {
  uint32_t len;
  if (GetVarint32(input, &len) && input->size() >= len) {
    *result = Slice(input->data(), len);
    input->remove_prefix(len);
    return true;
  } else {
    return false;
  }
}
```

##### 5.len

这个函数就比较简单了，根据传入的value，返回实际编码后的长度即可。

```
int VarintLength(uint64_t v) {
  int len = 1;
  while (v >= 128) {
    v >>= 7;
    len++;
  }
  return len;
}
```

至此，完毕。

下面补充一个Varint测试（来自网络）：

```
void TestVarintCode() { 
	uint32_t v = 8645; 
	int len = VarintLength(v); 
	cout << "len = " << len << endl; 
	string var; 
	PutVarint32(&var, v); 
	char* dst = const_cast<char*>(var.c_str());
	const int num = 8; 
	char* ptr = reinterpret_cast<char*>(dst); 
	while (len--) { 
		auto x = (bitset<num>)(*ptr++); 
		cout << x << " "; 
		if (x.to_ulong() < 128) { 
			cout << x.to_ulong() << ":" << (char)x.to_ulong() << endl; 
		} else { 
			cout << x.to_ulong() << ":" << "not found char" << endl; 
		} 
	}
	Slice s(dst); 
	cout << "size = " << s.size() << endl; 
	uint32_t rev_v; 
	cout << GetVarint32(&s, &rev_v) << endl; // 1 
	cout << rev_v << endl; // 8768
	
	string ps; 
	PutLengthPrefixedSlice(&ps, 				Slice("abcdefabcdefabcdefabcdefabcdefabcdefabcdefabcdefabcdefabcdefabcdefabcdefa bcdefabcdefabcdefabcdefabcdefabcdefabcdefabcdefabcdefabcdefabcdefabcdefabcdefabc def"));
    cout << "ps.data=" << ps.data() << ", ps.size=" << ps.size() << endl; 
	Slice p(ps), res; 
	const char *limit = p.data() + p.size(); 
	const char* xv = GetLengthPrefixedSlice(p.data(), limit, &res); 
	cout << "res.data=" << res.data() << ", res.size=" << res.size() << endl; 
	Slice input(ps), vres; 
	bool ret = GetLengthPrefixedSlice(&input, &vres); 
	cout << "vres.data=" << vres.data() << ", vres.size=" << vres.size() << endl; 
	cout << ret << endl; 
}
```

