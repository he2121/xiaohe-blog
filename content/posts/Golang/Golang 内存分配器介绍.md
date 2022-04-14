---
title: "Golang 内存分配器介绍"
author: "小贺"
date: 2022-04-10T11:10:12+08:00
tags: ["计算机基础","golang"]
---

内存管理属于计算机中的基石，在各种操作系统以及编程语言的设计中，最重要考量之一在于内存管理部分。而内存管理中可以分成两个主要部分：**内存分配**与**内存回收**。今天我们主要来分析下 Golang 内存分配器的设计，也不可避免对内存回收有所提及

## 基础概念

此节简单介绍下与此文相关的一些概念

### 内存布局

用户程序所看到内存是操作系统为其分配的虚拟内存，其内存布局可分为以下五个部分：

![memory-layout](https://ganghuan.oss-cn-shenzhen.aliyuncs.com/img/2021-02-28-16145096642893-memory-layout-2022-04-07.png)

- 栈区（Stack）：存储程序执行期间的本地变量和函数的参数，从高地址向低地址生长
- 堆区（Heap）：动态内存分配区域，通过 `malloc`、`new`、`free` 和 `delete` 等函数管理
- 未初始化变量区（BSS）：存储未被初始化的全局变量和静态变量
- 数据区（Data）：存储在源代码中有预定义值的全局变量和静态变量
- 代码区（Text）：存储只读的程序执行代码，即机器指令

其中 BSS，Data，Text 区内存占用不会变化（分配和回收），也可称为静态内存。栈区内存会由编译器自动分配释放，而堆区上的内存需要应用手动申请与释放。所以虽然上述内存布局中存在有 5 个部分，但当我们谈及内存管理时，都是指的堆内存

### 堆内存管理方式

#### 手动管理

C/C++ 系列的 malloc/free 的手动管理

优点：性能比较好

缺点

- 对程序员心智压力大，容易出 bug

  - 内存泄漏：垃圾对象占用堆内存没有被释放

  - 悬挂指针：指向被释放的内存区域

### 自动管理

JAVA、Golang、 Python 等语言。主要存在两种自动回收方式：垃圾回收、引用计数

优点

- 对程序员友好

缺点：性能有所损耗

### 内存分配

内存分配主要有两种方式：**线性分配器**与**空闲链表分配器**，其它任何不同的内存分配器都是这两者方式的变种。

#### 线性分配器

![Introduction-to-Golang/图解Go内存分配器.md at main · 0voice/Introduction-to-GolangGitHub](https://ganghuan.oss-cn-shenzhen.aliyuncs.com/img/68747470733a2f2f63646e2e6e6c61726b2e636f6d2f79757175652f302f323032302f706e672f3130363934372f313538323235323631373633362d66616638333034302d626334632d346165312d623233652d3262323464613036323030382e706e67-2022-04-07.png)

线性分配器易产生内存碎片，所以使用这种分配器时要搭配合适的内存回收算法使用，如 java 中的标记-压缩，复制回收等算法

#### 空闲链表分配器

把内存分为一个个块，然后利用链表连接不同的内存块

![free-list-allocator](https://ganghuan.oss-cn-shenzhen.aliyuncs.com/img/2021-02-28-16145096642935-free-list-allocator-2022-04-07.png)

这种方式可以重新利用回收的内存块，但是分配内存时需要**遍历链表**，时间复杂度 O(n)。

一般有以下几种方式在不同大小的内存块选择分配

1. 首次适应：从表头遍历，第一个大于申请内存的内存块
2. 循环首次适应：从上次遍历结束位置开始遍历，第一个大于申请内存的内存块
3. 最优适应：选择大小最合适的内存块
4. 最差适用：选择最大内存块

## Golang 内存分配器

### 整体框架

Golang 内存分配算法源自 Google 的 `TCMalloc 算法（Thread-Caching）`，具有减少内存碎片，适用于多核，更好的并行性支持等特性，主要利用了**多级缓存**与**大小分类**的思想来优化内存分配速度。TCMalloc 整体框架如下。

![TCMalloc解密（一） - 知乎](https://ganghuan.oss-cn-shenzhen.aliyuncs.com/img/v2-af5ba79341db113855678eb11507dee5_1440w-2022-04-07.jpg)

#### 多级缓存设计

可以简单认为分别是 ThreadCache、CentralCache、PageHeap 这三级

1. **ThreadCache:** 对于每个线程 TCMalloc 都为其分配了一份单独的缓存，因此从 TheadCache 取内存与回收内存不需要加锁。
2. **CentralCache:** 所有线程公用的缓存，供各线程的ThreadCache从中取用空闲内存，因此需要加锁
3. **PageHeap:** 处理向 OS 申请内存相关的操作，并提供了一层缓存，整个可供应用程序动态分配的内存的抽象

### 内存管理基本单元

**`page`**: 类似于操作系统的内存管理，TCMalloc 将整个虚拟内存空间划分为 n 个同等大小的 **Page**，每个 page 默认 8 KB。

**`span`**: 将连续的 n 个 page 称为一个 **Span**, 是 TCMalloc 内存管理的基本单元，多级缓存直接持有的都是 span 链表，其实就是内存块。

![在这里插入图片描述](https://ganghuan.oss-cn-shenzhen.aliyuncs.com/img/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQzMTg4NzQ0,size_16,color_FFFFFF,t_70-2022-04-07.png)

**`object`**：申请内存的是 `object`，因此 span 分割成一个个 `object`(逻辑上) ，实际上分配内存是根据 `object`的大小去寻找合适大小 有空闲位置的 span

![img](https://ganghuan.oss-cn-shenzhen.aliyuncs.com/img/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQzMTg4NzQ0,size_16,color_FFFFFF,t_70-20220407203048702-2022-04-07.png)

**`size class`**: 根据对象大小预定义的一些 span 种类，并且在其中分割成等大的区域来存对象（不同大小的内存块）

### Golang 分配器的具体实现

### Golang 内存布局（1.10 及以前）

![img](https://img-blog.csdnimg.cn/20210406192927464.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQzMTg4NzQ0,size_16,color_FFFFFF,t_70)

- `Arena`: 真正的堆区

- `bitmap`: 标识堆区上的每一个指针大小的区域是否回收、是否是指针，所以使用 2 bit 代表 8 byte 长的区域。用于垃圾回收

- `spans`:  保存 span 的指针，arena 区域中每一页都指向的 span。8 byte 代表一页（8 KB）

后来为了解决 C 与 Go 混合使用可能会导致程序崩溃的问题，Golang 1.11 后使用了稀疏内存布局，把虚拟内存分割成一个个离散的堆区（64 MB），但其基本思想没有变化，只是内存管理部分逻辑变得更加复杂，在此我们可以简单认为内存布局还是如上图所述。

### Golang 内存组件

#### `runtime.mspan`

- 对应 TCMalloc 中 Span 的实现

```go
type mspan struct {
  next *mspan     // 组成的 span 双向链表
	prev *mspan     
  
	startAddr uintptr // 起始地址
	npages    uintptr // 页数
  
	freeindex uintptr	// 空闲对象的初始索引
	allocBits  *gcBits // 划分的对象区域分配情况 bitmap
	gcmarkBits *gcBits // 划分的对像 GC 回收情况 bitmap
  
  spanclass   spanClass     // 这个 span 的所属种类（大小分级）
	...
}
```

`spanClass`: 对应 TCMalloc 中的 sizeClass，实现其实就是一个 uint8, 代表 span 按照对象大小划分出不同种类

```go
type spanClass uint8
```

如下图所示，Golang 中按对象大小分级总共有 67 种 span，运行时中还包含 spanClass ID 为 0 的特殊 span，它用于大于 32KB 的大对象的分配过程

```
// runtime.sizeclasses.go
// class  bytes/obj  bytes/span  objects  tail waste  max waste  min align
//     1          8        8192     1024           0     87.50%          8
//     2         16        8192      512           0     43.75%         16
//     3         24        8192      341           8     29.24%          8
//     4         32        8192      256           0     21.88%         32
// ....
//    65      27264       81920        3         128     10.00%        128
//    66      28672       57344        2           0      4.91%       4096
//    67      32768       32768        1           0     12.50%       8192
```

此外，为了垃圾回收的性能，spanClass 这个 uint8 使用了最后的 1 个 bit 位来区分存放带指针（scan）与不带指针（noscan）的对象。所以总的来说 span 总共有 68*2 种

```go
func (sc spanClass) sizeclass() int8 {
	return int8(sc >> 1)
}

func (sc spanClass) noscan() bool {
	return sc&1 != 0
}
```

#### `runtime.mcache`

```
type mcache struct {
 	alloc [numSpanClasses]*mspan // 持有 (68 * 2) 种 span 缓存
 	// ...
}
```

golang 中 mcache 是属于 P 的，如下图所示。

![Golang内存分配| Random walk to my blog](https://ganghuan.oss-cn-shenzhen.aliyuncs.com/img/mcache_alloc-2022-04-12.png)

#### `runtime.mcentral`

`mcache` 作为线程的私有资源为单个线程服务，而 central 则是全局资源，为多个线程服务。它会同时持有两个 `runtime.spanSet`，分别存储包含空闲对象和不包含空闲对象的内存管理单元。其中 partial 和 full 又分别有两个 spanSet，一个是被 GC 清理过的，一个是未被 GC 清理过的。

```go
type mcentral struct {
 spanclass spanClass	// 68 * 2 种 span
 partial [2]spanSet // 包含空闲对象的 span list，spanSet 支持安全并发操作
 full    [2]spanSet // 不包含空闲对象的 span list
}
```

`mcache` 会通过 `runtime.mcentral.cacheSpan` 方法从 `mcentral` 获取可用的 span，主要分为以下几个部分

1. 从 partial 并且清理过 spanlist 的获取可用 span
2. 从 partial 并且没有被清理过的 spanlist 获取可用 span
3. 从 full 并且未被清理过的 spanlist，获取 span 并且清理，获取可用 span
4. `mcentral` 从 `mheap`中申请内存

#### `runtime.mheap`

上述介绍的 `mcentral` 结构可知，一个 `mcentral` 实例只代表一种 span，实际上会有 68*2 种 span。因此需要一个长度为 68 *2 的 `mcentral` 数组, 而这个结构位于 `mheap`

```go
type mheap struct {
	lock  mutex
  
  allspans []*mspan 					// 所有的 span
  arenas [1 << arenaL1Bits]*[1 << arenaL2Bits]*heapArena // 离散的堆区
  
  central [numSpanClasses]struct {		// 持有的 68*2 种 mentral
		mcentral mcentral
		pad      [cpu.CacheLinePadSize - unsafe.Sizeof(mcentral{})%cpu.CacheLinePadSize]byte
	}
  // ...
}

// 离散的 heapArena 每个 64 MB
type heapArena struct {
	bitmap [heapArenaBitmapBytes]byte
	spans [pagesPerArena]*mspan // 页到 mspan 的映射
	pageInUse [pagesPerArena / 8]uint8 // bitmap 标记哪些页在使用中
}
```

Go 程序就是通过一个 `mheap` 类型的全局变量来进行内存管理的，其上向操作系统申请内存，下向 `mcentral` 提供 span

### 内存分配过程

1. 堆上分配的所有对象都通过 `runtime.newobject` 分配，该函数会调用 `runtime.mallocgc` 分配内存

```go
func newobject(typ *_type) unsafe.Pointer {
	return mallocgc(typ.size, typ, true)
}
```

2. Golang 对待分配对象大小不同有不同的分配逻辑，这段逻辑在 `runtime.mallocgc` 中
   1. tiny 对象：小于 16 B 且不包括指针的对象
   2. 大对象：大于 32 KB 的对象
   3. 正常对象： (0, 16 B) 包含指针的对象，[16 B, 32 KB] 的对象

```go
func mallocgc(size uintptr, typ *_type, needzero bool) unsafe.Pointer {
		// ....
  	// 获取当前线程的 mcache
		c := getMCache()
		var span *mspan
		var x unsafe.Pointer
		noscan := typ == nil || typ.ptrdata == 0
		if size <= maxSmallSize {
			if noscan && size < maxTinySize {
      	// Tiny 分配
      } else {
        // 正常对象分配
      }
    }else{
      	// 大对象分配
    }
		return x
}
```

3. 微对象与大对象都属于内存管理的优化手段，大对象所需要连续空间比较多，而且相对比较少见，直接从 `mheap` 申请。tiny 对象则是因为 sizeclass 中分割最小 object 是 8 B，对 bool，int8，int32 这种分配比较浪费内存空间，把这些挤在一起分配在一个 object 中。下面我们重点分析下正常对象的分配过程

```go
			var sizeclass uint8
			// 根据待分配对象确定 sizeclass
			if size <= smallSizeMax-8 {
				sizeclass = size_to_class8[divRoundUp(size, smallSizeDiv)]
			} else {
				sizeclass = size_to_class128[divRoundUp(size-smallSizeMax, largeSizeDiv)]
			}
			size = uintptr(class_to_size[sizeclass])
			spc := makeSpanClass(sizeclass, noscan)
			span = c.alloc[spc]
			// 在 mcache 中 对应 sizeclass 的 span 链表中获取可用的内存空间
			v := nextFreeFast(span)
			// mcache 中没有可用的，向上 mcentral 申请，如果 mcentral 也没有可用空间，继续向上 mheap 申请
			if v == 0 {
				v, span, shouldhelpgc = c.nextFree(spc)
			}
			x = unsafe.Pointer(v)
			// 把申请的这块空间清空
			if needzero && span.needzero != 0 {
				memclrNoHeapPointers(unsafe.Pointer(v), size)
			}
```
1. 获取当前线程的私有缓存 `mcache`
1. 根据分配对象的大小确定是 tiny 对象、大对象还是正常对象分配模式
1. 如果是正常对象分配，根据待分配对象的大小确定跨度类 `runtime.spanClass` 的ID
1. 在 `mcache` 找到对应 sizeclass 的 span list 寻找空闲的内存块
1. 如果在 `mcache` 没有可用的内存块，从 `mcentral` 申请一个可用 `span`, 如果 `mcentral` 也没有，则像 `mheap` 申请 `span`。当然，`mheap`也没有会继续向操作系统申请

## 总结

- Golang 内存分配器基于谷歌 TCMolloc 来实现的，其实质上是空闲链表分配的变种，主要优化思想是多级缓存池与大小分级
- Golang 内存管理将堆区分成页作为最小管理单位，每页 8 KB
- 连续页组成大小不同的内存块，即 `span`，`span` 是内存分配的基本单位
- `span` 根据分割对象大小以及内存页数预定义了 68 * 2 种 sizeclass 的 span，根据待分配对象大小去选择合适的种类的 span，这就是大小分级。 
- `mcache`，`mcentral`,`mheap` 作为三层内存缓冲区，也极大提高了内存分配的效率

## 参考

1. https://draveness.me/golang/docs/part3-runtime/ch07-memory/golang-memory-allocator/
1. https://zhuanlan.zhihu.com/p/323915446
1. https://juejin.cn/post/6844903795739082760
1. https://tech.bytedance.net/articles/6948356640802340877
1. https://blog.csdn.net/qq_43188744/article/details/115433514
1. https://blog.csdn.net/qq_43188744/article/details/115355566

