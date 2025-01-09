# 万字总结：从零开始学 Go

> Author mogd 2022-04-25
> Update mogd 2022-05-17

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

## 二、Go 基础 (变量，常量，类型)

在 Go 语言中声明的变量一定要使用，声明但未使用的变量在编译阶段会报错

Go 语言设计有一些默认的行为规则:
- 大写字母开头的变量是可导出的，也就是其它包可以读取的，是公有变量；小写字母开头的就是不可导出的，是私有变量
- 大写字母开头的函数也是一样，相当于 `class` 中的带 `public` 关键词的公有函数；小写字母开头的就是有 `private` 关键词的私有函数
 
### 2.1 Go 变量声明和初始化

Go 语言使用 `var` 声明变量，或者直接通过 `:=` 符合声明和赋值

```go
// 声明变量 var variableName type
var a int
var a, b int // 同类型变量可以一起声明
// 声明一组变量
var (
    a int
    b string
    c float64
)
// 声明并赋值
var a int = 1
// 简写，编译器会根据初始化的值自动推导出相应的类型
a := 1 
```

在 Go 语言中还有一种特殊的变量 `_` (下划线)，任何赋予它的值都会丢弃，但只能使用在函数内部

### 2.2 Go 常量

常量即在程序编译阶段就能确定下来的值，在程序运行时无法改变。在 Go 语言中，常量可定义为数值、布尔值或字符串等类型，通过 `const` 关键字定义常量

```go
const Pi = 3.1415926
const i = 10000
const MaxThread = 10
const prefix = "astaxie_"
```

Go 常量可以指定相当多的小数位数(例如200位)， 若指定給float32自动缩短为32bit，指定给float64自动缩短为64bit

### 2.3 一般的类型 (布尔，整型，字符串，错误类型)

```go
// boolean 布尔值类型，值为 true or false
var enabled, disabled = true, false 
// 数值类型，rune, int8, int16, int32, int64和byte, uint8, uint16, uint32, uint64
// 其中 rune 是 int32 的别称，byte 是 uint8 的别称
// 类型的变量之间不允许互相赋值或操作，不然会在编译时引起编译器报错
// Go 还支持复数，complex128（64位实数+64位虚数），complex64(32位实数+32位虚数)；复数的形式为 RE + IMi，其中 RE 是实数部分，IM 是虚数部分，而最后的 i 是虚数单位
var c complex64 = 5+5i
// 字符串 string，字符串是用一对双引号（""）或反引号（` `）括起来定义
var testString string = "test" 
m := `hello
	world` // 多行字符串
// 在 Go 中字符串是不可变的，如果需要修改，需要转化为 []byte (切片) 类型，再转换回 string 类型
s := "hello"
c := []byte(s) 
c[0] = 'c'
s2 := string(c)
fmt.Printf("%s\n", s2)
// 修改字符串精简写法，字符串虽不能更改，但可进行切片操作
s := "hello"
s = "c" + s[1:] 
fmt.Printf("%s\n", s)
// 错误类型 error，专门用来处理错误信息，Go 的 package 里面还专门有一个包 errors 来处理错误
err := errors.New("emit macho dwarf: elf header corrupted")
if err != nil {
	fmt.Print(err)
}
```

### 2.4 iota 枚举

Go 语言中有一个关键字 `iota` 用来声明 `enum` 时采用，默认开始值为0，每行 +1，同一行值相同

```go
const (
	x = iota // x == 0
	y = iota // y == 1
	z = iota // z == 2
	w        // 常量声明省略值时，默认和之前一个值的字面相同。这里隐式地说w = iota，因此w == 3。其实上面y和z可同样不用"= iota"
)

const v = iota // 每遇到一个const关键字，iota就会重置，此时v == 0

const (
	h, i, j = iota, iota, iota //h=0,i=0,j=0 iota在同一行值相同
)

const (
    a       = iota //a=0
    b       = "B"
    c       = iota             //c=2
    d, e, f = iota, iota, iota //d=3,e=3,f=3
    g       = iota             //g = 4
)
// 更有意义的使用
const (
    Failed = iota - 1 // == -1
    Unknown // == 0
    Succeeded // == 1
)
const (
    Readable = 1 << iota // 1 << 0 = 1 = 1
    Writable // 1 << 1 = 10 = 2
    Executable // 1 << 2 = 100 = 4
)

```
> 除非被显式设置为其它值或 `iota`，每个 `const` 分组的第一个常量被默认设置为它的 0 值，第二及后续的常量被默认设置为它前面那个常量的值，如果前面那个常量的值是 `iota`，则它也被设置为 `iota`

