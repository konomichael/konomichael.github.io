---
title: "Go进程引导过程-2"
date: 2021-02-08T03:08:12+08:00
draft: false
categories: ["Go"]
tags: ["source code"]
series: ["Go bootstrap"]
series_order: 2
---

本篇是Go进程引导的第二篇，将会探索栈池、内存分配、堆等知识。

<!--more-->

开始前，我们先回到上一篇结束的地方：`CALL runtime·schedinit(SB)`。函数`schedinit`用来初始化全局调度器`sched`，涉及到的函数比较多，本篇接下来的部分主要讨论这些函数。

## getg()

`schedinit`以`_g_ := getg()`开始，该函数从TLS中获取当前的goroutine。上一节中，我们提到函数调用时会将TLS中的的值拷贝到寄存器中：

{{<highlight text  "linenos=table, hl_lines=3 7-9">}}
00000000004576e0 <runtime.schedinit.abi0>:
  4576e0:	45 0f 57 ff          	xorps  %xmm15,%xmm15
  4576e4:	64 4c 8b 34 25 f8 ff 	mov    %fs:0xfffffffffffffff8,%r14
  4576eb:	ff ff 
  4576ed:	e9 4e 8b fd ff       	jmpq   430240 <runtime.schedinit>
0000000000430240 <runtime.schedinit>:
  430240:	49 3b 66 10          	cmp    0x10(%r14),%rsp
  430244:	0f 86 0d 03 00 00    	jbe    430557 <runtime.schedinit+0x317>
  ...
  430557:	e8 84 48 02 00       	callq  454de0 <runtime.morestack_noctxt.abi0>
{{</highlight>}}

回忆到`%fs:0xfffffffffffffff8`存的是`g0`的地址，`0x10(%r14)`表示g0偏移16个字节，正是`stackguard0`的位置，`cmp    0x10(%r14),%rsp`比较stackguard0是否小于等于栈顶，如果是，则说明需要更多栈空间。回到`getg()`：

```assembly
43024a:	48 83 ec 58          	sub    $0x58,%rsp
43024e:	48 89 6c 24 50       	mov    %rbp,0x50(%rsp)
430253:	48 8d 6c 24 50       	lea    0x50(%rsp),%rbp
430258:	4c 89 74 24 38       	mov    %r14,0x38(%rsp)
```

这一段是`getg`编译后的指令，将`%r14`中的值存入`0x38(%rsp)`内存中，显然，该值就是`g0`。

## 校验链接器符号

链接器符号是链接器向可执行文件和对象文件发出的数据。在runtime包中，链接器符号映射到moduledata结构体。runtime.moduledataverify函数负责对此数据执行一些检查，并验证其具有正确的结构并且不被损坏。

## 初始化栈池

要理解下一个初始化步骤需要对Go中栈的增长实现有一些了解。当创建新的 goroutine 时，为其分配了一个小的固定大小的栈。当栈到达某个阈值时，其大小将翻倍，并将栈复制到另一个位置。

Go使用栈池缓存当前未使用的栈。栈池是在 `runtime.stackinit` 函数中初始化的数组。此数组中的每一个元素包含具有相同大小的栈的链表。

另一个在此阶段初始化的变量是 `runtime.stackFreeQueue`。它也包含栈的链表，但这些栈在垃圾回收期间添加到列表中，并在完成后被清除。请注意，仅缓存 2 KB、4 KB、8 KB 和 16 KB 的栈，较大的栈直接分配。

## 初始化内存分配器

函数`runtime.mallocinit`初始化内存分配器，这个函数下我们首先看到这样一段判断：

```go
if class_to_size[_TinySizeClass] != _TinySize {
		throw("bad TinySizeClass")
}
```

其中的`class_to_size`保存了类到大小的映射。当分配一个小对象（小于32 KB）时，Go runtime会首先将其大小舍入到预定义的类大小。因此，分配的内存块只能具有预定义的一种大小，通常比对象本身所需的大小更大。当然，这也导致了内存浪费，但这样我们可以轻松重用分配的内存块以存储不同的对象。此外，还有`class_to_allocnpages`数组，该数组存储了从OS获取内存页面的数据，以填充给定类的一个对象，以及数组`size_to_class8`和`size_to_class128`。这些用于从对象大小转换为相应类索引。第一个用于转换大小小于1 KB的对象，第二个针对的对象大小为1-32 KB。

### mheap.init()

接下来，我们调用了`mheap.init()`来初始化分配器（这里不会详细讨论）：

