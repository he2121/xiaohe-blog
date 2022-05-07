---
title: "Golang 基础知识总结01"
author: "小贺"
date: 2022-05-10T11:10:12+08:00
tags: ["golang"]
---

## 1. Golang 中 make 与 new 的区别

相同点：都是用来申请内存的

make 一般是用于 slice，map，channel 类型的申请，使用时一般可以指定长度和容量参数, 返回相应的引用类型

new 用于普通类型对象内存的申请，返回的是指针

## 2. 数组与切片的区别

相同点：存放特定数据的集合

数据：固定长度，不可变，值传递

切片：长度可变，引用类型

```go
type slice struct {
	array unsafe.Pointer
	len   int
	cap   int
}
```

## 3. for range 