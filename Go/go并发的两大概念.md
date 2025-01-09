---
title: Go 并发的两大概念
slug: go-concurrency-two-concepts
categories:
  - Golang
tags:
  - Development language
halo:
  site: https://weblog.silentmo.cn
  name: 93e68ad9-5805-4639-9689-a1ec8e1aa64a
  publish: true
---
# Go 并发 | 通过示例理解数据竞争及竞争条件

Go 并发中有两个重要的概念：数据竞争 (data race) 和竞争条件 (race condition)

在并发程序中，竞争问题可能是程序面临的最难也是最不容易发现的错误之一

## 数据竞争 (data race)

当两个或多个协程同时访问同一个内存地址，并且至少有一个是在写时，就会发生数据竞争，看一下以下例子

```go
i := 0
go func() {
    i++
}()

go func() {
    i++
}()
```

当运行 `go run -race main.go`，会输出下面提示表明发生了数据竞争：

```go
==================
WARNING: DATA RACE
Write at 0x00c00008e000 by goroutine 7:
 main.main.func2()
Previous write at 0x00c00008e000 by goroutine 6:
 main.main.func1()
==================
```

这段代码的 `i` 值是不可预知的，首先 `i++` 是三个操作的组合：
- 从 `i` 中读取值 value
- 将 value 的值 `+1`
- 将值写回到 `i`

其次两个 `goroutine` 是同时进行的，导致会出现以下两个场景

**场景一：Goroutine1 在 Goroutine2 之前运行完成**

Goroutine1     | Goroutine2   | i 值
-------------- | ------------ | ------------
初始值         |               | 0 
读取i的值value | 0             |
将value值+1    | 0             |
将值写回到i     |              | 1
|              | 读取i的值value | 1
|              | 将value值+1   | 1
|              | 将值写回到i    | 2

**场景二：Goroutine1 和 Goroutine2 并发执行**

Goroutine1     | Goroutine2   | i 值
-------------- | ------------ | ------------
初始值         |               | 0 
读取i的值value |               | 0
|              | 读取i的值value | 0
将value值+1   |              | 0
将value值+1   | 0            | 
将值写回到i    |              | 1
|              | 将值写回到i    | 1

这就是数据竞争造成的影响，如果两个协程同时访问同一块内存，并且至少有一个协程写入，就会导致一个不可预期的结果

### 避免数据竞争的发生

避免数据竞争可以使用以下三种方式：
- 使用原子操作
- 使用mutex对同一区域进行互斥操作
- 使用管道 (channel) 进行通信以保证仅且只有一个协程在进行写操作

**方案一：就是让 `i++` 变成原子操作**

众所周知，数据库的事务是一个 IT 人员必备的知识点。事务的四大特性：原子性 (Atomicity)、一致性 (Consistency)、隔离性 (Isolation)、持久性 (Durability)

原子操作意味着 "不可中断的一个或一系列操作"，保证多个线程对同一块内存的操作是串行的，保证数据的唯一性

在 Go 中，可以通过 `atomic` 包执行原子操作，示例：

```go
var i int64
go func() {
    atomic.AddInt64(&i, 1)
}()
go func() {
    atomic.AddInt64(&i, 1)
}()
```

两个协程对 `i` 都是原子性的，一个原子操作不可中断，也就是另一个协程需要等待第一个协程的执行完成后才能对 `i` 操作；不过 `atomic` 包只能操作特定的类型 (如 int32，int64 等整数)，而对于其他数据类型 (如切片，map 以及结构体) 就无法解决

**方案二：同步原语 mutex**

`mutex` 表示互斥，在 Go 中，sync 包提供了 Mutex 类型即互斥锁；互斥锁防止两个线程或协程同时对一个内存地址进行读写操作

数据竞争改为互斥锁的示例：

```go
i := 0
mutex := sync.Mutex{}
go func() {
    mutex.Lock()
    i++
    mutex.Unlock()
}()

go func() {
    mutex.Lock()
    i++
    mutex.Unlock()
}()
```

**方案三：管道 (channel)**

通过管道在多线程进行通信，避免同时读取同一块内存，示例：

```go
i := 0
ch := make(chan int)
go func() {
    ch <- 1
}()

go func() {
    ch <- 1
}()

i += <-ch
i += <-ch
```

每个协程都将增量写入管道中，父协程管理管道并从管道中读取 `i` 进行写操作，只有一个协程对内存地址进行写操作，也就不存在数据竞争

## 竞争条件 (race condition)

看一下示例：

```go
i := 0
mutex := sync.Mutex{}
go func() {
    mutex.Lock()
    defer mutex.Unlock()
    i = 1
}()

go func() {
    mutex.Lock()
    defer mutex.Unlock()
    i = 2
}()
```

这里使用了 `mutex` 避免了数据竞争，但是输出的结果是不确定的。变量 `i` 的结果依赖于协程的执行顺序，可能是 1 也可能是 2。该示例不存在数据竞争，但存在 <font color=red>竞争条件 (race condition)，也称为资源竞争。当程序的行为依赖于执行顺序或事件发生的时机不可控时就会发生竞争条件</font>

保证协程间的执行顺序是协调和编排问题。如果要确保状态从0到1，然后再从1到2，我们就需要找到一种保证协程按序执行的方式。一种方式就是使用管道来解决该问题。此外，如果我们使用了管道进行协调和编排，也可以保证在同一时间只有一个协程在访问公共的部分，这也就意味着可以移除mutex

## 总结

**数据竞争** (data race) 的发生条件是：当多个协程同时访问一个相同内存地址，并且至少有一个在进行写操作时，数据竞争意味着不确定的行为

而不存在数据竞争不代表结果就是确定的。实际上，一个应用程序即使不存在数据竞争，但它的行为肯依赖于不可控的发生时间或执行顺序，这就是**竞争条件 (race condition)**