### 2.5 数组 array 和 切片 slice

`array` 是一个固定长度的数组，而 `slice` 在 Go 语言中称为切片，也可以理解为动态数组；`slice` 并不是真正意义上的动态数组，而是一个引用类型。`slice` 总是指向底层的一个 `array`

```go
// 数组声明 var arr [n]type
var arr [10]int  // 声明了一个int类型的数组
arr[0] = 42      // 数组下标是从0开始的
arr[1] = 13      // 赋值

// 切片声明，和声明array一样，只是少了长度 var arr []type
var fslice []int
// 声明并初始化
slice := []byte {'a', 'b', 'c', 'd'}
```

`slice` 可以从一个数组或一个已经存在的 `slice` 中再次声明
```go

// 声明一个含有10个元素元素类型为byte的数组
var ar = [10]byte {'a', 'b', 'c', 'd', 'e', 'f', 'g', 'h', 'i', 'j'}

// 声明两个含有byte的slice
var a, b []byte

// a指向数组的第3个元素开始，并到第五个元素结束，
a = ar[2:5]
//现在a含有的元素: ar[2]、ar[3]和ar[4]

// b是数组ar的另一个slice
b = ar[3:5]
// b的元素是：ar[3]和ar[4]
```
> `slice` 和数组在声明时的区别：声明数组时，方括号内写明了数组的长度或使用...自动计算长度，而声明 `slice` 时，方括号内没有任何字符
> `slice` 是引用类型，所以当引用改变其中元素的值时，其他的引用都会改变该值，即 Go 语言中的浅拷贝

如果不想改变初始值，需要使用 `copy()` 进行深拷贝

```go
slice1 := []int{1,2,3,4,5}
slice2 := make([]int,0,5)
copy(slice2,slice1)
```
> 多维数组：`var arrayName [ x ][ y ] variable_type`
### 2.6 字典 map 

`map` 就是 `Python` 中的字典概念，格式 `map[keyType]valueType`

```go
// 声明一个key是字符串，值为int的字典,这种方式的声明需要在使用之前使用make初始化
var numbers map[string]int
// 另一种map的声明方式
numbers = make(map[string]int)
numbers["one"] = 1  //赋值
numbers["ten"] = 10 //赋值
numbers["three"] = 3
delete(numbers, "one")  // 删除key为one的元素
fmt.Println("第三个数字是: ", numbers["three"]) // 读取数据
// 打印出来如:第三个数字是: 3
```

Go 语言中，`map` 需要注意几点：
- `map` 是无序的，每次打印出来的 `map` 都会不一样，它不能通过 `index` 获取，而必须通过 `key` 获取
- `map` 的长度是不固定的，也就是和 `slice` 一样，也是一种引用类型
- 内置的 `len` 函数同样适用于 `map`，返回 `map` 拥有的 `key` 的数量
- `map` 的值可以很方便的修改，通过 `numbers["one"]=11` 可以很容易的把 `key` 为 `one` 的字典值改为 `11`
- `map` 和其他基本型别不同，它不是 `thread-safe`，在多个 `go-routine` 存取时，必须使用 `mutex lock` 机制

`map` 也是一种引用类型，如果两个 `map` 同时指向一个底层，那么一个改变，另一个也相应改变
```go
m := make(map[string]string)
m["Hello"] = "Bonjour"
m1 := m
m1["Hello"] = "Salut"  // 现在m["hello"]的值已经是Salut了
```

### 2.7 make、new 操作

`make` 用于内建类型 (`map`, `slice` 和 `channel`) 内存分配，`new` 用于各种类型内存分配

内建函数 `new` 本质上说跟其它语言中的同名函数功能一样：`new(T)` 分配了零值填充的T类型的内存空间，并且返回其地址，即一个 `*T` 类型的值。用 Go 的术语说，它返回了一个指针，指向新分配的类型 `T` 的零值。有一点非常重要：
> `new` 返回指针

内建函数 `make(T, args)` 与 `new(T)` 有着不同的功能，`make` 只能创建 `slice`、`map` 和 `channel`，并且返回一个有初始值(非零)的 `T` 类型，而不是 `*T`。本质来讲，导致这三个类型有所不同的原因是指向数据结构的引用在使用前必须被初始化。例如，一个 `slice`，是一个包含指向数据（内部 `array`）的指针、长度和容量的三项描述符；在这些项目被初始化之前，`slice` 为 `nil`。对于 `slice`、`map` 和 `channel` 来说，`make` 初始化了内部的数据结构，填充适当的值
> `make` 返回初始化后的非零值

