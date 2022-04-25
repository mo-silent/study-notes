# 万字总结：从零开始学 Go

> Author mogd 2022-04-25
> Update mogd 2022-04-25

## 一、Go 命令行操作

Go 带有系列命令操作：
```go
Go is a tool for managing Go source code.

Usage:

        go <command> [arguments]

The commands are:

        bug         start a bug report
        build       compile packages and dependencies
        clean       remove object files and cached files
        doc         show documentation for package or symbol
        env         print Go environment information
        fix         update packages to use new APIs
        fmt         gofmt (reformat) package sources
        generate    generate Go files by processing source
        get         add dependencies to current module and install them
        install     compile and install packages and dependencies
        list        list packages or modules
        mod         module maintenance
        work        workspace maintenance
        run         compile and run Go program
        test        test packages
        tool        run specified go tool
        version     print Go version
        vet         report likely mistakes in packages

Use "go help <command>" for more information about a command.

Additional help topics:

        buildconstraint build constraints
        buildmode       build modes
        c               calling between Go and C
        cache           build and test caching
        environment     environment variables
        filetype        file types
        go.mod          the go.mod file
        gopath          GOPATH environment variable
        gopath-get      legacy GOPATH go get
        goproxy         module proxy protocol
        importpath      import path syntax
        modules         modules, module versions, and more
        module-get      module-aware go get
        module-auth     module authentication using go.sum
        packages        package lists and patterns
        private         configuration for downloading non-public code
        testflag        testing flags
        testfunc        testing functions
        vcs             controlling version control with GOVCS

Use "go help <topic>" for more information about that topic.
```

### 1.1 Go build

`build` 命令用来编译代码，有以下规则：
- 如果是 `main` 包，执行 `go build` 会在当前目录下生成一个可执行文件
- 如果是普通包，执行 `go build` 不会产生任何包
- `go build` 默认会编译当前目录下所有 `.go` 文件，但是会忽略 `_` 或 `.` 开头的 go 文件；可以在 `go build` 之后加上文件名，指定编译某个文件
- 如果代码针对不同操作系统做不同处理，`go build` 的时候会选择性地编译以系统名结尾的文件；如 `main_linux.go`、`main_windows.go`
  
### 1.2 Go clean

`clean` 命令是用来移除当前源码包和关联源码包编译生成的文件，如：
```go
_obj/            旧的object目录，由Makefiles遗留
_test/           旧的test目录，由Makefiles遗留
_testmain.go     旧的gotest文件，由Makefiles遗留
test.out         旧的test记录，由Makefiles遗留
build.out        旧的test记录，由Makefiles遗留
*.[568ao]        object文件，由Makefiles遗留

DIR(.exe)        由go build产生
DIR.test(.exe)   由go test -c产生
MAINFILE(.exe)   由go build MAINFILE.go产生
*.so             由 SWIG 产生
```
命令执行 `go clean -i -n`，参数：
- `-i` 清除关联的安装的包和可运行文件，也就是通过 `go install` 安装的文件
- `-n` 把需要执行的清除命令打印出来，但是不执行，这样就可以很容易的知道底层是如何运行的
- `-r` 循环的清除在 `import` 中引入的包
- `-x` 打印出来执行的详细命令，其实就是 `-n` 打印的执行版本

### 1.3 Go fmt

Go 语言中，代码有标准的风格，例如左大括号必须放在行尾，否则不能编译通过。为了避免在排版上浪费时间，Go 工具中提供了 `go fmt` 命令，能够帮着 `Gopher` 格式化写好的代码

`go fmt` 命令，实际是调用了 `gofmt`，直接 `go fmt <文件名>.go` 即可；而 `gofmt` 需要带上参数 `-w`，否则格式化结果不会写入文件，`gofmt -w -l src` 格式化整个目录

### 1.4 Go get

`get` 命令是用来动态获取代码包，目前支持 `BitBucket`、`GitHub`、`Google Code` 和 `Launchpad`；

