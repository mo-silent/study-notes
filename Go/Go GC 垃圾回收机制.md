# Go 语言中的垃圾回收机制 GC 详解

> Author mogd 2022-04-29 
> Update mogd 2022-04-29
> Adage `Be as you wish to seem`

在计算机科学中，垃圾回收 (Garbage Collection 简称 GC) 是一种自动管理内存的机制，垃圾回收器会去尝试回收程序不再使用的对象及占用的内存
> 程序员受益于 GC，无需操心、也不再需要对内存进行手动的申请和释放操作，GC 在程序运行时自动释放残留的内存
> GC 对程序员几乎不可见，仅在程序需要进行特殊优化时，通过提供可调控的 API，对 GC 的运行时机、运行开销进行把控的时候才得以现身

通常，垃圾回收器的执行过程划分为两个半独立的组件：
- 赋值器 (Mutator)：用户态代码，对于 GC 而言，用户态代码仅仅只是在修改对象之间的引用关系
- 回收器 (Colletor)：负责执行垃圾回收的代码







## 参考
[1] [GC 的认识](https://www.bookstack.cn/read/qcrao-Go-Questions/GC-GC.md)
[2] [深入解析 Go](https://www.gitbook.com/book/tiancaiamao/go-internals)