# Go 语言中的 logger 和 zap 日志库

在软件开发过程中，需要进行关键日志记录，便于后期的审计和排错。

一个好的日志记录器应该具备以下功能：

- 日志写入到文件而不是控制台输出
- 日志切割-按文件大小、时间或间隔等切割日志文件
- 支持不同的日志级别，如：INFO，DEBUG，ERROR 等
- 能打印基本信息，如调用文件/函数名和行号，日志时间等

## Go Logger

配置日志输出文件

```go
func SetupLogger() {
	logFileLocation, _ := os.OpenFile("./test.log", os.O_CREATE|os.O_APPEND|os.O_RDWR, 0744)
	log.SetOutput(logFileLocation)
}
```

编写示例，建立一个到 URL 的 HTTP 连接，记录状态码/错误记录到日志文件

```go
func simpleHttpGet(url string) {
	resp, err := http.Get(url)
	if err != nil {
		log.Printf("Error fetching url %s : %s", url, err.Error())
	} else {
		log.Printf("Status Code for %s : %s", url, resp.Status)
		resp.Body.Close()
	}
}
```

main 函数，执行示例

```go
func main() {
	SetupLogger()
	simpleHttpGet("www.google.com")
	simpleHttpGet("http://www.google.com")
}
```
看到 `test.log` 文件被创建，并且记录了下面的内容
```bash
2022/04/22 11:33:52 Error fetching url www.google.com : Get "www.google.com": unsupported protocol scheme ""
2022/04/22 11:33:53 Status Code for http://www.google.com : 200 OK
```

## Zap Logger
```go
go get -u go.uber.org/zap
```
Zap 提供两种类型的日志记录器- `Sugared Logger` 和 `Logger`

在性能很好但不是很关键的上下文中，使用 `SugaredLogger`。它比其他结构化日志记录包快 4-10 倍，并且支持结构化和 printf 风格的日志记录

在每一微秒和每一次内存分配都很重要的上下文中，使用 `Logger`。甚至比 `SugaredLogger` 更快，内存分配次数也更少，但它只支持强类型的结构化日志记录

### Logger
- 通过调用 `zap.NewProduction()/zap.NewDevelopment()` 或者 `zap.Example()` 创建一个 `Logger`
- 上面的每一个函数都将创建一个 `logger`。唯一的区别在于它将记录的信息不同。例如 `production logger` 默认记录调用函数信息、日期和时间等
- 通过 `Logger` 调用 `Info/Error` 等
- 默认情况下日志都会打印到应用程序的 `console` 界面
```go
var logger *zap.Logger

func main() {
	InitLogger()
  defer logger.Sync()
	simpleHttpGet("www.google.com")
	simpleHttpGet("http://www.google.com")
}

func InitLogger() {
	logger, _ = zap.NewProduction()
}

func simpleHttpGet(url string) {
	resp, err := http.Get(url)
	if err != nil {
		logger.Error(
			"Error fetching url..",
			zap.String("url", url),
			zap.Error(err))
	} else {
		logger.Info("Success..",
			zap.String("statusCode", resp.Status),
			zap.String("url", url))
		resp.Body.Close()
	}
}
```
### Sugared Logger
- 大部分的实现基本都相同
- 唯一的区别是，我们通过调用主 `logger` 的. `Sugar()` 方法来获取一个 `SugaredLogger`
- 然后使用 `SugaredLogger` 以 `printf` 格式记录语句

```go
var sugarLogger *zap.SugaredLogger

func main() {
	InitLogger()
	defer sugarLogger.Sync()
	simpleHttpGet("www.google.com")
	simpleHttpGet("http://www.google.com")
}

func InitLogger() {
    logger, _ := zap.NewProduction()
	sugarLogger = logger.Sugar()
}

func simpleHttpGet(url string) {
	sugarLogger.Debugf("Trying to hit GET request for %s", url)
	resp, err := http.Get(url)
	if err != nil {
		sugarLogger.Errorf("Error fetching URL %s : Error = %s", url, err)
	} else {
		sugarLogger.Infof("Success! statusCode = %s for URL %s", resp.Status, url)
		resp.Body.Close()
	}
}
```

### 定制 Logger 记录到文件中

1. Encoder: 编码器(如何写入日志)。我们将使用开箱即用的 `NewJSONEncoder()`，并使用预先设置的
2. WriterSyncer: 指定日志将写到哪里去。我们使用 `zapcore.AddSync()` 函数并且将打开的文件句柄传进去
3. Log Level: 哪种级别的日志将被写入

```go
var logger *zap.Logger
func main() {
	InitLogger()
    defer logger.Sync()
	simpleHttpGet("www.google.com")
	simpleHttpGet("http://www.google.com")
}

func InitLogger() {
	writeSyncer := getLogWriter()
	encoder := getEncoder()
	core := zapcore.NewCore(encoder, writeSyncer, zapcore.DebugLevel)

	logger := zap.New(core)
	// sugarLogger = logger.Sugar()
}

func getEncoder() zapcore.Encoder {
	return zapcore.NewJSONEncoder(zap.NewProductionEncoderConfig())
}

func getLogWriter() zapcore.WriteSyncer {
	file, _ := os.Create("./test.log")
	return zapcore.AddSync(file)
}

func simpleHttpGet(url string) {
	resp, err := http.Get(url)
	if err != nil {
		logger.Error(
			"Error fetching url..",
			zap.String("url", url),
			zap.Error(err))
	} else {
		logger.Info("Success..",
			zap.String("statusCode", resp.Status),
			zap.String("url", url))
		resp.Body.Close()
	}
}

```
将 JSON Encoder 更改为普通的 Log Encoder，只需要将 `NewJSONEncoder()` 更改为`NewConsoleEncoder()`

### zap logger 中加入 Lumberjack

`Lumberjack` 是一个 Go 包，用于日志切割归档功能

> zap 不支持日志切割归档，lumberjack 也是 [zap 官方推荐](https://github.com/uber-go/zap/blob/master/FAQ.md#does-zap-support-log-rotation)的

通过 `zapcore.AddSync` 加入 `lumberjack` 功能

```go
// lumberjack.Logger is already safe for concurrent use, so we don't need to
// lock it.
w := zapcore.AddSync(&lumberjack.Logger{
  Filename:   "/var/log/myapp/foo.log",
  MaxSize:    500, // megabytes
  MaxBackups: 3,
  MaxAge:     28, // days
})
core := zapcore.NewCore(
  zapcore.NewJSONEncoder(zap.NewProductionEncoderConfig()),
  w,
  zap.InfoLevel,
)
logger := zap.New(core)
```

Lumberjack Logger采用以下属性作为输入:

- Filename: 日志文件的位置
- MaxSize：在进行切割之前，日志文件的最大大小（以MB为单位）
- MaxBackups：保留旧文件的最大个数
- MaxAges：保留旧文件的最大天数
- Compress：是否压缩/归档旧文件

完整代码：https://gitee.com/MoGD/go-study/blob/main/go-zap/sugaredLogger/main.go

## 参考

[1] [在Go语言项目中使用Zap日志库(李文周)](https://www.liwenzhou.com/posts/Go/zap/)
[2] [Go Logger](https://pkg.go.dev/log)
[3] [Go zap](https://pkg.go.dev/go.uber.org/zap) 