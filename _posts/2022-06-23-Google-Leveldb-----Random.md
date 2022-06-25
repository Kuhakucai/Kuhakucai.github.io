---
layout:     post
title:      "Google Leveldb --- Random"
subtitle:   " \"学习谷歌开源项目Leveldb的一些笔记\""
date:       2022-06-23 12:00:00
author:     "Puppetsho"
header-img: "img/post-bg-halting.jpg"
catalog: true
tags:
    - 笔记
    - 分享
    - Google Leveldb


---

> “不适合人类阅读，非常水的自我笔记.”

#### 1.随机数生成算法

------

A、B、M都是常数（一般取质数）。

```
seed = （seed * A + B） % M 
```

当B = 0时，叫做乘同余法。B  != 0 时，叫做线性同余法。

再random中的实现均是采用 B = 0。

这里提一下，凡是通过一定算法程序生成的都是伪随机数，目前来说只有通过物理现象产生的随机数才是真随机数（如果以后发展到能通过初始状态就能完全预测粒子的最终结果的话，那可能也不能叫做真随机数了，例如刘慈欣在《镜子》里面提到的超弦计算机）

在计算机系统当中，伪随机数都是周期性的，只要持续的次数够多，就可以看到这种周期，而真随机数目前看来并不存在这种周期，这里有一个大佬把随机数得到的结果做成了图片，我们可以很直观地对比一下。

这是真随机数可视化后的图像结果：

![007S8ZIlgy1gjfg65h0djj30e80e8abn.jpg](https://s2.loli.net/2022/06/23/4vW5F7e3TIhXDk2.jpg)



这是伪随机可视化后的图像结果：

![007S8ZIlgy1gjfg752jykj30e80e8myr.jpg](https://s2.loli.net/2022/06/23/YtJEc7hez9nSFNw.jpg)

对比之下，可以明显的看出伪随机是有规律的，体现在图像的纹理当中。

#### 2.构造

内部含有私有成员seed_，使用无符号32位整型存储，其中0x7fffffffu表示 2^31 - 1，也即2147483647。

这里取&目的是保证seed_范围为[0，2147483647]。

如果seed为0或者2147483647L，那么seed_便是1。

```
class Random {
private:
	uint32_t seed_;
public:
	explicit Random(uint32_t s) : seed_(s & 0x7fffffffu) {
	// Avoid bad seeds.
	if (seed_ == 0 || seed_ == 2147483647L) {
		seed_ = 1;
 	}
 }
};
```

#### 3.随机数

生成随机数得到函数是Next，内部实现算法采用C = 0，乘同余法。

prodect % M 等价于 （product >> 31） + （product & M）。

下面来证明一下：

product类型是uint64_t，可以将product的二进制从左到右分解成高33位和低31位，假设高33位的值为H，低31位的值为L，此时product = H << 31 + L = H * 2^31 + L。

代码中：M = 2^31 -1, A = 16807。

由于seed与A均小于M，根据uint64_t product = seed_ * A；则H 小于M。

于是，上诉左式：

```
product%M = (H*2^31+L)%M = ((H*M+H)+L)%M = H+L
```

右式：

```
(product>>31) + (product&M) = (H*2^31 +L)>>31+L = (H*2^31+L)/2^31+L = H+L
```

当低31位的值可能等于M，那么左式等于H，此时右式等于H+M，代码中添加了：

```
if (seed_ > M) { 
	seed_ -= M; 
}
```

因此，此时右式也等于M，综上，左式等价于右式。

另外一个证明点在于注释 ((x << 31) % M) == x ：

```
(x << 31) % M=(x*2^31)%M=(x + x*(2^31-1))%M=(x + x*M)%M=x%M=x;
```

代码实现：

```
  uint32_t Next() {
    static const uint32_t M = 2147483647L;  // 2^31-1
    static const uint64_t A = 16807;        // bits 14, 8, 7, 5, 2, 1, 0
    // We are computing
    //       seed_ = (seed_ * A) % M,    where M = 2^31-1
    //
    // seed_ must not be zero or M, or else all subsequent computed values
    // will be zero or M respectively.  For all other values, seed_ will end
    // up cycling through every number in [1,M-1]
    uint64_t product = seed_ * A;

    // Compute (product % M) using the fact that ((x << 31) % M) == x.
    seed_ = static_cast<uint32_t>((product >> 31) + (product & M));
    // The first reduction may overflow by 1 bit, so we may need to
    // repeat.  mod == M is not possible; using > allows the faster
    // sign-bit-based test.
    if (seed_ > M) {
      seed_ -= M;
    }
    return seed_;
  }
```

#### 4.一些函数

最后是下面三个函数：

- uniform归一化到[0,n-1]。
- OneIn判断uniform后是否为0。
- Skewed返回[0，2m^max_log - 1]。

```
  // Returns a uniformly distributed value in the range [0..n-1]
  // REQUIRES: n > 0
  uint32_t Uniform(int n) { return Next() % n; }

  // Randomly returns true ~"1/n" of the time, and false otherwise.
  // REQUIRES: n > 0
  bool OneIn(int n) { return (Next() % n) == 0; }

  // Skewed: pick "base" uniformly from range [0,max_log] and then
  // return "base" random bits.  The effect is to pick a number in the
  // range [0,2^max_log-1] with exponential bias towards smaller numbers.
  uint32_t Skewed(int max_log) { return Uniform(1 << Uniform(max_log + 1)); }
```

#### 5.测试函数

```
#include "util/random.h" 
#include <iostream> 
using namespace std; 
using namespace leveldb; 
int main() { 
	uint32_t s = 102; 
	Random random(s); 
	cout << random.Next() << endl; 
	cout << random.Next() << endl; 
	cout << random.Uniform(10) << endl; 
	for (int i = 1; i <= 60; i++) { 
		cout << random.OneIn(10) << " "; 
	}
	cout << endl; 
	for (int i = 1; i <= 60; i++) { 
		cout << random.Skewed(3) << " "; // [0, 7] 
	}
	cout << endl; 
	return 0; 
}
```

