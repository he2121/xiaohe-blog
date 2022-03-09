---

title: "Golang channel 原理介绍"
author: "小贺"
date: 2021-11-29T17:10:12+08:00
tags: ["golang"]

---

### 问题

1. 从关闭的 channel 读取数据，如以下情况输出什么。如果是发送数据，再关闭一次 channel的操作呢？怎么判断 channel 的是否关闭

```go
func main() {
    ch := make(chan int,2)
    ch <- 1
    close(ch)
    num1, ok1 := <- ch
    num2,ok2 := <- ch
    println(num1,ok1)
    println(num2,ok2)
}
```

2. channel 底层实现
3. 无缓冲 channel 与有缓存 channel 区别
4. channel VS mutex

## channel 简介

> Do not communicate by sharing memory; instead, share memory by communicating.

不要通过共享内存来通信，而要通过通信来实现内存共享。这是 CSP（Communicating Sequential Processes）的思想，也是 Go 并发设计上的哲学。CSP 认为如果编程语言中把侧重点放在 processes 间的通信，那么并发编程会变得很简单，而 Go 中 channel 就是通信实体的实现，可以看作成一个协程间的消息队列。

## channel 使用

对 channel 的操作有四种：

- 创建 channel：`ch := make(chan int)` `ch:=make(chan string, 10) `
- 向 channel 发送消息: `ch <- num`
- 从 channel 接受消息: `num <- ch`, `num, ok := <- ch`
- 关闭 channel: `close(ch)`
  一些对 channel 操作可能出现的一些边界条件情况如下图所示，其中 full channel 与 empty channel 在无缓冲和有缓冲 channel 下e的概念稍有不同，在下文中解释

