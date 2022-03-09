---
title: "Golang 的骚操作：go:linkname"
author: "小贺"
date: 2021-12-02T17:10:12+08:00
tags: ["golang"]
---

## 背景

1. 在看源码时，一些源码方法没有方法体，难道说明这些方法为空？例如：[`time.Now`](https://github.com/golang/go/blob/5786a54cfe34069c865fead1b6d9c9e3485a40a5/src/time/time.go#L1073) 调用的 [`now()`](https://github.com/golang/go/blob/5786a54cfe34069c865fead1b6d9c9e3485a40a5/src/time/time.go#L1057), [`time.Sleep`](https://github.com/golang/go/blob/5786a54cfe34069c865fead1b6d9c9e3485a40a5/src/time/sleep.go#L9) , [`reflect.makechan`](https://github.com/golang/go/blob/5786a54cfe34069c865fead1b6d9c9e3485a40a5/src/reflect/value.go#L3383)

```go
// Provided by package runtime.
func now() (sec int64, nsec int32, mono int64)

func Sleep(d Duration)

func makechan(typ *rtype, size int) (ch unsafe.Pointer)
```

2. 在写代码时，如果我们想**使用**别的包下**没有导出的方法或者变量**时，怎么操作

## go:linkname 的用法

实际上，上述提到的三个没有方法体的方法，其实现都在 `src/runtime`包下

1. [time.now](https://github.com/golang/go/blob/5786a54cfe34069c865fead1b6d9c9e3485a40a5/src/runtime/timestub.go#L18) timestub.go 文件中

```go
//go:linkname time_now time.now
func time_now() (sec int64, nsec int32, mono int64) {
    sec, nsec = walltime()
    return sec, nsec, nanotime()
}
```

2. [time.Sleep](https://github.com/golang/go/blob/5786a54cfe34069c865fead1b6d9c9e3485a40a5/src/runtime/time.go#L177) time.go 文件中

```go
// timeSleep puts the current goroutine to sleep for at least ns nanoseconds.
//go:linkname timeSleep time.Sleep
func timeSleep(ns int64) {
   if ns <= 0 {
      return
   }

   gp := getg()
   t := gp.timer
   if t == nil {
      t = new(timer)
      gp.timer = t
   }
   t.f = goroutineReady
   t.arg = gp
   t.nextwhen = nanotime() + ns
   if t.nextwhen < 0 { // check for overflow.
      t.nextwhen = maxWhen
   }
   gopark(resetForSleep, unsafe.Pointer(t), waitReasonSleep, traceEvGoSleep, 1)
}
```

3. [reflect.makechan](https://github.com/golang/go/blob/5786a54cfe34069c865fead1b6d9c9e3485a40a5/src/runtime/chan.go#L59)

```go
//go:linkname reflect_makechan reflect.makechan
func reflect_makechan(t *chantype, size int) *hchan {
    return makechan(t, size)
}
```

我们可以看到具体实现都具有 `go:linkname`, 可以推测就是这个东西把两个不同包下的函数链接到一起了

### 官方介绍

```go
//go:linkname localname [importpath.name]
```

这个特殊的指令的作用域并不是紧跟的下一行代码，而是同一个包下生效。`//go:linkname`告诉 Go 的编译器把本地的(私有)变量或者方法`localname`链接到指定的变量或者方法`importpath.name`。简单来说，`localname`  `import.name`指向的变量或者方法是同一个。因为这个指令破坏了类型系统和包的模块化原则，只有在引入 `unsafe` 包的前提下才能使用这个指令。

### 使用实例

文件结构如下图所示, `inner` 包模拟 go 源码包/外部引入包，`outer`包需要调用`inner`包中的方法, `main.go` 调试

```bash
 tree
.
├── go.mod
├── inner
│   └── inner.go
├── main
├── main.go
└── outer
    ├── 1.s
    └── outer.go

```

#### 空body方法复现：

模拟方法无 body方法，真实实现在另外一个包中的案例

outer.go

```go
package outer

import (
    _ "github.com/he2121/demos/linkname_example/inner"    // 真实方法在 inner 包，必须引用这个编译器才能找到链接
)

func Hello()
```

如果出现：`missing function body ` 在包内加一个 `.s`文件。因为`go build`默认加会加上`-complete`参数，加这个 .s 可绕开这个限制。 

inner.go

```go
package inner

import (
    _ "unsafe"
)

//go:linkname hello github.com/he2121/demos/linkname_example/outer.Hello
func hello()  {
    println("hello")
}

```

main.go

```go
package main

import "github.com/he2121/demos/linkname_example/outer"

func main()  {
    outer.Hello()
}
// output:
// hello
```

这就是 go 源码中这么多没有 body 的方法的原因了，一般到 `src/runtime`包中搜索就能发现 `//go:linkname`的实现。

#### 实际使用

实现中，如果我们有一些骚操作需要使用源代码/第三方包中的未导出变量/方法，就可使用 `//go:linkname`实现, 如下例子

outer.go: 使用 inner 包中的未导出变量/方法

```go
package outer

import (
    _ "unsafe"

    _ "github.com/he2121/demos/linkname_example/inner"
)

//go:linkname A github.com/he2121/demos/linkname_example/inner.a
var A int

//go:linkname Hello github.com/he2121/demos/linkname_example/inner.hello
func Hello()
```

inner.go

```go
package inner

var a = 100

func hello()  {
    println("hello")
}

```

main.go

```go
package main

import "github.com/he2121/demos/linkname_example/outer"

func main()  {
    outer.Hello()
    println(outer.A)
}
// output
// hello
// 100

```

[示例 repo](git@github.com:he2121/demos.git)

## 总结

1. 能读懂多一点源码
2. 掌握获取并使用其它包中未导出变量/方法的骚操作

## 参考

1. https://cloud.tencent.com/developer/article/1491065
2. https://blog.csdn.net/lastsweetop/article/details/78830772
3. https://studygolang.com/articles/15842