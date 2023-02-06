---
title: "Go调度器"
date: 2021-07-13T20:03:49+08:00
draft: true
categories: ["Go"]
tags: ["source code"]
---

## TD;LR

Go语言是一门有[runtime](/posts/cs/note-of-runtime/)的语言，在go代码中内存分配，goroutine的创建，channel的通信等都是通过与runtime交互，runtime再通过系统调用与操作系统交互。Go语言中的runtime主要分为四个部分：`scheduler`、 `netpoll`、`memory management`和`garbage collector`构成，其中最核心的就是`scheduler`。本文结合源码阐述了调度器的原理，包括调度组件，调度器的启动，调度循环。

<!--more-->

## 调度组件

在深入调度器前面首先简单介绍下`GPM`的概念：

+ G：**G**oroutine，可以理解为一个计算任务，是调度系统的最基本单位，由需要执行的代码和其上下文组成。上下文包括stack信息、goroutine的状态等。
+ P：**P**rocessor，逻辑处理器，对于G来说，P相当于CPU核心，需要绑定到P才能被调度；对于M来说，P相当于上下文，需要获得P提供的信息比如内存分布状态，任务队列等才能执行代码。
+ M：**M**achine，系统线程，计算任务的执行实体。与C语言中的线程相同，需要通过系统调用`clone`来创建。M拿到P后进入调度循环，但M并不保留G的状态。

此外，在go调度器中还有两个不同的队列：全局运行队列（GRQ，Global Run Queue，双向链表）和本地运行队列（LRQ，Local Run Queue，大小为256的数组）。每个P都持有一个LRQ和`runnext`，管理分配到P上的G。

![调度组件](https://raw.githubusercontent.com/konomichael/blog-images/main/20230205172343.png)

## Go 进程如何启动的

对于一段go程序

```go
// main.go
package main
func main() {
  println("hello, world!")
}
```

main函数并非入口函数，入口函数在`asm_amd64.s`中定义。在linux下，通过`go build main.go`生成二进制文件`main`，并通过`readelf -h main`读取header信息：

```
ELF Header:
  Magic:   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00 
  Class:                             ELF64
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              EXEC (Executable file)
  Machine:                           Advanced Micro Devices X86-64
  Version:                           0x1
  Entry point address:               0x45ed60 <- 入口地址
  Start of program headers:          64 (bytes into file)
  Start of section headers:          456 (bytes into file)
  Flags:                             0x0
  Size of this header:               64 (bytes)
  Size of program headers:           56 (bytes)
  Number of program headers:         7
  Size of section headers:           64 (bytes)
  Number of section headers:         23
  Section header string table index: 3
```

通过`dlv exec main`并在入口地址打断点进行debug：

```
(dlv) b *0x45ed60
Breakpoint 1 set at 0x45ed60 for _rt0_amd64_linux() /usr/local/go/src/runtime/rt0_linux_amd64.s:8
```

`rt0_linux_amd64.s`相关内容如下：

```assembly
TEXT _rt0_arm64_linux(SB),NOSPLIT|NOFRAME,$0
	MOVD	0(RSP), R0	// argc
	ADD	$8, RSP, R1	// argv
	BL	main(SB)
TEXT main(SB),NOSPLIT|NOFRAME,$0
	MOVD	$runtime·rt0_go(SB), R2
	BL	(R2)
```

