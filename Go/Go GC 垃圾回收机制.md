# Go 语言中的垃圾回收机制 GC 详解

> Author mogd 2022-04-29 
> Update mogd 2022-05-05
> Adage `Be as you wish to seem`

在计算机科学中，垃圾回收 (Garbage Collection 简称 GC) 是一种自动管理内存的机制，垃圾回收器会去尝试回收程序不再使用的对象及占用的内存
> 程序员受益于 GC，无需操心、也不再需要对内存进行手动的申请和释放操作，GC 在程序运行时自动释放残留的内存
> GC 对程序员几乎不可见，仅在程序需要进行特殊优化时，通过提供可调控的 API，对 GC 的运行时机、运行开销进行把控的时候才得以现身

在计算中，内存空间包含两个重要的区域：栈区 (Stack) 和堆区 (Heap)；栈区一般存储了函数调用的参数、返回值以及局部变量，不会产生内存碎片，由编译器管理，无需开发者管理；而堆区会产生内存碎片，在 Go 语言中堆区的对象由内存分配器分配并由垃圾收集器回收

通常，垃圾回收器的执行过程划分为两个半独立的组件：
- 用户程序 (Mutator)：用户态代码，对于 GC 而言，用户态代码仅仅只是在修改对象之间的引用关系
- 收集器 (Colletor)：负责执行垃圾回收的代码

## 一、内存管理和分配

当内存不再使用时，Go 内存管理由其标准库自动执行，即从内存分配到 Go 集合。内存管理一般包含三个不同的组件，分别是用户程序 (Mutator)、分配器 (Allocator) 和收集器 (Collector)，当用户程序申请内存时，它会通过内存分配器申请新内存，而分配器会负责从堆中初始化相应的内存区域

![Go-memory-design](./images/mutator-allocator-collector.png)

### 1.1 内存分配器的分配方法

在编程语言中，内存分配器一般有两种分配方法：
1. 线性分配器 (Sequential Allocator，Bump Allocator)
2. 空闲链表分配器 (Free-List Allocator)  

**线性分配器**

线性分配 (Bump Allocator) 是一种高效的内存分配方法，但是有较大的局限性。当用户使用线性分配器时，只需要在内存中维护一个指向内存特定位置的指针，如果用户程序向分配器申请内存，分配器只需要检查剩余的空闲内存、返回分配的内存区域并修改指针在内存中的位置；

虽然线性分配器有较快的执行速度以及较低的实现复杂度，但线性分配器无法在内存释放后重用内存。如下图，如果已经分配的内存被回收，线性分配器无法重新利用红色的内存

![bump-allocator-reclaim-memory](./images/bump-allocator-reclaim-memory.png)

因此线性分配器需要与适合的垃圾回收算法配合使用
1. 标记压缩 (Mark-Compact)
2. 复制回收 (Copying GC)
3. 分代回收 (Generational GC)
   
以上算法可以通过拷贝的方式整理存活对象的碎片，将空闲内存定期合并，这样就能利用线性分配器的效率提升内存分配器的性能了

**空闲链表分配器**

空闲链表分配器 (Free-List Allocator) 可以重用已经被释放的内存，它在内部会维护一个类似链表的数据结构。当用户程序申请内存时，空闲链表分配器会依次遍历空闲的内存块，找到足够大的内存，然后申请新的资源并修改链表

![free-list-allocator](./images/free-list-allocator.png)

空闲链表分配器常见有四种策略：
- 首次适应 (First-Fit) — 从链表头开始遍历，选择第一个大小大于申请内存的内存块
- 循环首次适应 (Next-Fit) — 从上次遍历的结束位置开始遍历，选择第一个大小大于申请内存的内存块
- 最优适应 (Best-Fit) — 从链表头遍历整个链表，选择最合适的内存块
- 隔离适应 (Segregated-Fit) — 将内存分割成多个链表，每个链表中的内存块大小相同，申请内存时先找到满足条件的链表，再从链表中选择合适的内存块

其中第四中策略与 Go 语言中使用的内存分配策略相似
![segregated-list](./images/segregated-list.png)

该策略会将内存分割成由 4、8、16、32 字节的内存块组成的链表，当我们向内存分配器申请 8 字节的内存时，它会在上图中找到满足条件的空闲内存块并返回。隔离适应的分配策略减少了需要遍历的内存块数量，提高了内存分配的效率

### 1.2 Go 中的内存分配

在 Go 语言中，堆上的所有对象都会通过调用 [runtime.newobject](https://github.com/golang/go/blob/master/src/runtime/malloc.go) 函数分配内存，该函数会调用 [runtime.mallocgc](https://github.com/golang/go/blob/master/src/runtime/malloc.go) 分配指定大小的内存空间，这也是用户程序向堆上申请内存空间的必经函数

```go
func mallocgc(size uintptr, typ *_type, needzero bool) unsafe.Pointer {
	mp := acquirem()
	mp.mallocing = 1

	c := gomcache()
	var x unsafe.Pointer
	noscan := typ == nil || typ.ptrdata == 0
	if size <= maxSmallSize {
		if noscan && size < maxTinySize {
			// 微对象分配
		} else {
			// 小对象分配
		}
	} else {
		// 大对象分配
	}

	publicationBarrier()
	mp.mallocing = 0
	releasem(mp)

	return x
}
```

从代码中可以看出 `runtime.mallocgc` 根据对象的大小执行不同的分配逻辑

## 参考
[1] [GC 的认识](https://www.bookstack.cn/read/qcrao-Go-Questions/GC-GC.md)
[2] [深入解析 Go](https://www.gitbook.com/book/tiancaiamao/go-internals)
[3] [Go 语言设计与实现](https://draveness.me/golang/)
[4] [Go: How Does the Garbage Collector Mark the Memory?](https://medium.com/a-journey-with-go/go-how-does-the-garbage-collector-mark-the-memory-72cfc12c6976)