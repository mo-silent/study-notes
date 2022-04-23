# Golang 中 select 语句死锁问题

一切问题的答案都在 spec^[1]^ 里

## Select 语句执行步骤

[Select_statements](https://go.dev/ref/spec#Select_statements)

> Execution of a "select" statement proceeds in several steps:
> 1. For all the cases in the statement, the channel operands of receive operations and the channel and right-hand-side expressions of send statements are evaluated exactly once, in source order, upon entering the "select" statement. The result is a set of channels to receive from or send to, and the corresponding values to send. Any side effects in that evaluation will occur irrespective of which (if any) communication operation is selected to proceed. Expressions on the left-hand side of a RecvStmt with a short variable declaration or assignment are not yet evaluated.
> 2. If one or more of the communications can proceed, a single one that can proceed is chosen via a uniform pseudo-random selection. Otherwise, if there is a default case, that case is chosen. If there is no default case, the "select" statement blocks until at least one of the communications can proceed.
> 3. Unless the selected case is the default case, the respective communication operation is executed.
> 4. If the selected case is a RecvStmt with a short variable declaration or an assignment, the left-hand side expressions are evaluated and the received value (or values) are assigned.
> 5. The statement list of the selected case is executed.

看一下执行的第一步，对于语句中的所有情况，在输入“select”语句时，接收操作的通道操作数以及发送语句的通道和右侧表达式仅按源顺序计算一次。意味着 select 会拿到右侧语句的结果才执行，如果拿不到就会一直堵塞，造成死锁

接下来看一下示例代码

## 示例

```go
// deadlock
package main

import "sync"

func main() {
    var wg sync.WaitGroup
    foo := make(chan int)
    bar := make(chan int)
    closing := make(chan struct{})
    wg.Add(1)
    go func() {
        defer wg.Done()
        select {
            case foo <- <-bar:
            case <-closing:
            println("closing")
        }
    }()
    //bar <- 123
    close(closing)
    wg.Wait()
}
// print closing
package main

import "sync"

func main() {
    var wg sync.WaitGroup
    foo := make(chan int)
    bar := make(chan int)
    closing := make(chan struct{})
    wg.Add(1)
    go func() {
        defer wg.Done()
        select {
            case v := <-bar:
                foo <- v
            case <-closing:
                println("closing")
        }
    }()
    close(closing)
    wg.Wait()
}
```
看一下上面的两段代码，第一个 `deadlock`，第二个能够正常执行输出 `closing`。从一开始的 `select` 语句执行步骤，很容易理解，第一段代码中 `foo <- <-bar` 会先执行 `<-bar`，读取 `bar` 管道，但是 `bar` 没有被写入，一直读取失败。`case` 一直堵塞，导致了死锁

第二段语句把管道赋值给了 `v` ，就不会堵塞。但是笔者就很疑惑为什么输出 `closing`

这是因为 `closing` 这个管道在 `master goroutine` 中关闭。那如果把 `bar` 和 `foo` 管道关闭，是不是执行第一个 `case`，答案是不可以的。因为如果只是关闭管道，那么 `foo` 管道并没有被读，就会出现 `panic: send on closed channel`，但是在 `foo` 管道关闭前，读取就会执行第一个 `case`，看一下下面代码，这段代码会输出 `bar`

```go
// print bar
package main

import (
	"sync"
)

func main() {
	var wg sync.WaitGroup
	foo := make(chan int)
	bar := make(chan int)
	closing := make(chan struct{})
	wg.Add(1)
	go func() {
		defer wg.Done()
		select {
		case v := <-bar:
			println("bar")
			foo <- v
		case <-closing:
			println("closing")
		}
	}()
	// bar <- 123
	close(closing)
	close(bar)
	<-foo
	close(foo)
	wg.Wait()
}

```

从这里可以看出，实际上造成 select 死锁是因为 channel 管道堵塞，如果我们把管道关闭，或者对管道写入并读取，管道不阻塞就不会出现死锁

最初始的代码还可以改成下面，避免死锁

```go
package main

import (
	"sync"
)

func main() {
	var wg sync.WaitGroup
	foo := make(chan int)
	bar := make(chan int)
	closing := make(chan struct{})
	wg.Add(1)
	go func() {
		defer wg.Done()
		select {
		case foo <- <-bar:
			println("bar")
		case <-closing:
			println("closing")
		}
	}()
	// bar <- 123
	close(closing)
	close(bar)
	<-foo
	close(foo)
	wg.Wait()
}
```

## 参考
[1] [spec](https://go.dev/ref/spec)
[2] [一个select死锁问题](https://mp.weixin.qq.com/s/F1RGLrh371l_NpeC42FRKw)