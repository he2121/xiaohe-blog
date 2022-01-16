---
title: "Golang 逃逸分析"
author: "小贺"
date: 2022-01-12T16:10:12+08:00
tags: ["golang"]
---

## 问题

1. Go 中 New/Make 出来的对象存在哪，栈还是堆？
2. 下面 FooA 与 FooB 函数哪个性能高一点，为什么(来自 [stack overflow](https://stackoverflow.com/questions/41178729/why-passing-pointers-to-channel-is-slower))？

```go
package main

import (
	"fmt"
	"time"
)

type User struct {
	Name string
	Age int
}

func main() {
	fooA()
	fooB()
}

func fooA()  {
	start := time.Now()
	var c = make(chan User, 1024)
	for i := 0; i < 1024; i ++ {
		user := User{Name: "he", Age: 25}
		c <- user
	}
	since := time.Since(start)
	fmt.Println("FooA User", since.Microseconds())
}

func fooB()  {
	start := time.Now()
	var c = make(chan *User, 1024)
	for i := 0; i < 1024; i ++ {
		user := &User{Name: "he", Age: 25}
		c <- user
	}
	since := time.Since(start)
	fmt.Println("FooB *User ", since.Microseconds())
}
```

3. 参数 值传递 VS 指针传递

## 问题解答

#### 1. Go 中 New/Make 出来的对象存在哪，栈还是堆？

Linux 进程布局：

1. 代码段：存放程序可执行的指令的区域
2. 数据段：存放程序中已初始化的全局变量与静态变量的一块内存区域
3. BSS 段： 放程序中未初始化的全局变量与静态变量的一块内存区域
4. **栈：** 栈保存了一个函数调用所需要的维护信息
5. **堆： **堆是用于存放进程运行中被动态分配的内存段，它的大小并不固定，可动态扩张或缩减

在 C/C++ 语言等无垃圾回收机制的语言上，New 出来的对象一定在堆上，而且必须要程序员用手动 free 掉，不然造成内存泄漏。

而在 Go/Java 等有垃圾回收机制的语言上，分配在堆上的对象不需要程序员进行管理。但是如果堆上的对象太多会导致垃圾回收太久，影响主程序的性能。

Go 的编译器做了一个**优化**: 优先把对象分配在栈上，不能分配在栈上的对象再分配到堆上。由于栈空间随函数调用结束自动回收，不需要垃圾回收，这个优化通过减少垃圾回收简介提高了程序的性能。

所以，Go 中 New/Make 的对象可能在栈上又可能在堆上。而编译器决定这个对象分在栈上还是堆上的过程，也即对象能否从栈逃逸到堆的分析过程，被称作**逃逸分析**。

#### 2. FooA 与 FooB 函数哪个性能高一点，为什么？

(来自 [stack overflow](https://stackoverflow.com/questions/41178729/why-passing-pointers-to-channel-is-slower))

FooB 传递的指针，但其性能会差一些。这是由于其 New 出来的 &User 逃逸到堆上，垃圾回收导致的，具体看 [stack overflow](https://stackoverflow.com/questions/41178729/why-passing-pointers-to-channel-is-slower) 原回答

#### 3. 值传递 VS 指针传递

先根据你的目的：

1. 你想保证参数不被修改 - 值传递
2. 参数结构很大 - 指针传递
3. map、slice 等引用类型 - 值传递

在一般情况下，推荐值传递。因为由于逃逸分析的存在，值传递情况下性能一般比指针传递性能要高。

## Golang 逃逸分析

以下命令查看 Go 编译器的逃逸分析结果，一般搭配 `-l` 禁止内联使用

```bash
go build -gcflags="-m"
```

上述 FooB 的例子: 

```bash
$ go build -gcflags='-m -l' main.go 
# command-line-arguments
./main.go:33:11: &User{...} escapes to heap
./main.go:37:13: ... argument does not escape
./main.go:37:14: "FooB *User " escapes to heap
./main.go:37:47: since.Microseconds() escapes to heap
```

### 逃逸分析基本原则

1. 指向堆栈对象的指针不能存储在堆中
2. 指向堆栈对象的指针不能超过那个对象的寿命

这是逃逸分析算法的两点原则与工作内容，当然还有一些其它情况决定对象需要直接分配在堆上。

### 发生逃逸的情形

#### 1. 指针逃逸, 将对象指针作为函数返回值

如下面例子，违背了原则 2：指向堆栈对象的指针不能超过那个对象的寿命

```Go
type User struct {
  Name string
	Age int
}
func main() {
	GetUser()
}

func GetUser() *User {
  user := User{}
	return &user
}
```

由于 *user 这个指针被传到 main 函数中使用，而 User 这个对象在 GetUser 方法中 New 出来，如果不逃逸到堆上，则指针的寿命比对象长。因此需要逃逸到堆上，让对象的寿命比其指针长。因此上述例子发生了逃逸

```bash
$ go build -gcflags='-m -l' main.go 
# command-line-arguments
./main.go:13:2: moved to heap: user
```

#### 2. 指针逃逸，将对象指针赋值到一个堆对象中

如下面例子，违背了原则 1：指向堆栈对象的指针不能存储在堆中

```go
type User struct {
  Name string
	Age int
	Car *Car
}
type Car struct {
}

func main() {
	user := GetUser()
	car := Car{}
	user.Car = &car
}

func GetUser() *User {
	return &User{}
}
```

`GetUser` 返回的是一个逃逸的对象，即其已经存在堆中。我们给其一个字段赋值 `*Car`,  如果把 Car 对象分配在栈中，发生了指向 栈中对象（Car）的指针存储在堆中对象中（User） ，违背了原则 1，所以 Car 对象需要逃逸到堆。[`fmt.Println`](https://github.com/golang/go/issues/8618) 也是这个原因存在逃逸

```bash
$ go build -gcflags='-m -l' main.go 
# command-line-arguments
./main.go:20:9: &User{} escapes to heap
./main.go:15:2: moved to heap: car
```

#### 3. 大对象分配在堆上

经过测试，大于 64 KB 的对象分在在堆上，小于或等于 64 KB 的对象分配在堆上。

```go
func main() {
	_ = make([]byte, 1 << 16)	// 1 << 16 B 的 slice 没有逃逸，分配在栈上
}

func main() {
	_ = make([]byte, 1 << 16 + 1)	// 1 << 16 + 1 B 的 slice 逃逸到堆上
}
// output 1
// $ go build -gcflags='-m -l' main.go 
// # command-line-arguments
// ./main.go:11:10: make([]byte, 1 << 16) does not escape

// output 2
// $ go build -gcflags='-m -l' main.go 
// # command-line-arguments
// ./main.go:11:10: make([]byte, 1 << 16 + 1) escapes to heap

```

这里测试出的 64 KB 与 当前 Golang 内存分配器讲的 32 KB 的大对象直接分配在堆中**不符**， 原因还**待查**。

#### 3. 动态分配空间

在编译期间难以确定其分配内存的大小，需要在运行期间确定，动态分配 slice 的长度。

```go
func main() {
	l := 10
	_ = make([]byte,l)
}
```

上面例子，`l` 是变量，编译器不确定其分配内存的大小，所以分配到堆上。

```bash
$ go build -gcflags='-m -l' main.go 
# command-line-arguments
./main.go:13:10: make([]byte, l) escapes to heap
```

#### 4. slice, map, chan 上的指针指向的对象

如问题2 中的 FooB，如下面例子

```go
func main() {
	users := make([]*User, 1)
	user := User{}
	users[0] = &user
}
```

users slice 没有逃逸， user 结构分配了堆上

```bash
$ go build -gcflags='-m -l' main.go 
# command-line-arguments
./main.go:17:2: moved to heap: user
./main.go:16:15: make([]*User, 1) does not escape
```

#### 5. 闭包函数引用对象

如下例子

```go
func add() func() int {
	a := 1
	return func() int {
		a ++
		return a
	}
}
```

其实这里违背了原则 2：指向堆栈对象的指针不能超过那个对象的寿命。这里的例子如果不把 `a` 逃逸到 堆上，会导致闭包函数对 `a`的引用超过 `a`的生命周期。

```bash
$ go build -gcflags='-m -l' main.go 
# command-line-arguments
./main.go:16:2: moved to heap: a
./main.go:17:9: func literal escapes to heap

```

#### Golang 编译器逃逸分析算法

[源代码](https://github.com/golang/go/blob/master/src/cmd/compile/internal/escape/escape.go)

这里简要说下过程（没仔细看），算法基于以上两个基本原则进行的。

输入：编译器解析用户程序代码得到的抽象语法树（AST）

过程：

1. 首先，我们构建一个有向加权图，其中顶点（称为 "位置"）代表由语句和表达式分配的变量，边代表变量之间的分配（其权重代表寻址/引用计数）
2. 接下来，我们在图中寻找那些可能违反上述基本原则。如果一个变量 v 的地址被存储在堆或其他地方，可能会超过它的寿命，那么 v 就被标记为需要堆分配
3. 为了支持函数调用间分析，我们还记录了从每个函数的参数到被存到堆以及作为返回值的数据流。这些信息被总结为 "参数标签"，在静态调用使用，以提高对函数参数的逃逸分析

这部分了解下大概意思就行，我也没研究

## 总结

本文从 3 个与逃逸分析相关的问题出发，介绍了什么是逃逸分析，逃逸分析的作用。后面介绍了 Golang 逃逸分析的相关知识，特别是对象逃逸到堆的 5 个常见场景。总之逃逸分析与垃圾回收、程序性能息息相关，了解它能让我们写出对程序性能友好的代码。

## 参考

1. https://medium.com/a-journey-with-go/go-introduction-to-the-escape-analysis-f7610174e890
2. https://mayurwadekar2.medium.com/escape-analysis-in-golang-ee40a1c064c1
3. https://stackoverflow.com/questions/41178729/why-passing-pointers-to-channel-is-slower
4. https://qcrao91.gitbook.io/go/bian-yi-he-lian-jie/tao-yi-fen-xi-shi-zen-mo-jin-hang-de

