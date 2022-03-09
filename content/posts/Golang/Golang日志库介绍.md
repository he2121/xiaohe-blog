---
title: "Golang 常用日志库介绍"
date: 2021-11-09T14:30:12+08:00
tags: ["golang", "日志库", "log", "logrus", "zap"]
---

## 前言

### 为什么需要日志

- 调试开发
- 程序运行日志
- 用户行为日志

不同的目的决定了日志输出的格式、频率。作为开发人员，调试开发阶段打印日志目的是输出尽可能全的信息（如上下文，变量值...），辅助开发测试，因此日志格式要易读，打印频率要高。而在程序运行时，日志格式倾向于结构化（便于分析与搜索），而且为了性能和聚焦于关键信息（如error ），打印频率更偏低。

## Go 标准库 Log

### 使用

我们常使用 Go log 以下三组函数：

- ```Print/Printf/Println``` : 打印日志信息
- ``` Panic/Panicf/Panicln``` : 打印日志信息后，以拼装好的字符串为参数调用 Panic
- ``` Fatal/Fatalf/Fatalln``` : 打印日志信息后，`os.Exit(1)` 退出程序

带 f 后缀的是格式化输出，ln 后缀增加换行符，不过在打印日志场景中会自动增加一个换行符，这里 ln 后缀差别不大。Panic 与 Fatal 的区别在于 Panic 可以被捕获。

示例如下: 

```go
package main

import "log"

func main() {
    log.Println("日志信息1")
    log.Print("日志信息2")
    log.Panicln("日志信息3")
    log.Fatalln("日志信息4") // 运行不到
}
```

```bash
2021/11/09 15:41:34 日志信息1
2021/11/09 15:41:34 日志信息2
2021/11/09 15:41:34 日志信息3
panic: 日志信息3
goroutine 1 [running]:
log.Panicln({0xc0000bdf60, 0x0, 0x1015ba5})
        /usr/local/opt/go/libexec/src/log/log.go:368 +0x65
main.main()
        /Users/xiaohe/go/src/github.com/he2121/log_test/main.go:8 +0xa8
```

可以看到日志格式默认是：时间 + msg。为什么是这样，可以改吗？

### 原理与自定义

```go
func Println(v ...interface{}) {
   std.Output(2, fmt.Sprintln(v...))
}
```

可以看到前面的方法都是调用 ```std``` 对象中 `Output` 方法。

```go
var std = New(os.Stderr, "", LstdFlags)

func New(out io.Writer, prefix string, flag int) *Logger {
    return &Logger{out: out, prefix: prefix, flag: flag}
}

type Logger struct {
    mu     sync.Mutex // ensures atomic writes; protects the following fields
    prefix string     // prefix on each line to identify the logger (but see Lmsgprefix)
    flag   int        // properties
    out    io.Writer  // destination for output
    buf    []byte     // for accumulating text to write
}
```

``` std``` 使用 ```log.New``` 构造出来，三个参数

- ``` io.Writer``` : 任何实现了 ```Writer``` 接口的结构，日志会写到这个里面，``` std``` 的输出位置是 ```os.Stderr``` 
- ```prefix``` : 自定义前缀字符串，std 传入的是空字符串
- ```flag``` : 一些标志位，定制化 prefix，`std` 的是 `LstdFlags`, 默认日志格式中的时间就从这个标志位而来

```go
const (
    Ldate         = 1 << iota     // the date in the local time zone: 2009/01/23
    Ltime                         // the time in the local time zone: 01:23:23
    Lmicroseconds                 // microsecond resolution: 01:23:23.123123.  assumes Ltime.
    Llongfile                     // full file name and line number: /a/b/c/d.go:23
    Lshortfile                    // final file name element and line number: d.go:23. overrides Llongfile
    LUTC                          // if Ldate or Ltime is set, use UTC rather than the local time zone
    Lmsgprefix                    // move the "prefix" from the beginning of the line to before the message
    LstdFlags     = Ldate | Ltime // initial values for the standard logger
)
```

而日志输出位置，自定义前缀字符串，flag 这三者都能够自定义

示例 (这里直接修改的`std`logger, 实际项目中应该自己 New 一个新的 logger)：

