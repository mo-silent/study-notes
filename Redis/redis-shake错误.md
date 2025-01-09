---
title: redis-shake 使用中遇到的错误整理
slug: redis-shake-in-use-problem
categories:
  - Redis
tags:
  - Database
halo:
  site: https://weblog.silentmo.cn
  name: 7700ad46-4236-4bc2-ba97-999950ea7236
  publish: true
---
# redis-shake 使用中遇到的错误整理

本文记录一下，笔者在使用 redis-shake 遇到的一些错误

## redis-shake decode 错误测试

在运行 redis-shake 的 decode 模式时出现下面错误，但是百度和谷歌找不到答案，错误信息：
```go
2022/04/23 09:45:49 [PANIC] parse rdb header error
[error]: EOF
	5   github.com/alibaba/RedisShake/pkg/rdb/reader.go:102
			github.com/alibaba/RedisShake/pkg/rdb.(*rdbReader).Read
	4   io/io.go:328
			io.ReadAtLeast
	3   io/io.go:347
			io.ReadFull
	2   github.com/alibaba/RedisShake/pkg/rdb/reader.go:445
			github.com/alibaba/RedisShake/pkg/rdb.(*rdbReader).readFull
	1   github.com/alibaba/RedisShake/pkg/rdb/loader.go:34
			github.com/alibaba/RedisShake/pkg/rdb.(*Loader).Header
	0   github.com/alibaba/RedisShake/redis-shake/common/utils.go:953
			github.com/alibaba/RedisShake/redis-shake/common.NewRDBLoader.func1
```

从报错信息上看，是在 `github.com/alibaba/RedisShake/redis-shake/common/utils.go` 953 行代码出错，而且还是 io 读取文件出错

这个错误就很奇怪了，为什么读取的 rdb 文件会出错？因为笔者的 rdb 文件是运行 redis-shake 的 dump 模式得到的

这时笔者想到 redis-shake 是开源，就拉取官方源代码进行测试，这里是把 decode 部分代码简化出来，并不是完整流程，只是到笔者报错的地方

把文件读取的流程整理出来，从运行 redis-shake 的机器上拉取这个 rdb 文件进行测试

当笔者把代码都切割出来后，重新在运行 go 程序，发现并没有报错，正常读取了文件；然后重新在 redis-shake 的机器上运行 decode 模式，发现正常了

真是让人哭笑不得，不过后面思考了一下出现这个问题原因，应该是这个 rdb 文件格式问题

因为笔者后面测试的 rdb 文件其实是重新运行 dump 模式生成的新文件，一开始使用的 rdb 文件是前一天生成并且多次运行 dump 覆盖的，可能导致有一些问题

### 结论

redis-shake 生成的 rdb 文件尽量别多次覆盖模式，每一次最好都重新命名

### 笔者代码

https://github.com/mo-silent/go-study/tree/main/go-redis-shake-decode