![mermaid-diagram-20211130225330](http://ganghuan.oss-cn-shenzhen.aliyuncs.com/img/mermaid-diagram-20211130225330-2021-11-30.png)

## channel 实现

### 数据结构

channel 的底层数据结构如下代码所示，个人认为主要由**一个 buf 数组 + 2 个 goroutine 链表**组成

```go
// src/runtime/chan.go
type hchan struct {
    qcount   uint           // chan 中元素个数
    dataqsiz uint           // chan 底层循环队列的长度
    buf      unsafe.Pointer // 指向 循环队列的指针
    elemsize uint16                    // chan 中元素大小
    closed   uint32                    // 是否关闭的状态位
    elemtype *_type                 // 元素类型
    sendx    uint                   // 已发送元素在循环队列中的索引
    recvx    uint                   // 已接受元素在循环队列中的索引
    recvq    waitq                  // 等待接受数据的 goroutine 队列
    sendq    waitq                  // 等待发送数据的 goroutine 列队

    lock mutex    // 保护上述字段
}

```

- `buf`: channel 流动的缓冲区，实际上是一个指向循环队列的指针，数据有无缓冲的 channel 的区别在这。无缓冲的 channel 即这个 buf 在初始化时没有分配内存。
- `recvq` 与 `sendq`: 阻塞的 goroutine 链表，分别是等待接受和等待发送数据的 goroutine 链表
- 其它字段基本是描述 buf 状态的：如 buf 容量(`dataqsiz`)，buf 具有的元素个数(`qcount`)，buf 中一个元素所占内存大小（`elemsize`），`sendx` 与 `recvx`描述 buf 循环数组中可向channel发送消息与接受channel 中消息的索引位置 ...

附上一个 channel 结构的网图供参考

![深度解密Go语言之channel - Stefno - 博客园](https://user-images.githubusercontent.com/7698088/61179068-806ee080-a62d-11e9-818c-16af42025b1b.png)

### 向 channel 发送消息

- `ch <- num`

一般的向 channel 发送消息流程如下图:

![mermaid-diagram-20211130230713](http://ganghuan.oss-cn-shenzhen.aliyuncs.com/img/mermaid-diagram-20211130230713-2021-11-30.png)

> 当然实际代码稍微复杂一些：在 channel 与 `select`,`case` 一起使用时，case 语句需要即时收到返回值，不能阻塞等待。因此需要一个非阻塞的模式，在 channel 为 full 时，不等待直接返回结果。

直接上代码，实现起来也比较简单，基本是入队出队操作，有兴趣的可以看一下

```go
// src/runtime/chan.go
ep 是发送数据的指针, block 表示是否可以把当前协程阻塞，放到 sendq 队列，同步等待结果，一般都是true。只有 select 时，不阻塞协程，直接返回。返回值表示是否成功发送数据
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
  // channel 为 nil
  if c == nil {
    // select 非阻塞，直接返回 false
        if !block {
            return false
        }
    // 阻塞协程
        gopark(nil, nil, waitReasonChanSendNilChan, traceEvGoStop, 2)
        throw("unreachable")
    }
  // 省略一些...

  // select 非阻塞,如果 channel 没有关闭, channel 为 full：有缓冲（buf 满了），无缓冲（recvq 为空）
  if !block && c.closed == 0 && full(c) {
        return false
    }
  // ... 省略了一些...
  // 如果 channel 关闭了，panic
  if c.closed != 0 {
        unlock(&c.lock)
        panic(plainError("send on closed channel"))
    }
  // 若 receq 不为空，把 receq 队列取一个 goroutine，直接把其要发送的消息ep发到取到的 goroutine 中。存在两种情况
  // 1. 这是无缓冲 channel发送数据的过程，
  // 2. 有缓冲 channel receq 不为空，则说明其 buf 数组为空，
    if sg := c.recvq.dequeue(); sg != nil {
        send(c, sg, ep, func() { unlock(&c.lock) }, 3)
        return true
    }
  // 有缓冲 channel，且其 buff 数组还没满，把 ep 数据放到 buf 中
  if c.qcount < c.dataqsiz {
        // 返回sendx 指向的地址
        qp := chanbuf(c, c.sendx)
    // 把 ep 中的消息 copy 到 buf 中
        typedmemmove(c.elemtype, qp, ep)
    // sendx count... 的修改
        c.sendx++
        if c.sendx == c.dataqsiz {
            c.sendx = 0
        }
        c.qcount++
        unlock(&c.lock)
        return true
    }
  // 以上都不成功，阻塞当前协程
  // ...
  c.sendq.enqueue(mysg)    // 当前协程入队 sendq
  // ...
  gopark(chanparkcommit, unsafe.Pointer(&c.lock), waitReasonChanSend, traceEvGoBlockSend, 2)    // 阻塞
  // ...
}
```

这里解释下 full/empty channel。

| Channel 类型 | Full                   | Empty                  |
|:----------:|:----------------------:|:----------------------:|
| 无缓冲        | reveq 队列为空：没有等待接受消息的协程 | sendq 队列为空：没有等待发送消息的协程 |
| 有缓冲        | buf 数组已满               | buf 数组为空               |

**总结**

- 无缓冲 channel 发送消息是**直接将消息从发送者的栈拷贝到接收者的栈**
- 有缓冲 channel发送消息是**将消息拷贝到 channel buf 中**

### 从 channel 中接受消息

- `num <- ch`
- `num,ok <- ch`
  第二个布尔返回值代表着这次通信是否成功，只有当 channel 是关闭状态并且 channel 是 Empty 才返回 false，也就是说即使channel 已经关闭，但如果其还有消息在 buf 中，返回的布尔值是 true。

向 channel 接受消息的流程与发送消息类似，整体如下图所示：

> 此示意图也省略了考虑不阻塞的场景（与`select` ,`case`）

![image-20211130231739724](http://ganghuan.oss-cn-shenzhen.aliyuncs.com/img/image-20211130231739724-2021-11-30.png)

代码详情如下

```go
// ep: 接受数据，block：是否阻塞式接受, selected: 是否有返回值(select 语句 并且 channel empty 无返回值) received: 此返回值是否是正常由发送者发送过来的数据
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
  // channel 为空
    if c == nil {
    // 非阻塞直接返回 false false
        if !block {
            return
        }
    // 阻塞协程
        gopark(nil, nil, waitReasonChanReceiveNilChan, traceEvGoStop, 2)
        throw("unreachable")
    }

    // 非阻塞快速失败
    if !block && empty(c) {
        // channel 没有关闭，返回 false,false
        if atomic.Load(&c.closed) == 0 {
            return
        }
    // channel 已经关闭, 需要返回零值，因此返回:true，false
        if empty(c) {
            if ep != nil {
                typedmemclr(c.elemtype, ep)
            }
            return true, false
        }
    }
    // ...
  // 得操作 buf 或者 sendq 队列了，并发需要加锁
    lock(&c.lock)

  // 如果 channel 已经关闭, 并且 buf 为空，返回零值
    if c.closed != 0 && c.qcount == 0 {
        unlock(&c.lock)
        if ep != nil {
            typedmemclr(c.elemtype, ep)
        }
        return true, false
    }

  // 发送队列不为空, 有两种情况sendq 不为空 
  // 1. 无缓冲 channel，直接将队首的消息 copy 到 ep 指向的数据中
    // 2. 有缓冲 channel，但是其 buf 已满。其中操作需要把 buf 队首数据 copy 到 ep 指向的数据， sendq 队首出队，将其发送的消息copy 到 buf 中
    if sg := c.sendq.dequeue(); sg != nil {
        recv(c, sg, ep, func() { unlock(&c.lock) }, 3)
        return true, true
    }

    // buf 不为空，将 buf 队首数据 copy 到 ep 
    if c.qcount > 0 {
        // Receive directly from queue
        qp := chanbuf(c, c.recvx)
        // ....
        if ep != nil {
            typedmemmove(c.elemtype, ep, qp)
        }
        typedmemclr(c.elemtype, qp)
        c.recvx++
        if c.recvx == c.dataqsiz {
            c.recvx = 0
        }
        c.qcount--
        unlock(&c.lock)
        return true, true
    }
    // 非阻塞直接返回
    if !block {
        unlock(&c.lock)
        return false, false
    }

    // 阻塞
    gp := getg()
    mysg := acquireSudog()
    c.recvq.enqueue(mysg)            // 入 recvq 队
  // ...
    gopark(chanparkcommit, unsafe.Pointer(&c.lock), waitReasonChanReceive, traceEvGoBlockRecv, 2)    // 阻塞
  // ...
    return true, success
}
```

**总结**

- 可以向 closed channel 接受值，若其 buf 为空，则返回零值。若其 buf 不为空，正常返回
- 无缓冲 channel 接受消息，是由另一个 goroutine 里的栈直接拷贝过来的
- 有缓冲 channel 总是取 buf 队首消息，但如果 sendq 不为空，还需要把 sendq 需要发送的消息拷贝到 buf 中

## channel vs mutex

> Do not communicate by sharing memory; instead, share memory by communicating.

上述思想实际上就是并发编程中的 **共享内存模型** VS **CSP 信息传递模型**，更具体的说： **mutex** VS **channel**，Go 是第一个引入 CSP 思想并且发扬光大的语言。那这是不是意味着我们在编程中要摒弃 mutex，全部使用 channel 呢？既然 Go 设计者这么推崇 CSP， 为什么 Go 中还是有 mutex 包呢？

channel 与 mutex 侧重点不一样，channel 侧重于协程之间传递数据，mutex 用来保护并发数据

适用于 channel 的场景：

1. 传递数据的所有权，即把某个数据发送给其他协程
2. 分发任务，每个任务都是一个数据
3. 交流异步结果，结果是一个数据

适用于 mutex 的场景：

1. 缓存
2. 状态

实际上，channel 的底层数据结构中也利用 mutex 保护数据，但是适用于 mutex 的场景去使用 channel 性能会差一些。

更具体的我们可以根据下图进行判断

![img](http://ganghuan.oss-cn-shenzhen.aliyuncs.com/img/2de7743a1386dec3af56f6903af7bfde_482x309-2021-11-30.png)

**channel 实际场景**

1. 超时控制/定时任务
2. 并发控制
3. 解耦生产者，消费者
4. ...

**总结**

1. channel 不是银弹，该用 mutex 就用 mutex
2. channnel 用于关注数据流动，mutex 保护固定数据

## 参考

1. https://github.com/golang/go/blob/5786a54cfe34069c865fead1b6d9c9e3485a40a5/src/runtime/chan.go
2. https://www.bookstack.cn/read/qcrao-Go-Questions/channel-%E4%BB%80%E4%B9%88%E6%98%AF%20CSP.md
3. https://www.cnblogs.com/qcrao-2018/p/11220651.html