```go
func main() {
   var buf bytes.Buffer
   log.SetOutput(&buf)
   log.SetPrefix("[info: ]")
   log.SetFlags(log.LstdFlags | log.Lshortfile)
   log.Println("自定义日志格式")
   //fmt.Print(&buf)
}
```

上述例子把日志输出位置放在了 `bytes.Buffer`，还可以改成 file、stdout、网络... 

前缀加了  [info: ] 字符串

flag 中加了 `Lshortfile`：打印日志语句所处的文件名与位置。

执行上述语句，发现控制台无输出。

因为日志输出位置不再是 ```stderr```. 去掉最后一行的注释，打印 buff，输出为：

```
[info: ]2021/11/09 16:49:27 main.go:14: 自定义日志格式
```

### 优点

- go 内置，使用简单

### 缺点

- 缺少常用的几种日志级别，`Info/debug/error...`
- 缺少日志结构化输出的功能
- 性能

## Go 第三方 log 库

### logrus

[logrus](https://github.com/sirupsen/logrus) : Logrus 是 Go (golang) 的结构化日志记录器，与标准库 log 完全 API 兼容。（19.1 k stars）

#### **Why logrus ?**

1. 比较全的日志级别

```bash
PanicLevel: 记录日志，panic
FatalLevel: 记录日志，程序 exit
ErrorLevel: 错误级别日志
WarnLevel:  警告级别日志
InfoLevel:  关键信息级别日志
DebugLevel: 调试级别
TraceLevel: 追踪级别
```

2. 支持结构化日志

3. 易用性
- 钩子
- `WithField` 
- ...

#### 简单使用

```go
package main

import "github.com/sirupsen/logrus"

func main() {
    logrus.SetLevel(logrus.TraceLevel)    // 在测试环境中设置低等级级别，
    //logrus.SetLevel(logrus.InfoLevel)    // 在生产环境中需要考虑性能，关注关键信息，level 设置高一点
    //logrus.SetReportCaller(true)            // 调用者文件名与位置
    //logrus.SetFormatter(new(logrus.JSONFormatter))    // 日志格式设置成json
    logrus.Traceln("trace 日志")
    logrus.Debugln("debug 日志")
    logrus.Infoln("Info 日志")
    logrus.Warnln("warn 日志")
    logrus.Errorln("error msg")
    logrus.Fatalf("fatal 日志")
    logrus.Panicln("panic 日志")
}

```

考虑到易用性，库一般会使用默认值创建一个对象，包最外层的方法一般都是操作这个默认对象，logrus 与 log 都是这样

结构化前：

```bash
INFO[0000]/Users/xiaohe/go/src/github.com/he2121/log_test/main.go:12 main.main() Info 日志                                      
WARN[0000]/Users/xiaohe/go/src/github.com/he2121/log_test/main.go:13 main.main() warn 日志                                      
ERRO[0000]/Users/xiaohe/go/src/github.com/he2121/log_test/main.go:14 main.main() error msg                                    
FATA[0000]/Users/xiaohe/go/src/github.com/he2121/log_test/main.go:15 main.main() fatal 日志
```

结构化后：

```bash
{"file":"/Users/xiaohe/go/src/github.com/he2121/log_test/main.go:12","func":"main.main","level":"info","msg":"Info 日志","time":"2021-11-09T17:37:27+08:00"}
{"file":"/Users/xiaohe/go/src/github.com/he2121/log_test/main.go:13","func":"main.main","level":"warning","msg":"warn 日志","time":"2021-11-09T17:37:27+08:00"}
{"file":"/Users/xiaohe/go/src/github.com/he2121/log_test/main.go:14","func":"main.main","level":"error","msg":"error msg","time":"2021-11-09T17:37:27+08:00"}
{"file":"/Users/xiaohe/go/src/github.com/he2121/log_test/main.go:15","func":"main.main","level":"fatal","msg":"fatal 日志","time":"2021-11-09T17:37:27+08:00"}
```

### zap

[zap](https://github.com/uber-go/zap) : 快速的、结构化的、分层次的日志记录器 (14k stars)。

#### **why zap?**

1. 比较全的日志级别

2. 支持结构化日志

3. 性能
   ![image-20211109201453244](http://ganghuan.oss-cn-shenzhen.aliyuncs.com/img/image-20211109201453244-2021-11-09.png)

#### 简单使用

Zap提供了两种类型的日志记录器 — `Sugared Logger` 和 `Logger`

`Sugared Logger` 并重性能与易用性，支持结构化和 `printf` 风格的日志记录。

`Logger` 非常强调性能，不提供 printf 风格的 api （减少了 interface{} 与 反射的性能损耗），如下例子：

```go
package main

import "go.uber.org/zap"

func main() {
   // sugared
   sugar := zap.NewExample().Sugar()
   sugar.Infof("hello! name:%s,age:%d", "xiaomin", 20)    // printf 风格，易用性
   // logger
   logger := zap.NewExample()
   logger.Info("hello!", zap.String("name", "xiaomin"), zap.Int("age", 20)) // 强调性能
}
```

```bash
// output 
{"level":"info","msg":"hello! name:xiaomin,age:20"}
{"level":"info","msg":"hello!","name":"xiaomin","age":20}
```

zap 有三种默认配置创建出一个 logger，分别为 `example，development，production`，示例：

```go
package main

import "go.uber.org/zap"

func main() {
   // example
   logger := zap.NewExample()
   logger.Info("example")
   // Development
   logger,_ = zap.NewDevelopment()
   logger.Info("Development")
   // Production
   logger,_ = zap.NewProduction()
   logger.Info("Production")
}
```

```bash
// output
{"level":"info","msg":"example"}
2021-11-09T21:02:07.181+0800    INFO    log_test/main.go:11     Development    
{"level":"info","ts":1636462927.182271,"caller":"log_test/main.go:14","msg":"Production"}

```

可以看出：日志等级，日志输出格式，默认字段都有所差异。

也可以自定义 logger，如以下例子

```go
package main

import (
    "go.uber.org/zap"
    "go.uber.org/zap/zapcore"
    "os"
)

func main() {
    encoder := getEncoder()
    sync := getWriteSync()
    core := zapcore.NewCore(encoder, sync, zapcore.InfoLevel)
    logger := zap.New(core)

    logger.Info("info 日志",zap.Int("line", 1))
    logger.Error("info 日志", zap.Int("line", 2))
}

// 负责设置 encoding 的日志格式
func getEncoder() zapcore.Encoder {
    return zapcore.NewJSONEncoder(zap.NewProductionEncoderConfig())
}

// 负责日志写入的位置
func getWriteSync() zapcore.WriteSyncer {
    file, _ := os.OpenFile("./log.txt", os.O_CREATE|os.O_APPEND|os.O_RDWR, os.ModePerm)
    syncFile := zapcore.AddSync(file)
    syncConsole := zapcore.AddSync(os.Stderr)
    return zapcore.NewMultiWriteSyncer(syncConsole, syncFile)
}
// output
// 创建 log.txt，追加日志
// console 打印日志
//{"level":"info","ts":1636471657.16419,"msg":"info 日志","line":1}
//{"level":"error","ts":1636471657.1643898,"msg":"info 日志","line":2}

```

从 ```New(core zapcore.Core, options ...Option) *Logger``` 出发，需要构造 `zapcore.Core `

1. 通过 ```NewCore(enc Encoder, ws WriteSyncer, enab LevelEnabler) Core``` 方法，又需要传入三个参数
   
   - ```Encoder``` : 负责设置 encoding 的日志格式, 可以设置 json 或者 text结构，也可以自定义json中 key 值，时间格式...
   
   - ```ws WriteSyncer```: 负责日志写入的位置，上述例子往 file 与 console 同时写入，这里也可以写入网络。
   
   - ```LevelEnabler```: 设置日志记录级别

2. `options` 是一个实现了```apply(*Logger)``` 方法的接口，可以扩展很多功能
   
   - 覆盖 core 的配置
   - 增加 hook
   - 增加键值信息
   - error 日志单独输出

## 总结

首先介绍了 Go 标准库 log的使用以及自定义过程，总结了 log 的优缺点，由此引入了第三方库 logrus 与 zap 的话题，两个库都十分优秀与强大，十分推荐使用