### 2.8 结构体 struct

Go 语言通过 `struct` 关键字声明结构体

```go
// 定义一个结构体
type person struct {
	name string
	age int
}

// 声明结构体类型变量
var P person  

P.name = "Astaxie"  // 赋值 "Astaxie" 给 P 的 name 属性.
P.age = 25  // 赋值 "25" 给变量 P 的 age 属性
fmt.Printf("The person's name is %s", P.name)  // 访问 P 的 name 属性.
```

而在 Go 语言中还支持匿名字段，也称为嵌入字段；即支持只提供类型，而不写字段名的方式，匿名字段的变量名就是类型本身

当匿名字段是一个 `struct` 的时候，那么这个 `struct` 所拥有的全部字段都被隐式地引入了当前定义的这个 `struct`

```go
type Human struct {
	name string
	age int
	weight int
}

type Student struct {
	Human  // 匿名字段，那么默认Student就包含了Human的所有字段，
	speciality string
    int    // 内置类型作为匿名字段，变量名就是 int
}
jane := Student{Human:Human{"Jane", 35, 100}, speciality:"Biology", int: 1}
fmt.Println("Her name is ", jane.name) // 匿名字段为 struct，可等同于继承
fmt.Println("Her preferred number is", jane.int)
```

## 三、Go 语言中的深拷贝和浅拷贝

### 3.1 Python 与 Go 深浅拷贝的不同

深拷贝：
- Go 语言是拷贝值，内存地址不同
- Python 拷贝所有层引用，内存地址相同
  
浅拷贝：
- Go 语言是拷贝引用，内存地址相同
- Python 拷贝第一层引用 (即拷贝值)，内存地址不同

### 切片说明 Go 深浅拷贝

**深拷贝**
拷贝切片与源切片指向不同的底层数组，任何数组元素的改变都不影响另一个
```go
slice1 := []int{1,2,3,4,5}
slice2 := make([]int,0,5)

copy(slice2,slice1)
 
slice1[1]=6  //只会影响slice1
```

**浅拷贝**
目的切片和源切片指向同一个底层数组，任何一个数组元素改变，都会同时影响两个数组
```go
slice1 := []int{1,2,3,4,5}
slice2 := slice1

//同时改变两个数组
slice1[1]=6
```

## 四、Go 函数和方法 method

在Go语言中，函数是指不属于任何结构体、类型的方法，也就是说，函数是没有接收者的；而方法是有接收者的

### 4.1 函数
函数是 Go 的核心设计，通过`func` 关键字来声明

```go
func funcName(input1 type1, input2 type2) (output1 type1, output2 type2) {
	//这里是处理逻辑代码
	//返回多个值
	return value1, value2
}
```
函数中的参数和返回值都是可选，函数可以完全没有参数和返回，也可以多个参数多个返回值

Go 函数还支持变参，接受变参的函数是有着不定数量的参数的
```go
func myfunc(arg ...int) {} 
```

在 Go 中函数也是一种变量，可以通过 `type` 来定义
```go
package main

import "fmt"

type testInt func(int) bool // 声明了一个函数类型

func isOdd(integer int) bool {
	if integer%2 == 0 {
		return false
	}
	return true
}

func isEven(integer int) bool {
	if integer%2 == 0 {
		return true
	}
	return false
}

// 声明的函数类型在这个地方当做了一个参数

func filter(slice []int, f testInt) []int {
	var result []int
	for _, value := range slice {
		if f(value) {
			result = append(result, value)
		}
	}
	return result
}

func main(){
	slice := []int {1, 2, 3, 4, 5, 7}
	fmt.Println("slice = ", slice)
	odd := filter(slice, isOdd)    // 函数当做值来传递了
	fmt.Println("Odd elements of slice are: ", odd)
	even := filter(slice, isEven)  // 函数当做值来传递了
	fmt.Println("Even elements of slice are: ", even)
}
```

### 4.2 方法 method

方法的声明和函数类似，他们的区别是：方法在定义的时候，会在 `func` 和方法名之间增加一个参数，这个参数就是接收者，这样我们定义的这个方法就和接收者绑定在了一起，称之为这个接收者的方法

调用的方法非常简单，使用类型的变量进行调用即可，类型变量和方法之前是一个 `.` 操作符，表示要调用这个类型变量的某个方法的意思

