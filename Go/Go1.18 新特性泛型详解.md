---
title: Go 1.18 新特性泛型详解
slug: go1.18-generics-details
categories:
  - Golang
tags:
  - Development language
halo:
  site: https://weblog.silentmo.cn
  name: 78cfdc72-49f0-41bd-8b26-d7342b88e2e7
  publish: true
---
# Go 1.18 新特性泛型详解

Go 1.18 版本新增了一个功能：支持泛型编程。

如果是其他语言转 Go 语言的开发者，那么能够理解什么是泛型，以及如何使用？

但只是 Go 语言的初学者，并没有接触过泛型编程的人来说，这个功能可能一头雾水。

> 本文希望能让为接触泛型编程的人也能很好的理解和使用 Go 的泛型
> a general guideline for programming Go: write Go programs by writing code, not by defining types
> Go 编程的通用准则：通过编写代码，而不是定义类型来写 Go 程序

## 什么是泛型？

泛型就是编写模板适应所有类型，只有在具体使用时才定义具体变量类型

### 函数的形参和实参

函数定义时的参数是形参 (parameter)，在实际使用函数传入的参数为实参 (argument)

假设有一个加法函数，这个函数有两个参数都是 `int` 类型，返回值也是 `int`；定义如下：
```go
func Test(a,b int) int {
    return a + b
}
```

如果传入的两个实参都是 `int` 类型，那么函数自然能够正常执行。但是这个函数只能用来做 `int` 类型的加法运算，假设还需要进行 `float64` 类型的加法运算，我们就需要再写一个函数

两三个类型加法计算写出来也不麻烦，复制粘贴而已。但是如果所有可计算类型都要进行加法运行，那么代码就会不够精简，阅读起来很不友好

这时，我们就会思考，如果一个函数能够接收所有的计算类型，这样就两三行代码写完一个计算函数了。只需要在定义函数形参时，不指定具体类型，只是定义一个类型组合或者一个占位符，就能够实现这个功能

这个类型组合或占位符就是类型参数，在定义时使用类型形参 (type parameter)，实际调用时使用类型实参 (type argument)

一开始的计算函数转为类型形参函数如下：
```go
// T 是一个类型形参，在定义函数时类型是不确定的，这里的 any 是 go 泛型定义好的一组类型组合
func Test[T any](a,b T) T {
    return a + b
}
// 调用时传入类型实参，伪代码Test[int](1,2)
Test(1,2)
```

通过引入**类型形参**和**类型实参**的概念，让一个函数能够处理多种不同类型数据的能力，这种编程方式被称为**泛型编程**

### 为什么是泛型?

前面的加法示例，除了使用泛型，还可以通过 Go 的接口+反射实现动态数据类型处理。泛型能实现的功能通过接口+反射也基本能够实现，但是如果你使用过反射，那么就会明白反射机制有很多问题：

- 使用麻烦，需要有很强的逻辑思维
- 失去编译时类型检查，容易出现 bug
- 性能不好

但也并不能说所有场景都使用泛型，泛型并不是万金油，泛型有对应的适用场景，可以阅读一下 Go 泛型设计者 Ian Lance Taylor 在官方博客网站上发表了一篇文章 [when to use generics](https://go.dev/blog/when-generics)

> 一句话总结泛型使用场景：当你分别为不同类型写逻辑完全相同的代码时，那么使用泛型是最合适的选择

## Go 泛型的示例

### 泛型函数
```go
// Add sums the values of T. It supports string, int, int64 and float64
//
// @Description A simple additive generic function
// @Description 一个简单的加法泛型函数
// @parameter	a, b	T string | int | int64 | float64	"generics parameter"
// @return		c		T string | int | int64 | float64	"generics return"
func Add[T string | int | int64 | float64](a, b T) T {
	return a + b
}

// 使用
Add(1, 2)
Add(1.0,2.0)
```
### 泛型类型
```go
// MyChan Custom generics chan type
// 一个泛型通道，可用类型实参 int 或 string 实例化
type MyChan[T int | string] chan T
```

### 声明类型限制 (type constraint)

在 Go 的类型限制是通过接口实现

```go
// CustomizationGenerics custom generics
//
// @Description custom generics, which are type restrictions
// @Description ~is a new symbol added to Go 1.18, and the ~ indicates that the underlying type is all types of T. ~ is pronounced astilde in English
// @Description 自定义泛型，即类型限制
// @Desciption ~ 是 Go 1.18 新增的符号，~ 表示底层类型是T的所有类型。~ 的英文读作 tilde
//
// @Example With the addition of ~, MyInt can be used, otherwise there will be type mismatch
// @Example 加上 ~，那么 MyInt 自定义的类型能够被使用，否则会类型不匹配
type CustomizationGenerics interface {
	~int | ~int64
}
```

## 参考

[1] [When To Use Generics](https://go.dev/blog/when-generics)
[2] [Tutorial: Getting started with generics](https://go.dev/doc/tutorial/generics)
[3] [Go 1.18 泛型全面讲解：一篇讲清泛型的全部](https://segmentfault.com/a/1190000041634906)