`get` 命令分两步，第一步是下载源码包，第二步执行 `go install`

下载源码包的go工具会自动根据不同的域名调用不同的源码工具，对应关系如下：
```go
BitBucket (Mercurial Git)
GitHub (Git)
Google Code Project Hosting (Git, Mercurial, Subversion)
Launchpad (Bazaar)
```

### 1.5 Go install

`install` 命令也分成两步操作：第一步是生成可执行文件或 `.a` 包，第二步把编译好的文件移到 `$GOPATH/pkg` 或者 `$GOPATH/bin` 下

> 注：使用 `go install` 时，加上 `-v` 以便查看底层执行信息

### 1.6 Go test

`test` 命令会自动执行源码目录下的 `*_test.go` 文件，并生成测试用的可执行文件

默认情况不需要任何参数，不过有几个常用参数需要清楚：
- `-bench regexp` 执行相应的 `benchmarks`，例如 `-bench=.`
- `-cover` 开启测试覆盖率
- `-run regexp` 只运行 `regexp` 匹配的函数，例如 `-run=Array` 那么就执行包含有 `Array` 开头的函数
- `-v` 显示测试的详细命令

### 1.7 Go tool

`go tool` 下聚集了很多命令，这里只介绍两个，`fix` 和 `vet`

- `go tool fix .` 用来修复以前老版本的代码到新版本，例如 `go1` 之前老版本的代码转化到 `go1`, 例如 `API 的变化
- `go tool vet directory|files` 用来分析当前目录的代码是否都是正确的代码,例如是不是调用 `fmt.Printf` 里面的参数不正确，例如函数里面提前 `return` 了然后出现了无用代码之类的

### 1.8 Go generate

`go generate` 命令是在 Go 语言 1.4 版本里面新添加的一个命令，当运行该命令时，它将扫描与当前包相关的源代码文件，找出所有包含 `//go:generate` 的特殊注释，提取并执行该特殊注释后面的命令

> `go generate` 命令注意点：
> - 该特殊注释必须在 `.go` 源码文件中
> - 每个源码文件可以包含多个 `generate` 特殊注释
> - 运行 `go generate` 命令时，才会执行特殊注释后面的命令
> - 当 `go generate` 命令执行出错时，将终止程序的运行
> - 特殊注释必须以 `//go:generate` 开头，双斜线后面没有空格

在下面这些场景下，我们会使用 `go generate` 命令:
- `yacc`：从 `.y` 文件生成 `.go` 文件
- `protobufs`：从 `protocol buffer` 定义文件（`.proto`）生成 `.pb.go` 文件
- `Unicode`：从 `UnicodeData.txt` 生成 `Unicode` 表
- `HTML`：将 `HTML` 文件嵌入到 `go` 源码
- `bindata`：将形如 `JPEG` 这样的文件转成 `go` 代码中的字节数组

### 1.9 godoc and pkgsite

在 `Go1.2` 版本前还支持 `go doc` 命令，之后全部移到了 `godoc`。但是在2019年11月之后，`godoc.org` 已经下线，会重定向到 `pkg.go.dev`。不过两者使用方式类似

```go
// godoc
go get -v golang.org/x/tools/cmd/godoc
cd XXXX; godoc -http=:6060
// pkgsite
go install golang.org/x/pkgsite/cmd/pkgsite@latest

cd XXXX; pkgsite -http=:6060
```

### 1.10 其他命令

`go` 还提供了很多工具，例如：

```go
go version 查看go当前的版本
go env 查看当前go的环境变量
go list 列出当前全部安装的package
go run 编译并运行Go程序
```

## 二、Go 基础 (变量，常量，类型，函数，包)


## 参考
[1] [astaxie/build-web-application-with-golang](https://github.com/astaxie/build-web-application-with-golang)
[2] [Go语言入门教程，Golang入门教程（非常详细）](http://c.biancheng.net/golang/)