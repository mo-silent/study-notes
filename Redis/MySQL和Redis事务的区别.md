# MySQL 和 Redis 事务的区别

redis 事务不支持原子性和持久性，并且 mysql 事务与 redis 事务的实现原理不同

## 事务的四大特性

ACID，事务的四大特性缩写：原子性 (Atomicity)、一致性 (Consistency)、隔离性 (Isolaction)、持久性 (Durability)

**原子性**

一个事务内所有操作就是最小操作单元，要么全部成功，要么全部失败

**一致性**

应用系统从一个正确的状态到另一个正确的状态

一个事务可以封装状态改变 (除非它是一个只读的)，事务必须保证系统处于一致状态，不管在任何给定的时间并发事务有多少
 
数据库的 "一致性" 从底层来说，是一组约束。这组约束可以是约束条件、可以是触发器等，也可以是它们的组合。从更高的层面来说，"一致性" 是一种目的，即保持数据库与真实世界之间的正确映射。此时，需要靠各种锁来达成 "一致性"

**隔离性**

事务可以有多个原子并发执行，但事务之间是互不干扰的；一个事务所做的修改在最终提交之前，对其他事务是不可见的

**持久性**

当一个事物提交之后，数据库状态永远的发生了改变，即这个事物只要提交了，哪怕提交后宕机，他也确确实实的提交了，不会出现因为刚刚宕机了而让提交不生效，是要事物提交，他就像洗不掉的纹身，永远的固化了，除非你毁了硬盘


## 命令行执行事务的命令

**Mysql**

```bash
Begin: 显示开启一个事务
Commit: 提交事务，将数据库进行所有修改变成永久性
Rollback: 结束事务，并撤销现在正在进行的为提交修改
```

mysql 默认会开启一个事务，且缺省设置自动提交，即每成功执行sql，一个事务就会马上 commit，不能 rollback

用Begin、Rollback、commit显式开启并控制一个 新的 Transaction

执行命令 set autocommit=0，用来禁止当前会话自动commit，控制 默认开启的事务

**Redis**

```bash
Multi：标记事务的开始
Exec：执行事务的 commands 队列
Discard：结束事务，并清除 commands 队列
```

redis 默认不开启事务，即command会立即执行，而不会排队，并不支持rollback

用multi、exec、discard，显式开启并控制一个Transaction

## 实现原理

**Mysql**

mysql 是基于 undo/redo 日志实现事务
- undo 记录修改前状态，rollback 基于 undo 日志实现
- redo 记录修改后状态，commit 基于 redo 日志实现

**Redis**

redis 是基于 commands 队列实现事务
- 没有开启事务，command将会被立即执行并返回执行结果，并且直接写入磁盘
- 开启事务，command不会被立即执行，而是排入队列，并返回排队状态（具体依赖于客户端（例如：spring-data-redis）自身实现）
  
调用 `exec` 从会执行 commands 队列

Redis 事务并不支持 Rollback，redis 命令在事务执行时失败，仍会继续执行剩下命令而不是 Rollback (事务回滚)

Redis 事务中的两大命令错误：
- 一个命令可能排队失败，所以在 `EXEC` 调用之前可能会出错。例如，该命令可能在语法上是错误的（参数数量错误，命令名称错误，...），或者可能存在一些紧急情况，例如内存不足情况（如果使用该 maxmemory 指令将服务器配置为具有内存限制）
- 调用后 命令可能会失败 `EXEC`，例如，因为我们对具有错误值的键执行了操作（例如对字符串值调用列表操作）

从 Redis 2.6.5 开始，服务器会在命令累积过程中检测到错误。然后它将拒绝执行在 期间返回错误 EXEC 的事务，丢弃该事务

> Redis < 2.6.5 的注意事项：在 Redis 2.6.5 之前，客户端需要 `EXEC` 通过检查排队命令的返回值来检测之前发生的错误：如果命令以 QUEUED 回复，则表示正确排队，否则 Redis 返回错误。如果在排队命令时出现错误，大多数客户端将中止并丢弃事务。否则，如果客户端选择继续事务，则该 `EXEC` 命令将成功执行所有排队的命令，而不管先前的错误如何

## Go 语言开启事务代码

**Mysql**

这里使用 [GORM](https://gorm.io/zh_CN/docs/) 的来执行 MySQL 事务

`Transaction` 封装了事务的 `Begin`，`Rollback` 和 `Commit`

有兴趣的可以看一下 [源码](https://github.com/go-gorm/gorm/blob/v1.23.4/finisher_api.go#L543) 和 [Transaction API](https://pkg.go.dev/gorm.io/gorm#DB.Transaction)

```go

// 安装
go get -u gorm.io/gorm
go get -u gorm.io/driver/sqlite

// 封装的事务
db.Transaction(func(tx *gorm.DB) error {
  // 在事务中执行一些 db 操作（从这里开始，应该使用 'tx' 而不是 'db'）
  if err := tx.Create(&Animal{Name: "Giraffe"}).Error; err != nil {
    // 返回任何错误都会回滚事务
    return err
  }

  if err := tx.Create(&Animal{Name: "Lion"}).Error; err != nil {
    return err
  }

  // 返回 nil 提交事务
  return nil
})

// 手动事务
// 开始事务
tx := db.Begin()

// 在事务中执行一些 db 操作（从这里开始，应该使用 'tx' 而不是 'db'）
tx.Create(...)

// ...

// 遇到错误时回滚事务
tx.Rollback()

// 否则，提交事务
tx.Commit()
```


**Redis**

这里使用 [go-redis](https://redis.uptrace.dev/guide/#why-go-redis) 来执行 Redis 事务 

`TxPipelined` 封装了 MULTI 和 EXEC 命令

[TxPipelined源码](https://github.com/go-redis/redis/blob/v8.11.5/redis.go#L633) 和 [TxPipelined API](https://pkg.go.dev/github.com/go-redis/redis/v8#Client.TxPipelined)

```go
// 安装
go get github.com/go-redis/redis/v8

// 封装事务 TxPipelined
cmds, err := rdb.TxPipelined(ctx, func(pipe redis.Pipeliner) error {
	for i := 0; i < 100; i++ {
		pipe.Get(ctx, fmt.Sprintf("key%d", i))
	}
    _, err := pipe.Exec(ctx)
    if err!=nil {
        //取消提交
        pipe.Discard()
    }
	return nil
})
// MULTI
// GET key0
// GET key1
// ...
// GET key99
// EXEC

if err != nil {
	panic(err)
}

for _, cmd := range cmds {
    fmt.Println(cmd.(*redis.StringCmd).Val())
}

// 手动事务 TxPipeline
pipe := rdb.TxPipeline()
defer pipe.Close()

incr := pipe.Incr(ctx, "tx_pipeline_counter")
pipe.Expire(ctx, "tx_pipeline_counter", time.Hour)

// Execute
//
//     MULTI
//     INCR pipeline_counter
//     EXPIRE pipeline_counts 3600
//     EXEC
//
// using one rdb-server roundtrip.
_, err = pipe.Exec(ctx)
if err != nil {
    //取消提交
    pipe.Discard()
}
fmt.Println(incr.Val(), err)
```