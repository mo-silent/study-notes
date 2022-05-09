# Go 语言中 switch 类型断言的用法
> Author mogd 2022-05-09
> Update mogd 2022-05-09
> Adate `Don't live in the past.`

Go 语言官方有推荐的[编码规范](https://go.dev/ref/spec)，在这里记录一次编码中 switch 进行类型断言判断的标准用法

> 使用类型断言的建议是复用结果，而不是直接使用

## 原因

在一次编码中，vscode 出现一个黄色的警告 (不必要的类型断言)：
```go
assigning the result of this type assertion to a variable (switch docs := docs.(type)) could eliminate type assertions in switch cases
```

修改方法很简单，就是将类型断言的结果赋值给变量，而不是直接使用，否则可能触发 `panic`

## 错误例子

```go
switch docs.(type) {
case []interface{}:
    // Triggers a panic as "docs" doesn't hold
    // concrete type "interface" but "[]interface"
    fmt.Println(x.([]interface{}))
}
```

## 推荐语法

> 不要在 `switch` 语句中每次都使用类型断言，而是将类型断言的结果分配给变量并使用
> 类型断言提供对接口底层具体值访问
> 如果不将类型断言结果分配给变量，可能会触发 `panic`

```go
switch docs := docs.(type) {
case []interface{}:
    // Triggers a panic as "docs" doesn't hold
    // concrete type "interface" but "[]interface"
    fmt.Println(docs)
}
```