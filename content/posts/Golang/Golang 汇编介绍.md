---
title: "Golang 汇编介绍"
author: "小贺"
date: 2021-12-22T16:10:12+08:00
tags: ["golang"]
---

> 在阅读 Golang 源代码时，总是被其中的汇编代码卡住，读起来不流畅。今天来简要了解下 Golang 中的汇编语言。

## 汇编分类

#### 按指令集架构分类（针对 CPU）

1. **x86汇编(32bit)**:这种架构常被称为`i386`, `x86`

2. **x86汇编(64bit)**, 这种架构常被称为 `AMD64`, `Intel64`, `x86-64`, `x64`, 它是 AMD 设计的, 是 x86 架构的 64 位扩展, 后来公开

3. **ARM汇编**, ARM处理器由于高性能, 低耗电, 常用于嵌入式, 移动设备.

4. ...

#### 按汇编格式分类（针对人的阅读习惯）

1. **Intel 格式**
2. **AT&T 格式**

平时我们说 golang 中汇编属于 **plan9 风格**，是按第二种方式分类的，其阅读风格（符号）与 Intel 与 AT&T 都有不同。plan9 汇编作者是 unix 操作系统的同一批人，bell 实验室所开发的。

Go汇编语言是基于 [plan9 汇编](https://www.symbolcrash.com/2021/03/02/go-assembly-on-the-arm64/)，但是现实世界还有这么多不同架构的 CPU 在这。所以 golang 汇编在 plan9 风格下，同一个方法还有不同指令集架构的多种实现。

## 在哪能看到 Golang 汇编代码

1. Golang 源代码中，如`src/runtime/asm_amd64.s` ，`src/math/big/`    ...
2. `go tool compile -S main.go`,把自己编写的代码编译成汇编代码。如：在我的 Mac Intel 机器上，`amd64`的架构，汇编代码生成如下:

```bash
$ cat main.go 
package main

func main() {
        a, b := 0, 0
        println(a + b)
}
$ go tool compile -S main.go 
"".main STEXT size=66 args=0x0 locals=0x10 funcid=0x0
        0x0000 00000 (main.go:3)        TEXT    "".main(SB), ABIInternal, $16-0
        0x0000 00000 (main.go:3)        CMPQ    SP, 16(R14)
        0x0004 00004 (main.go:3)        PCDATA  $0, $-2
        0x0004 00004 (main.go:3)        JLS     57
        0x0006 00006 (main.go:3)        PCDATA  $0, $-1
        0x0006 00006 (main.go:3)        SUBQ    $16, SP
        0x000a 00010 (main.go:3)        MOVQ    BP, 8(SP)
        0x000f 00015 (main.go:3)        LEAQ    8(SP), BP
        0x0014 00020 (main.go:3)        FUNCDATA        $0, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
        0x0014 00020 (main.go:3)        FUNCDATA        $1, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
        0x0014 00020 (main.go:5)        PCDATA  $1, $0
        0x0014 00020 (main.go:5)        CALL    runtime.printlock(SB)
        0x0019 00025 (main.go:5)        XORL    AX, AX
...

```

## Go 汇编基础语法

### 1. 寄存器

#### 通用寄存器

寄存器与物理机架构相关, 不同的架构有不同的物理寄存器。

在 `amd64` 架构上提供了 16 个通用寄存器给用户使用

plan9 汇编语言提供了如下映射，在汇编语言中直接引用就可使用物理寄存器了。

| amd64 | rax | rbx | rcx | rdx | rdi | rsi | rbp | rsp | r8  | r9  | r10 | r11 | r12 | r13 | r14 | rip |
| ----- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Plan9 | AX  | BX  | CX  | DX  | DI  | SI  | BP  | SP  | R8  | R9  | R10 | R11 | R12 | R13 | R14 | PC  |

如上文的例子中使用到了：`SP`,`AX`,`R14`,`BP`

#### 虚拟寄存器

Go 汇编引入了 4 个 虚拟寄存器

- `FP`: Frame pointer: arguments and locals.      帧指针，快速访问函数的参数和返回值
- `PC`: Program counter: jumps and branches.   程序计数器，指向下一条指令的地址。在 `amd64` 其实就是 rip 寄存器
- `SB`: Static base pointer: global symbols.          静态基址指针，全局符号。
- `SP`: Stack pointer: the highest address within the local stack frame. 栈指针, 指向局部变量

**用法**：

- FP：`0(FP)` 表示第一个**参数**，`8(FP)` 表示第二个参数(AMD64 架构)。`first_arg+0(FP)` 表示把第一个参数地址绑定到符号 first_arg

- SP：`localvar0-8(SP)`  在 plan9 中表示函数中第一个**局部变量**。物理寄存器中也有 SP，硬件 SP 才是真正表示 栈顶位置。所以为了**区分 SP** 到底是指硬件 SP 还是指虚拟寄存器。plan9 代码中需要以特定的格式来区分。eg：`symbol+offset(SP)` 表示虚拟寄存器 SP。`offset(SP)` 则表示硬件 SP。如上述例子中的 `8(SP)` 指的是硬件 SP

- PC：除个别跳转治理，一般用不到

- SB：表示全局内存起点。`foo(SB)` 表示符号 foo 作为内存地址使用。这种形式用于声明全局函数、数据。`foo+4(SB)`表示 foo 往后 4 字节的地址。`<> `限制符号只能在当前源文件使用

从网上偷的图：

![图源《Go语言高级编程》](https://blogimagee.oss-cn-beijing.aliyuncs.com/images/ch3-3-arch-amd64-02.ditaa.png)

### 2. 指令

#### 1. 变量声明

**格式:**  使用 `DATA` 与 `GLOBL` 来声明一个全局变量

```GO
DATA symbol+offset(SB)/width, value
GLOBL symbol(SB), flag, $size
```

**表示意义**：

- DATA 部分: 对 `symbol` 变量中的字节赋值，把 `offset` 到 `offset + width` 位置的字节赋值为 `value`。

- GLOBL 部分：必须在 DATA 后，表示声明了一个大小为`size` 的全局变量`symbol`。[`flag`](https://go.dev/doc/asm#directives)代表变量一些属性，如 `RODATA`指只读。在 GLOBL 中加入 `<>`, 如 `GLOBL bio<>(SB), RODATA, $16`  也是表示这个全局变量只在本文件中生效。

**实际例子：**

```GO
// src/runtime/asm_amd64.s, 这里声明的 argc，argv 是 Go 程序的入参
DATA _rt0_amd64_lib_argc<>(SB)/8, $0
GLOBL _rt0_amd64_lib_argc<>(SB),NOPTR, $8
DATA _rt0_amd64_lib_argv<>(SB)/8, $0
GLOBL _rt0_amd64_lib_argv<>(SB),NOPTR, $8
```

`NOPTR` 这个表示不是指针，不需要垃圾回收扫描

**局部变量**：其在栈帧中，不需要声明。直接依靠 `offset`取出使用。例如`0(FP)` 代表函数第一个参数，`localvar0-8(SP)` 函数中第一个局部变量。

#### 2. 函数声明

**格式：**

`TEXT pkgname·funname(SB),flag,$framesize-argsize`

**表示意义：**

`pkgname` :可以省略，最好省略。不然修改包名还要级联修改；

`funname`: 声明的函数名

`flag`: 标志位，如 `NOSPLIT`，我们知道 Go Runtime 会追踪每个 stack 的使用情况，然后动态自增。而`NOSPLIT `标志位禁止检查，节省开销，但是写程序的人要保证这个函数是安全的。

`framesize`: 函数栈帧大小 = 局部变量 + 调用其它函数参数空间的总大小

`argsize`: 一些参考资料说这里是 **参数+返回值**大小，但在实验中已有些许差异。这个应该和 GO 1.7 的更新有关，[GO1.7 基于寄存器的调用规约](https://go.googlesource.com/proposal/+/master/design/40724-register-calling.md)       [GO 1.7 的优化](https://xargin.com/go1-17-new-calling-convention/)

- GO 1.7 之前 参数+返回值都存在栈帧中
- GO 1.7 更新后， 优先使用 9 个 通用寄存器传递参数与返回值，超出部分再存在栈中。并且寄存器中返回值会覆盖参数中的值
- 参数 + 返回值少于 9 个，argsize 值是参数的大小
- 返回值 > 9 个，argsize = 参数大小 + 返回值超出 9 个的部分

**实际例子：**

```bash
$ cat main.go
package main

func main() {
}

func add(a int64, b int64) int64 {
        return a + b
}

$ go tool compile -S main.go
"".main STEXT nosplit size=1 args=0x0 locals=0x0 funcid=0x0
...
"".add STEXT nosplit size=4 args=0x10 locals=0x0 funcid=0x0
        0x0000 00000 (main.go:6)        TEXT    "".add(SB), NOSPLIT|ABIInternal, $0-16
        0x0000 00000 (main.go:6)        FUNCDATA        $0, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
        0x0000 00000 (main.go:6)        FUNCDATA        $1, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
        0x0000 00000 (main.go:6)        FUNCDATA        $5, "".add.arginfo1(SB)
        0x0000 00000 (main.go:7)        ADDQ    BX, AX
        0x0003 00003 (main.go:7)        RET
        ...

```

上述例子：函数内不需存放局部变量，framesize = 0， 两个 int64 的参数，argsize = 16

#### 3. 常见操作指令

[自主查询链接](http://68k.hax.com/)

下面介绍下常见的

##### 1. 数据搬运

- `MOV` 指令：其后缀表示搬运长度, `$NUM` 表示具体的数字，如下面例子

```go
MOVB $1, DI          // 1 byte,  DI=1
MOVW $0x10, BX       // 2 bytes, BX=10
MOVD $1, DX          // 4 bytes, DX=1
MOVQ $-10, AX     // 8 bytes, AX=-10
```

- `LEA`, 将有效地址加载到指定的地址寄存器中

```go
// ret+24(FP) 这代表了第三个函数参数，是个地址
LEAQ    ret+24(FP), AX    // 把 ret+24(FP) 地址移到 AX 寄存器中
```

##### 2. 计算指令

- `ADD`，`SUB`,`IMULQ`，如下面例子

```go
ADDQ  AX, BX   // BX += AX
SUBQ  AX, BX   // BX -= AX
IMULQ AX, BX   // BX *= AX
```

- 可以利用计算指令来调整栈空间，我们知道 `SP` 指向栈顶位置，调整 `SP`中的值即可。

```go
// 栈空间: 高地址向低地址
SUBQ $0x18, SP // 对 SP 做减法，为函数分配函数栈帧
ADDQ $0x18, SP // 对 SP 做加法，清除函数栈帧
```

##### 3. 条件跳转/无条件跳转

- `JMP`， `JZ`,`JLS` ...

```go
// 无条件跳转
JMP addr   // 跳转到地址，地址可为代码中的地址，不过实际上手写不会出现这种东西
JMP label  // 跳转到标签，可以跳转到同一函数内的标签位置
JMP 2(PC)  // 以当前指令为基础，向前/后跳转 x 行
JMP -2(PC) // 同上

// 有条件跳转
JZ target // 如果 zero flag 被 set 过，则跳转
JLS num        // 如果上一行的比较结果，左边小于右边则执行跳到 num 地址处
```

##### 4. 其它

- 比较：`CMP`, 与挑战指令搭配使用
  
  ```go
  CMPQ    BX, $0    // 比较与 BX 与 0 的大小
  JNE    3(PC)            // 左边小于右边则执行跳到当前 PC 指令后第三条指令的位置
  ```

- 位运算： `AND`,`OR`,`XOR`

## 总结

经过这篇文章，相信你已经能大致读懂一些简单的汇编程序了。这里推荐几个源代码的汇编阅读。

- Go 程序的起点：`src/runtime/asm_amd64.s` 中的 `rt0_go(SB)` 函数
- Go 原子包：`src/runtime/internal/atomic_amd64.s` 中的 `Case` 函数

## 参考

1. https://segmentfault.com/a/1190000039978109
2. https://go.dev/doc/asm
3. https://medium.com/martinomburajr/go-tools-the-compiler-part-1-assembly-language-and-go-ffc42cbf579d
4. https://xargin.com/go-and-plan9-asm/
5. https://kcode.icu/posts/go/2021-03-20-go-%E4%BD%BF%E7%94%A8%E7%9A%84-plan9-%E6%B1%87%E7%BC%96%E8%AF%AD%E8%A8%80%E5%88%9D%E6%8E%A2/
6. https://mioto.me/2021/01/plan9-assembly/
7. https://www.symbolcrash.com/2021/03/02/go-assembly-on-the-arm64/