```go
h.spanalloc.init(unsafe.Sizeof(mspan{}), recordspan, unsafe.Pointer(h), &memstats.mspan_sys)
	h.cachealloc.init(unsafe.Sizeof(mcache{}), nil, nil, &memstats.mcache_sys)
	h.specialfinalizeralloc.init(unsafe.Sizeof(specialfinalizer{}), nil, nil, &memstats.other_sys)
	h.specialprofilealloc.init(unsafe.Sizeof(specialprofile{}), nil, nil, &memstats.other_sys)
	h.specialReachableAlloc.init(unsafe.Sizeof(specialReachable{}), nil, nil, &memstats.other_sys)
	h.arenaHintAlloc.init(unsafe.Sizeof(arenaHint{}), nil, nil, &memstats.other_sys)
```

分配器列表如下：

```go
spanalloc             fixalloc // allocator for span*
cachealloc            fixalloc // allocator for mcache*
specialfinalizeralloc fixalloc // allocator for specialfinalizer*
specialprofilealloc   fixalloc // allocator for specialprofile*
specialReachableAlloc fixalloc // allocator for specialReachable record allocators.
arenaHintAlloc        fixalloc // allocator for arenaHints
```

它们都是`fixalloc`类型的分配器，是针对固定大小对象的空闲列表分配器的简单实现。为了更好理解一个分配器是什么，我们来看一下它是如何使用的。当我们想要分配一个新的`mspan`，`mcache`，`specialfinalizer`，`specialprofile`，`specialReachable`和`arenaHints`时都会最终调用`alloc`方法，该方法核心逻辑就三行

```go
if uintptr(f.nchunk) < f.size {
		f.chunk = uintptr(persistentalloc(uintptr(f.nalloc), 0, f.stat))
		f.nchunk = f.nalloc
}
```

这个函数分配内存，但不是分配结构体实际大小——f.size字节，而是留下_FixAllocChunk字节（当前等于16KB）。剩下的空间存储在分配器中。下次需要分配同类型的结构体时，不需要调用persistentalloc，这样可以减少时间开销。

`persistentalloc`函数负责分配不应被垃圾回收的内存。它的工作流程如下：

1. 如果分配的块大于64KB，则直接从操作系统内存中分配。
2. 否则，我们需要首先找到一个持久分配器： 
   + 每个处理器都附加了一个持久分配器。这样做是为了避免使用锁时处理持久分配器。因此，我们尝试使用当前处理器的持久分配器。
   +  如果无法获取当前处理器的信息，则使用全局系统分配器。
3.  如果分配器的缓存中没有足够的空闲内存，我们从操作系统中预留更多内存。
4. 所需内存量从分配器的缓存中返回。

`persistentalloc`和`fixAlloc.alloc`函数的工作方式类似。可以说这些函数实现了两个级别的缓存。此外，persistentalloc不仅在fixAlloc.alloc中使用，还在许多其他需要分配持久内存的地方使用。

让我们回到`mheap.init`函数。在这里还要回答的一个重要问题是，这个函数开头初始化的六个结构体如何使用：

+ mspan是应该进行垃圾回收的内存块的wrapper。当我们需要分配一个新的特定大小类的对象时，将创建一个新的mspan。
+ mcache是附加到每个P上的结构,它负责缓存span。为每个处理器单独设置缓存的原因是为了避免上锁。 
+ specialfinalizeralloc是在调用`runtime.SetFinalizer`函数时分配的结构体。如果我们希望在清除对象时系统执行一些清代码，则可以这样做。比如`os.NewFile`函数，它为每个新文件关联一个finilizer，这个finilizer应该关闭OS文件描述符。 
+ specialprofilealloc是内存分析器中使用的结构体。
+ specialReachable跟踪一个对象下个循环是否可达，用于测试
+ arenaHints是一个尝试添加更多heap arena的地址列表。

然后初始化`mheap.central`， 存储的是小对象（小于32 KB）的 span，在 mheap.central 中，列表根据它们的大小类别（spanClass）分组。最后，初始化`pageAlloc`。

## 分配缓存

初始化堆的最后一步是`mcach0=allocmcache()`。正如我们前面讨论的，`allocmcache`最终会调用`fixAlloc.alloc`方法来到分配`mcache`。m0则将会在后面的处理器初始化时绑定到`id=0`的P上。

## 总结

本篇文章主要讨论了内存分配器的初始化，之后还有垃圾收集器初始化和一些其他工作，在之后的垃圾收集会有讨论。另外，还会深入讨论内存分配器。在完成runtime的初始化后，通过调用`runtime.newporc()`创建新的goroutine，这个goroutine封装了真正的main函数，之后我们在讨论Go调度器时会涉及到。
