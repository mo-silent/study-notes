---
title: Go 语言使用 Redis
slug: go-operation-redis
categories:
  - Golang
tags:
  - Development language
halo:
  site: https://blog.silentmo.cn
  name: 1326a87d-c46c-4800-a320-87215dca4d58
  publish: true
---
# Go 语言使用 Redis

[Redis client for Go](https://github.com/go-redis/redis)

## 连接到 Redis 客户端
安装
```go
go get github.com/go-redis/redis/v8
```
连接
```go
import "github.com/go-redis/redis/v8"
// 参数方式连接
rdb := redis.NewClient(&redis.Options{
	Addr:	  "localhost:6379",
	Password: "", // no password set
	DB:		  0,  // use default DB
})

// 连接字符串方式
opt, err := redis.ParseURL("redis://<user>:<pass>@localhost:6379/<db>")
if err != nil {
	panic(err)
}

rdb := redis.NewClient(opt)

// 连接 TLS redis
rdb := redis.NewClient(&redis.Options{
	TLSConfig: &tls.Config{
		MinVersion: tls.VersionTLS12,
		//Certificates: []tls.Certificate{cert}
	},
})
```

如果 TLS 报错模式 `x509: cannot validate certificate for xxx.xxx.xxx.xxx because it doesn't contain any IP SANs`，设置 `ServerName` 选项即可

```go
rdb := redis.NewClient(&redis.Options{
	TLSConfig: &tls.Config{
		MinVersion: tls.VersionTLS12,
		ServerName: "your.domain.com",
	},
})
```

SSH 方式连接

```go
sshConfig := &ssh.ClientConfig{
	User:			 "root",
	Auth:			 []ssh.AuthMethod{ssh.Password("password")},
	HostKeyCallback: ssh.InsecureIgnoreHostKey(),
	Timeout:		 15 * time.Second,
}

sshClient, err := ssh.Dial("tcp", "remoteIP:22", sshConfig)
if err != nil {
	panic(err)
}

rdb := redis.NewClient(&redis.Options{
	Addr: net.JoinHostPort("127.0.0.1", "6379"),
	Dialer: func(ctx context.Context, network, addr string) (net.Conn, error) {
		return sshClient.Dial(network, addr)
	},
	// Disable timeouts, because SSH does not support deadlines.
	ReadTimeout:  -1,
	WriteTimeout: -1,
}) 
```

## 一般写操作
[go-redis API](https://pkg.go.dev/github.com/go-redis/redis/v8#pkg-examples)

```go
// String
err := rdb.Set(ctx, "key", "value", 0).Err()
if err != nil {
    panic(err)
}

// 列表 List
rdb.LPush(ctx, "list", 1, 2, 3)

// 集合
rdb.SAdd(ctx, "team", "kobe", "jordan")
rdb.SAdd(ctx, "team", "curry")
rdb.SAdd(ctx, "team", "kobe")

// hash
rdb.HSet(ctx, "user", "key1", "value1", "key2", "value2")
rdb.HSet(ctx, "user", []string{"key3", "value3", "key4", "value4"})
rdb.HSet(ctx, "user", map[string]interface{}{"key5": "value5", "key6": "value6"})

// 有序集合
rdb.ZAdd(ctx, "zSet", &redis.Z{
    Score:  0,
    Member: 1,
})
rdb.ZAdd(ctx, "zSet", &redis.Z{
    Score:  0,
    Member: 2,
})
rdb.ZAdd(ctx, "zSet", &redis.Z{
    Score:  0,
    Member: 3,
})

// 管道
pipe := rdb.Pipeline()

incr := pipe.Incr(ctx, "pipeline_counter")
pipe.Expire(ctx, "pipeline_counter", time.Hour)

cmds, err := pipe.Exec(ctx)
if err != nil {
	panic(err)
}

// The value is available only after Exec is called.
fmt.Println(incr.Val())

```

## 事务

除了通过 `Pipeline` 开启事务，还可以通过官方提供的 `func (c *Client) TxPipelined(ctx context.Context, fn func(Pipeliner) error) ([]Cmder, error)` 来开启事务

`TxPipelined` 是封装 MULTI 和 EXEC 命令的 `TxPipeline` 管道，看一下 `TxPipeline` 和 `TxPipelined` 的例子

```go
// TxPipeline
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

// 事务 TxPipelined
cmds, err := rdb.TxPipelined(ctx, func(pipe redis.Pipeliner) error {
    for i := 0; i < 2; i++ {
        pipe.Get(ctx, fmt.Sprintf("key%d", i))
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

```

另外还可以通过 Watch 处理事务管道

```go
// Redis transactions use optimistic locking.
const maxRetries = 1000

// Increment transactionally increments the key using GET and SET commands.
func increment(key string) error {
	// Transactional function.
	txf := func(tx *redis.Tx) error {
		// Get the current value or zero.
		n, err := tx.Get(ctx, key).Int()
		if err != nil && err != redis.Nil {
			return err
		}

		// Actual operation (local in optimistic lock).
		n++

		// Operation is commited only if the watched keys remain unchanged.
		_, err = tx.TxPipelined(ctx, func(pipe redis.Pipeliner) error {
			pipe.Set(ctx, key, n, 0)
			return nil
		})
		return err
	}

    // Retry if the key has been changed.
	for i := 0; i < maxRetries; i++ {
		err := rdb.Watch(ctx, txf, key)
		if err == nil {
			// Success.
			return nil
		}
		if err == redis.TxFailedErr {
			// Optimistic lock lost. Retry.
			continue
		}
		// Return any other error.
		return err
	}

	return errors.New("increment reached maximum number of retries")
}
```
