---
title: "Go进程引导过程-1"
date: 2021-02-05T18:17:07+08:00
draft: false
categories: ["Go"]
tags: ["source code"]
series: ["Go bootstrap"]
series_order: 1
---

本系统介绍了Go进程是如何引导启动的，以此来一窥go运行时的更多细节。本文为第一部分，涵盖了从程序入口到运行时初始化的过程。注意，阅读本文需要有一定的[汇编](https://go.dev/doc/asm)知识。

<!--more-->

## 找到入口

我的go代码很简单:

```go
package main

func main() {
	println("Hello, playground")
}
```

使用命令`go build main.go`编译成可执行文件。使用`readelf -h main`：

{{<highlight text "linenos=table,hl_lines=11">}}

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
  Entry point address:               0x456720
  Start of program headers:          64 (bytes into file)
  Start of section headers:          456 (bytes into file)
  Flags:                             0x0
  Size of this header:               64 (bytes)
  Size of program headers:           56 (bytes)
  Number of program headers:         7
  Size of section headers:           64 (bytes)
  Number of section headers:         23
  Section header string table index: 3
{{</highlight >}}

可以看到，程序入口在`0x456720`位置。通过`dlv exec main`并在入口地址打断点进行debug：

```text
(dlv) b *0x45ed60
Breakpoint 1 set at 0x45ed60 for _rt0_amd64_linux() /usr/local/go/src/runtime/rt0_linux_amd64.s:8
```

现在，我们找到了程序的入口`rt0_linux_amd64.s`的第8行。

## 启动顺序

该入口函数如下：

```assembly
TEXT _rt0_amd64_linux(SB),NOSPLIT,$-8
	JMP	_rt0_amd64(SB)
```

该函数跳转到`_rt0_amd64`函数，通过搜索可以知道它在通用文件`asm_amd64.s`中：

```assembly
TEXT _rt0_amd64(SB),NOSPLIT,$-8
	MOVQ	0(SP), DI	// argc
	LEAQ	8(SP), SI	// argv
	JMP	runtime·rt0_go(SB)
```

`_rt0_amd64`很简单，它将参数`argc`和`argv`分别存入寄存器`DI`和`SI`中。这两个参数可以从栈上根据`SP（栈指针）`获得。
{{<alert "circle-info">}}

+ 当使用`mov{bwlq}`指令传递数据时，数据的大小由mov指令的后缀决定。movq:就是8字节。
+ `lea` 为`load effecitve address`的缩写，复制指针。
{{</alert>}}

现在，我们来看`runtime.rt0_go(SB)`的第一段：

```assembly
MOVQ	DI, AX		// argc
MOVQ	SI, BX		// argv
SUBQ	$(5*8), SP		// 3args 2auto
ANDQ	$~15, SP
MOVQ	AX, 24(SP)
MOVQ	BX, 32(SP)
```

这里，我们将之前保存的命令行参数复制到寄存器`AX`和`BX`中。然后从SP中减去40个字节为3个参数和2个自动变量腾出空间。通过使用位运算`AND`，将SP对齐到最接近的16字节边界（读取性能最佳）。最后，将`AX`和`BX`中的值复制到栈上。

```assembly
MOVQ	$runtime·g0(SB), DI
LEAQ	(-64*1024+104)(SP), BX
MOVQ	BX, g_stackguard0(DI)
MOVQ	BX, g_stackguard1(DI)
MOVQ	BX, (g_stack+stack_lo)(DI)
MOVQ	SP, (g_stack+stack_hi)(DI)
```

这一段首先将全局变量`runtime.g0`加载到`DI`，该变量在`proc.go`文件：

```go
//proc.go
var (
	m0           m
	g0           g
	mcache0      *mcache
	raceprocctx0 uintptr
)
```

类型`g`就是`goroutine`的类型，`runtime.g0`就是根goroutine。然后为g0初始化字段。`stack_lo`和`stack_hi`分别指向当前goroutine中的栈的开始和结束。但是`g_stackguard0`和`g_stackguard1`的含义是什么呢？我们还需要进一步了解Go中的栈。



## Go中的可变大小栈

Go语言使用了可变大小栈，每个goroutine一开始有个小栈，每当栈的规模达到阈值时改变。

```go
type g struct {
  stack       stack // 16 字节
	stackguard0 uintptr // 栈指针，在Go栈增长序言中使用与SP比较
	stackguard1 uintptr // C语言中使用
  ...
}
type stack struct {
  lo uintptr
	hi uintptr
}
```



何时检查呢，显然，在每个goroutine执行函数开始前检查。使用命令`go tool compile -N -l -S main.go`可以得到：

```assembly
main.main STEXT size=66 args=0x0 locals=0x18 funcid=0x0 align=0x0
        0x0000 00000 (main.go:3)        TEXT    main.main(SB), ABIInternal, $24-0
        0x0000 00000 (main.go:3)        CMPQ    SP, 16(R14)
        0x0004 00004 (main.go:3)        PCDATA  $0, $-2
        0x0004 00004 (main.go:3)        JLS     57
        0x0006 00006 (main.go:3)        PCDATA  $0, $-1
        0x0006 00006 (main.go:3)        SUBQ    $24, SP
        0x000a 00010 (main.go:3)        MOVQ    BP, 16(SP)
        0x000f 00015 (main.go:3)        LEAQ    16(SP), BP
        0x0014 00020 (main.go:3)        FUNCDATA        $0, gclocals·g2BeySu+wFnoycgXfElmcg==(SB)
        0x0014 00020 (main.go:3)        FUNCDATA        $1, gclocals·g2BeySu+wFnoycgXfElmcg==(SB)
        0x0014 00020 (main.go:4)        PCDATA  $1, $0
        0x0014 00020 (main.go:4)        CALL    runtime.printlock(SB)
        0x0019 00025 (main.go:4)        LEAQ    go.string."Hello, world!\n"(SB), AX
        0x0020 00032 (main.go:4)        MOVL    $14, BX
        0x0025 00037 (main.go:4)        CALL    runtime.printstring(SB)
        0x002a 00042 (main.go:4)        CALL    runtime.printunlock(SB)
        0x002f 00047 (main.go:5)        MOVQ    16(SP), BP
        0x0034 00052 (main.go:5)        ADDQ    $24, SP
        0x0038 00056 (main.go:5)        RET
        0x0039 00057 (main.go:5)        NOP
        0x0039 00057 (main.go:3)        PCDATA  $1, $-1
        0x0039 00057 (main.go:3)        PCDATA  $0, $-2
        0x0039 00057 (main.go:3)        CALL    runtime.morestack_noctxt(SB)
        0x003e 00062 (main.go:3)        PCDATA  $0, $-1
        0x003e 00062 (main.go:3)        NOP
        0x0040 00064 (main.go:3)        JMP     0

```

首先，在进入函数前我们从线程本地存储（TLS）加载一个值到`R14`寄存器（下面会讲到），这个值总是包含一个指向当前goroutine对应的runtime.g结构的指针。然后，我们将SP与runtime.g结构中偏移量为16字节的位置的值进行比较。我们可以很容易地计算出它对应于`stackguard0`字段。

因此，这就是我们检查是否达到堆栈阈值的方法。如果还没有达到它，检查就会失败。在这种情况下，我们不断调用`runtime.morestack_noctxt`函数，直到为堆栈分配了足够的内存。stackguard1字段与stackguard0非常相似，但它在C堆栈增长序言代码中被使用，而不是Go。现在，让我们回到引导过程。

## 获取cpu信息

```assembly
  MOVL	$0, AX
  CPUID
  CMPL	AX, $0
  JE	nocpuinfo

  CMPL	BX, $0x756E6547  // "Genu"
  JNE	notintel
  CMPL	DX, $0x49656E69  // "ineI"
  JNE	notintel
  CMPL	CX, $0x6C65746E  // "ntel"
  JNE	notintel
  MOVB	$1, runtime·isIntel(SB)
notintel:
  // Load EAX=1 cpuid flags
  MOVL	$1, AX
  CPUID
  MOVL	AX, runtime·processorVersionInfo(SB)
```

这一部分对主要的Go概念不是很重要，我们快速过一下。这里，我们想知道处理器类型。如果是`intel`，就把全局变量`isIntel`设为true。然后将CPUID存入全局变量`processorVersionInfo`中，这两个全局变量最终在`memmove_amd64.so`中使用。

```assembly
nocpuinfo:
  // if there is an _cgo_init, call it.
  MOVQ	_cgo_init(SB), AX
  TESTQ	AX, AX
  JZ	needtls
  // arg 1: g0, already in DI
  MOVQ	$setg_gcc<>(SB), SI // arg 2: setg_gcc
```

这一段是启用cgo的代码，直接跳过，到`needtls`:

```assembly
needtls:
#ifdef GOOS_plan9
	// skip TLS setup on Plan 9
	JMP ok
#endif
#ifdef GOOS_solaris
	// skip TLS setup on Solaris
	JMP ok
#endif
#ifdef GOOS_illumos
	// skip TLS setup on illumos
	JMP ok
#endif
#ifdef GOOS_darwin
	// skip TLS setup on Darwin
	JMP ok
#endif
#ifdef GOOS_openbsd
	// skip TLS setup on OpenBSD
	JMP ok
#endif

	LEAQ	runtime·m0+m_tls(SB), DI
	CALL	runtime·settls(SB)

	// store through it, to make sure it works
	get_tls(BX)
	MOVQ	$0x123, g(BX)
	MOVQ	runtime·m0+m_tls(SB), AX
	CMPQ	AX, $0x123
	JEQ 2(PC)
	CALL	runtime·abort(SB)
```

这一部分，先检查系统，决定是否配置TLS。现在，我们可以可见TLS是如何实现的了。

## TLS实现

仔细看，我们可以发现上面代码就有两行是我们需要的：

```assembly
LEAQ	runtime·m0+m_tls(SB), DI
CALL	runtime·settls(SB)
```

此处`m0`为全局变量，`runtime.m0+tls`即m0的tls字段，将该字段地址存入`DI`寄存器中。m0类型为m：

```go
type m struct {
  g0 *g
  ...
  tls [tslSlots]uintptr // TLS,tlsSlots=6
}
```

在`sys_linux_amd64.s`中，`settls`如下：

```assembly
TEXT runtime·settls(SB),NOSPLIT,$32
#ifdef GOOS_android
	// Android stores the TLS offset in runtime·tls_g.
	SUBQ	runtime·tls_g(SB), DI
#else
	ADDQ	$8, DI	// ELF wants to use -8(FS)
#endif
	MOVQ	DI, SI
	MOVQ	$0x1002, DI	// ARCH_SET_FS
	MOVQ	$SYS_arch_prctl, AX
	SYSCALL
	CMPQ	AX, $0xfffffffffffff001
	JLS	2(PC)
	MOVL	$0xf1, 0xf1  // crash
	RET
```

这段函数进行了`arch_prctl`系统调用，并且传入了`ARCH_SET_FS`和`runtime·m0+m_tls(SB)+8`作为参数。
{{<alert "circle-info">}}
``arch_prctl(ARCH_SET_FS, fsbase)` 是一个 Linux 系统调用，用于设置架构相关的特定控制。在这段代码中，它被用来设置当前进程的段寄存器。

具体来说，它用于设置 ARCH_SET_FS 标志，以指示对当前进程的 FS 寄存器进行设置，值为`fsbase`。FS 寄存器在 x86-64 架构中用于线程本地存储 (TLS)。
{{</alert>}}

返回到上面的代码:

```assembly
get_tls(BX)
MOVQ	$0x123, g(BX)
```

这里`get_tls`和`g`为宏：

```c
#ifdef GOARCH_amd64
#define	get_tls(r)	MOVQ TLS, r
#define	g(r)	0(r)(TLS*1)
#endif
```

使用`objdump -d main`命令搜索可以知道这两条指令最终汇编如下：

```assembly
454af7:	64 48 c7 04 25 f8 ff 	movq   $0x123,%fs:0xfffffffffffffff8
```

`%fs:0xfffffffffffffff8`为段选择器，`0xfffffffffffffff8`为负值。

```assembly
MOVQ	runtime·m0+m_tls(SB), AX
CMPQ	AX, $0x123
```

用段选择器存入存入$0x123，再用`runtime·m0+m_tls(SB)`取出来，判断是否还是0x123，确保tls设置成功。

## 初始化g0和m0

现在，还有两块代码没有读到，其中一块为:

```assembly
ok:
  get_tls(BX)
  LEAQ	runtime·g0(SB), CX
  MOVQ	CX, g(BX)
  LEAQ	runtime·m0(SB), AX

  // save m->g0 = g0
  MOVQ	CX, m_g0(AX)
  // save m0 to g0->m
  MOVQ	AX, g_m(CX)

  CLD				// convention is D is always left cleared
```

首先将TLS地址加载到`BX`中，然后将`g0`地址拷贝到TLS中和`CX`中。`LEAQ runtime·m0(SB), AX`将`m0`的地址存入`AX`中，`m_g0(AX)`表示相对于`AX`中存入地址偏移量为`m_g0`的值，也就是m0.g0, 因此`MOVQ CX, m_g0(AX)`将g0存入到`m0.g0`中。`MOVQAX, g_m(CX)`则表示`g0.m=m0`。

`CLD`指令清除掉`FLAGS`寄存器中的`direction` flag，该flag影响字符串处理的方向。

最后一大块代码如下：

```assembly
CALL	runtime·check(SB)

MOVL	24(SP), AX		// copy argc
MOVL	AX, 0(SP)
MOVQ	32(SP), AX		// copy argv
MOVQ	AX, 8(SP)
CALL	runtime·args(SB)
CALL	runtime·osinit(SB)
CALL	runtime·schedinit(SB)

// create a new goroutine to start program
MOVQ	$runtime·mainPC(SB), AX		// entry
PUSHQ	AX
CALL	runtime·newproc(SB)
POPQ	AX

// start this M
CALL	runtime·mstart(SB)

CALL	runtime·abort(SB)	// mstart should never return
RET
```

`runtime.check`检查了内建数据类型的大小，cas操作等，如果出错则panic。

## runtime.args

函数`runtime.args`如下：

```go
// runtime1.go
func args(c int32, v **byte) {
	argc = c
	argv = v
	sysargs(c, v)
}
```

将c和v存入全局变量，然后调用`sysargs`。该函数相对麻烦些，在linux系统下，它负责分析ELF 辅助向量，初始化调用地址。linux启动后栈布局如下：

```
+----------------------+
|     Stack Layout     |
+----------------------+
| ...                  |
| argc                 |
| argv                 |
| envp                 |
| ...                  |
| argv pointers        |
| NULL                 |
| environment pointers |
| NULL                 |
| ELF Auxiliary Table  |
| argv strings         |
| environment strings  |
| program name         |
| NULL                 |
+----------------------+
| argv (rsp+32)        |
| argc (rsp+24)        |
| (Not Set) (rsp+16)   |
| (Not Set) (rsp+8)    |
| (Not Set) (rsp+0)    |
+----------------------+ 
```

当操作系统加载程序到内存时，它使用预定义格式的数据初始化该程序的初始堆栈。在堆栈顶部是参数——环境变量的指针。在底部，我们可以找到「ELF辅助向量」，实际上是一个包含其他有用信息的记录数组。该函数最终回调用`sysauxv`：

```go
func sysauxv(auxv []uintptr) int {
	var i int
	for ; auxv[i] != _AT_NULL; i += 2 {
		tag, val := auxv[i], auxv[i+1]
		switch tag {
		case _AT_RANDOM:
			// The kernel provides a pointer to 16-bytes
			// worth of random data.
			startupRandomData = (*[16]byte)(unsafe.Pointer(val))[:]

		case _AT_PAGESZ:
			physPageSize = val
		}

		archauxv(tag, val)
		vdsoauxv(tag, val)
	}
	return i / 2
}
```

这里初始化了两个全局变量：`startupRandomData`和`physPageSize`。前者主要用来生成初始化hash函数的seed，后者主要初始化malloc。

## runtime.osinit

`runtime.osinit`主要用来获取`ncpu`和`physHugePageSize`。其中`ncpu`是当前cpu的核数，计算方法涉及到`sched_getaffinity`系统调用，这里不再赘述。

## 总结

这篇文章中我们探索了go程序入口，可变大小栈和TLS实现。我们知道了m0绑定了g0, g0也绑定了m0，这意味着调度即将发生，下一篇我们将会见识到调度器是如何启动的。
