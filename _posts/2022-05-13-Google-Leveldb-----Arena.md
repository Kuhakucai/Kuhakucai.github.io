---
layout:     post
title:      "Google Leveldb --- Arena"
subtitle:   " \"学习谷歌开源项目Leveldb的一些笔记\""
date:       2022-05-13 12:00:00
author:     "Puppetsho"
header-img: "img/post-bg-infinity.jpg"
catalog: true
tags:
    - 笔记
    - 分享
    - Google Leveldb


---

> “不适合人类阅读，非常水的自我笔记. ”

源码相关文件<util/arena.h> <util/arena.cc>

#### 1. 什么是Arena？

​		Arena是leveldb的一个内存池，申请内存时，将申请的内存块放入动态数组中，在Arena生命周期结束，循环该动态数组，将其释放掉，其内部结构如下图所示。

![Snipaste_2022-05-20_15-16-16.png](https://s2.loli.net/2022/05/20/nSgXFmtxVlf65AR.png)



#### 2. 基本结构

​		这里主要分析Arena的基本数据结构。提一下析构的时候，是循环动态数组释放内存。

​		动态数组是blocks_，此外还用几个成员变量表示当前块的可用内存及指向位置。

​		alloc_ptr_ 表示当前块的偏移位置，alloc_bytes_remaining_ 表示当前块剩余字节，memory_usage_ 表示整块blocks_ 所占用 的内存，包括分配的块与指向块的指针。

```
class Arena {
public:
	Arena();
	Arena(const Arena&) = delete;
	Arena& operator=(const Arena&) = delete;
	~Arena();
private:
	char* alloc_ptr_;
	size_t alloc_bytes_remaining_; 
	std::vector<char*> blocks_; 
	std::atomic<size_t> memory_usage_; 
};

Arena::Arena() 
	: alloc_ptr_(nullptr), alloc_bytes_remaining_(0), memory_usage_(0) {} 
	
Arena::~Arena() { 
	for (size_t i = 0; i < blocks_.size(); i++) { 
	delete[] blocks_[i]; 
	} 
}
```



#### 3. Arena接口

​		首先是public成员。

```
  // Return a pointer to a newly allocated memory block of "bytes" bytes.
  char* Allocate(size_t bytes);

  // Allocate memory with the normal alignment guarantees provided by malloc.
  char* AllocateAligned(size_t bytes);

  // Returns an estimate of the total memory usage of data allocated
  // by the arena.
  size_t MemoryUsage() const {
    return memory_usage_.load(std::memory_order_relaxed);
  }
```

##### (1) Allocate

Allocate表示根据传入的bytes分配相应的内存。

分配时有如下三个规则：

- 如果bytes小于当前块剩余内存大小，直接分配，并调整alloc_ptr_ 与 alloc_bytes_remaining_
- 如果bytes大于当前块剩余内存大小，且小于1024（kBlockSize / 4），则申请一个新的块（block），插入到block_ 中，将alloc_ptr_ 指向新块，调整alloc_bytes_remaining_ 为kBlockSizebytes.	
- 如果bytes大于1024（kBlockSize / 4），那么直接分配一个新的块（block），并插入blocks中，此时alloc_ptr 与 alloc_bytes_remainning_ 不做调整。	

可以看到第一个规则如下代码的if语句里面实现，其他规则调用了内部的AllocateFallback函数，我们继续跟踪。

```
inline char* Arena::Allocate(size_t bytes) {
  // The semantics of what to return are a bit messy if we allow
  // 0-byte allocations, so we disallow them here (we don't need
  // them for our internal use).
  assert(bytes > 0);
  // 第一个规则
  if (bytes <= alloc_bytes_remaining_) {
    char* result = alloc_ptr_;
    alloc_ptr_ += bytes;
    alloc_bytes_remaining_ -= bytes;
    return result;
  }
  // 其他规则
  return AllocateFallback(bytes);
}
```

此时可以看到第三个规则为if语句实现，第二个规则为if之后的实现。

```
static const int kBlockSize = 4096;
char* Arena::AllocateFallback(size_t bytes) {
  // 第三个规则
  if (bytes > kBlockSize / 4) {
    // Object is more than a quarter of our block size.  Allocate it separately
    // to avoid wasting too much space in leftover bytes.
    char* result = AllocateNewBlock(bytes);
    return result;
  }
  //第二个规则
  // We waste the remaining space in the current block.
  alloc_ptr_ = AllocateNewBlock(kBlockSize);
  alloc_bytes_remaining_ = kBlockSize;

  char* result = alloc_ptr_;
  alloc_ptr_ += bytes;
  alloc_bytes_remaining_ -= bytes;
  return result;
}
```

​	可以看到在从第二个规则和第三个规则发生时，会浪费掉当前块的内存（没有使用当前块剩余内存），并且浪费的块内存不会超过kBlockSize / 4。这里是kBlockSize为4096。

​	在第三个规则中又调用了私有成员AllocateNewBlock，此时作用是分配一个新的block。

​	在分配新块中，要做几个操作：

- 申请block_bytes大小的内存
- 将申请好的块放进block中
- 当前内存总共使用了 之前的内存 + 当前分配内存块大小 + 指向当前内存块指针大小 

```
char* Arena::AllocateNewBlock(size_t block_bytes) {
  char* result = new char[block_bytes];
  blocks_.push_back(result);
  memory_usage_.fetch_add(block_bytes + sizeof(char*),
                          std::memory_order_relaxed);
  return result;
}
```

​	至此，一个完整的内存分配流程就搞定了，同理，我们来看一下Arena按对齐方式分配内存。

##### （2) AllocateAligned

​	首先是对齐大小，最多为8字节，这里断言有点不一样，判断当前对齐是不是 x^2，可以使用x & (x - 1)。

```
  const int align = (sizeof(void*) > 8) ? sizeof(void*) : 8;
  static_assert((align & (align - 1)) == 0,
                "Pointer size should be a power of 2");
```

​	随后，计算alloc_ptr_ % align，当前分配内存 % 对齐字节数。

​	当前分配内存计算是采用指针转整数计算，实现方式是：

```
reinterpret_cast<uintptr_t>(alloc_ptr_);
```

​	而取模的计算方式是采用 x & (y - 1)方式，对应 x % y。

```
size_t current_mod = reinterpret_cast<uintptr_t>(alloc_ptr_) & (align - 1);
```

​	随后计算了分配当前内存所需要填充的字节slop：

```
size_t slop = (current_mod == 0 ? 0 : align - current_mod);
```

​	并计算总共分配的字节：

```
size_t needed = bytes + slop
```

​	如果所需要分配的总字节数不超过当前块剩余字节大小，直接分配即可，并调整当前块的偏移及当前块剩余字节大小，对应Allocate规则1。

​	否则，按照Allocate的规则2、3进行分配即可。

```
char* result; 
if (needed <= alloc_bytes_remaining_) { 
	result = alloc_ptr_ + slop; 
	alloc_ptr_ += needed; 
	alloc_bytes_remaining_ -= needed; 
} else { 
	// AllocateFallback always returned aligned memory 
	result = AllocateFallback(bytes); 
}
assert((reinterpret_cast<uintptr_t>(result) & (align - 1)) == 0);
```

​	为了清楚的理解这一块内存分配，我们以在当前块分配6字节内存为例，假设之前已经分配好了5字节，此时当前块状态对应于左图，字节对齐以8字节为例，那么在分配6字节内存时，currend_mod、slop、needed如右图所示，此时由于可以在当前块内存，直接进入if语句，并进行分配操作，调整alloc_ptr_ ，并计算alloc_bytes_remaining_ 。

![Snipaste_2022-05-20_17-20-48.png](https://s2.loli.net/2022/05/20/3b6d7UBQPepVmYZ.png)

##### （3）MemoryUsage

​		直接返回memory_usage_，这个没什么好说的了。

​		至此，Arena内存池讲完了，接下来给出两个测试用例，详细的对上诉三个规则在不按字节对齐分配与按对齐方式分配内存情况进行测试。