```go
type person struct {
	name string
}

func (p person) String() string{
	return "the person name is "+p.name
}
func main() {
	p:=person{name:"张三"}
	fmt.Println(p.String())
}
```

Go 语言提供了另外一种数据类型即接口，它把所有的具有共性的方法定义在一起，任何其他类型只要实现了这些方法就是实现了这个接口，也可以理解为继承

下面的示例中定义了一个接口 `Phone`，接口里面有一个方法 `call()`。然后我们在 `main` 函数里面定义了一个 `Phone` 类型变量，并分别为之赋值为 `NokiaPhone` 和 `IPhone`。然后调用 `call()` 方法
```go
package main

import (
    "fmt"
)
// 定义接口
type Phone interface {
    call()
}
// 定义结构体
type NokiaPhone struct {
}
// 实现接口方法
func (nokiaPhone NokiaPhone) call() {
    fmt.Println("I am Nokia, I can call you!")
}
// 定义结构体
type IPhone struct {
}
// 实现接口方式
func (iPhone IPhone) call() {
    fmt.Println("I am iPhone, I can call you!")
}

func main() {
    var phone Phone

    phone = new(NokiaPhone)
    phone.call()

    phone = new(IPhone)
    phone.call()

}
```

### 4.3 defer

`Go` 语言中有种不错的设计，即延迟（`defer`）语句，你可以在函数中添加多个 `defer` 语句。当函数执行到最后时，这些 `defer` 语句会按照逆序执行 (即堆栈的方式，先进后出)，最后该函数返回

```go
defer fmt.Printf(1)
defer fmt.Printf(2)
defer fmt.Printf(3)
```

得到的输出结果是 `3 2 1`

### 4.4 panic 和 recover

`Go` 没有像 `Java` 那样的异常机制，它不能抛出异常，而是使用了 `panic` 和 `recover` 机制

Panic
> 是一个内建函数，可以中断原有的控制流程，进入一个 `panic` 状态中。当函数 `F` 调用 `panic`，函数 `F` 的执行被中断，但是 `F` 中的延迟函数会正常执行，然后 `F` 返回到调用它的地方。在调用的地方，`F` 的行为就像调用了 `panic`。这一过程继续向上，直到发生 `panic` 的 `goroutine` 中所有调用的函数返回，此时程序退出。`panic` 可以直接调用 `panic` 产生。也可以由运行时错误产生，例如访问越界的数组

```go
var user = os.Getenv("USER")

func init() {
	if user == "" {
		panic("no value for $USER")
	}
}
```

Recover
> 是一个内建的函数，可以让进入 `panic` 状态的 `goroutine` 恢复过来。`recover` 仅在延迟函数中有效。在正常的执行过程中，调用 `recover` 会返回 `nil`，并且没有其它任何效果。如果当前的 `goroutine` 陷入 `panic` 状态，调用 `recover` 可以捕获到 `panic` 的输入值，并且恢复正常的执行

### 4.5 main 函数和 init 函数

`Go` 里面有两个保留的函数：`init` 函数（能够应用于所有的 `package`）和 `main` 函数（只能应用于 `package main`）

在加载一个代码包的过程中，所有的声明在此包中的 `init` 函数将被串行调用并且仅调用执行一次。 一个代码包中声明的 `init` 函数的调用肯定晚于此代码包所依赖的代码包中声明的 `init` 函数。 所有的 `init` 函数都将在调用 `main` 入口函数之前被调用执行

在同一个源文件中声明的 `init` 函数将按从上到下的顺序被调用执行。 对于声明在同一个包中的两个不同源文件中的两个 `init` 函数，Go 语言白皮书推荐（但不强求）按照它们所处于的源文件的名称的词典序列（对英文来说，即字母顺序）来调用。 所以最好不要让声明在同一个包中的两个不同源文件中的两个 `init` 函数存在依赖关系

在加载一个代码包的时候，此代码包中声明的所有包级变量都将在此包中的任何一个 `init` 函数执行之前初始化完毕

在同一个包内，包级变量将尽量按照它们在代码中的出现顺序被初始化，但是一个包级变量的初始化肯定晚于它所依赖的其它包级变量

执行流程：
![go-func-init](https://gallery-lsky.silentmo.cn/i_blog/2025/01//go-func-init.png)


## 参考
[1] [astaxie/build-web-application-with-golang](https://github.com/astaxie/build-web-application-with-golang)
[2] [Go语言入门教程，Golang入门教程（非常详细）](http://c.biancheng.net/golang/)