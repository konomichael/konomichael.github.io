+++
title = "Memo of Linux Kernel Bootstrap"
date = "2023-03-11T14:28:53+08:00"
author = "konomichael"
authorTwitter = "konomichael_" #do not include @
tags = ["os", "Linux"]
keywords = ["os", "Linux"]
showFullContent = false
readingTime = true

+++

I'm reading a book about Linux kernel `0.11` recently, and this is a memo of what I've learned.
<!--more-->

## 1. BIOS

The CPU runs in `real mode` when it's powered on: the `CS` is `0xFFFF`, `IP` is `0x0000`, `CS:IP` points to the first instruction of the BIOS. The BIOS is stored in the ROM:

|Interrupt Vector Table|BIOS Data       | |Interrupt Service|--->|BIOS Code       |
|:--:|:--:|:--:|:--:|:--:|:--:|
|0x00000 - 0x003FF       |0x00400  - 0x004FF| |0x0E05B - 0x0FFFE  |    |0xFE000 - 0xFFFFF|
> + The `CS` is `0xFFFF`, `IP` is `0x0000`, `CS:IP` is calculated as `0xFFFF*16 + 0x0000 = 0xFFFF0`.
> + A Interrupt Vector comprises of 4 bytes: `CS:IP`. And there are 256 Interrupt Vectors in the Interrupt Vector Table: `0x0400/4 = 256`.
> + In read mode, the CPU can only access the first 1MB(`0x00000-0xFFFFF`) of memory.


## 2. Loading Kernel
### 2.1. Loading Boot Sector
After the BIOS have finished `POST(Power On Self Test)`, it interrupts with `INT 0x19` to read the first sector of the boot disk into the memory at `0x7C00`, where the code `bootsec.s` is loaded to.

### 2.2. Loading Setup
The `bootsec.s` first desins memory placement:
```assembly
SETUPLEN = 4
BOOTSEG = 0x07C0
INITSEG = 0x9000
SETUPSEG = 0x9020
SYSSEG = 0x1000
ENDSEG = SYSSEG + SYSSIZE
```
Then it copys itself to `0x9000:0x0000(INITSEG:0x0000)` and loads the `setup.s` with the interrupt `INT 0x13`:
```assembly
load_setup:
    mov dx, #0x0000          ! drive 0, head 0
    mov cx, #0x0002          ! sector 2, track 0
    mov bx, #0x0200          ! address = 512, in INITSEG
    mov ax, #0x0200+SETUPLEN ! service 2, nr of sectors
    int 0x13                 ! read it
    jnc ok_setup
```
The interrupt service `INT 0x13` reads `SETUPLEN` sectors at sector `2` of the boot disk into the memory at `INITSEG:0x0200`.
> + The high 8 bits of `dx` is the drive number, and the low 8 bits is the head number.
> + The high 8 bits of `cx` is the traack number, and the low 8 bits is the sector number.
> + The high 8 bits of `ax` is the service number, and the low 8 bits is the number of sectors to read.
> + The `es:bx` is the address to read the data into. We have set `es = INITSEG` before.

The memory layout after the `setup.s` is loaded:
|bootsec.s|setup.s|...SP(Stack Pointer)|
|:--:|:--:|:--:|
|0x90000 - 0x901FF|0x90200 - 0x903FF|...0xFF000|

### 2.3. Loading System
The `system` is then loaded into the memory at `0x10000`, which reads 240 sectors after the `setup` sector. After that, the root device is checked and it's number is stored at `INITSEG:508`:
```assembly
org 508                 ; 0x1FC
root_dev dw ROOT_DEV    ; the root device number is stored here, a word(2 bytes)
boot_flag dw 0AA55h
```

> The number of the root device is calculated as `major * 256 + minor`. The more details can be found [here](https://www.kernel.org/doc/Documentation/admin-guide/devices.txt).

### 2.4. Jumping to Setup
Then it jumps to the `setup.s` with `jmpi 0, SETUPSEG`, start extracting the machine information with interrupts. The informations and their memory addresses are listed below:
|Information|Memory Address|Interrupt|
|:--:|:--:|:--:|
|Cursor Position|0x90000|0x10|
|Extended Memory Size|0x90002|0x15|
|Active Page|0x90004|0x10|
|Video Mode|0x90006|0x10|
|Number of Character Columns|0x90007|0x10|
|??|0x90008|0x10|
|EGA Memory|0x9000A|0x10|
|Color(Mono) Mode|0x9000B|0x10|
|Switch settings|0x9000C|0x10|
|Feature bits|0x9000D|0x10|
|1st Hard Disk Parameter Table|0x90080|the value of 0x41-0x45 interrupt vectors|
|2nd Hard Disk Parameter Table|0x90090|the value of 0x46-0x50 interrupt vectors|
|Root Device Number|0x901FC|stored when loading setup|

## 3. Preparing for Protected Mode
### 3.1. Closing the Interrupt
To close the interrupt, it first sets the `IF(Interrupt Flag)` of the `EFLAGS` to `0` with the instruction: `cli`. It then copies the kernel program at the `0x10000` to the start of memory `0x00000`, which overwrites the interrupt vector table and the BIOS data. The system can not deal with the interrupt anymore until it rebuilds the interrupt vector table, which is why the `cli` is called before.
### 3.2. Setting up the GDT and IDT
In real mode, the CPU can access the physic address with `DS:SI`, where the `DS` stores `segment base address` and the `SI` stores `offset`. And the segment address is calculated as `DS*16 + SI`. However, in protected mode, the `DS` stores the `segment selector`:
```text
 15                                                                    2    1
┌─────────────────────────────────────────────────────────────────────┬────┬────┐
│             Descriptor  index                                       │ TI │RPL │
└─────────────────────────────────────────────────────────────────────┴────┴────┘
```
The `TI` is the `Table Indicator`, which is `0` for the `GDT` and `1` for the `LDT`. The `RPL` is the `Requestor Privilege Level`, which is the privilege level of the segment. A `Segment Descriptor` can be accessed in the `GDT` or `LDT` with the `index` and the `TI`. 
```text
 31               23 22 21 20 19           15 14   12 11        7
┌────────────────┬──┬──┬──┬──┬────────────┬──┬────┬──┬─────────┬────────────────┐
│  Base address  │  │D │  │A │ Segment    │  │ D  │  │         │  Base address  │
│   31 - 24      │G │/ │0 │V │ limit      │P │ P  │S │ Type    │   23 - 16      │
│                │  │B │  │L │ 19-16      │  │ L  │  │         │                │
└────────────────┴──┴──┴──┴──┴────────────┴──┴────┴──┴─────────┴────────────────┘
┌─────────────────────────────────────────┬─────────────────────────────────────┐
│     Base address  15 - 0                │        Segment limit 15 - 0         │
└─────────────────────────────────────────┴─────────────────────────────────────┘
```
The base address can be extracted from the descriptor, and the physics address is calculated as `base address + offset`.

The `GDT(Global Descriptor Table)` is stored in the memory, can be accessed by the `GDTR(Global Descriptor Table Register)`:
```text
 47                                                  15                       
┌────────────────────────────────────────────────────┬────────────────────────┐
│        Offset(liner address of GDT)                │   table size           │
└────────────────────────────────────────────────────┴────────────────────────┘
```
The `setup.s` declares a `GDT` with 256 entries and sets the `GDTR` to point to it:
```assembly
lgdt gdt_48
gdt_48:
    .word   0x800       ; gdt limit=2048, 256 GDT entries, a descriptor is 8 bytes
    .word   512+gdt,0x9 ; gdt base = 0X9xxxx
```

The `gdt` is defined as:
```assembly
gdt:
    .word   0,0,0,0     ; dummy

    .word   0x07FF      ; 8Mb - limit=2047
    .word   0x0000      ; base address=0
    .word   0x9A00      ; code read/exec
    .word   0x00C0      ; granularity=4096, 386

    .word   0x07FF      ; 8Mb - limit=2047
    .word   0x0000      ; base address=0
    .word   0x9200      ; data read/write
    .word   0x00C0      ; granularity=4096, 386
```
There're 3 descriptors in the `GDT`, the first is empty, the second is for the code segment and the third is for the data segment.
Explanation of the second descriptor:
```text
 31               23 22 21 20 19           15 14   12 11        7
┌────────────────┬──┬──┬──┬──┬────────────┬──┬────┬──┬─────────┬────────────────┐
│       0x00     │1 │1 │0 │0 │   0x0      │1 │ 00 │1 │ 1010    │     0x00       │
└────────────────┴──┴──┴──┴──┴────────────┴──┴────┴──┴─────────┴────────────────┘
┌─────────────────────────────────────────┬─────────────────────────────────────┐
│                0x0000                   │                0x07FF               │
└─────────────────────────────────────────┴─────────────────────────────────────┘
```
+ `limit(0-15)`: `0x07FF`
+ `base address(0-15)`: `0x0000`
+ `base address(16-23)`: `0x00`
+ `S`: `1` for code/data segment
+ `Type`: `1010` for code segment, execute/read,
+ `DPL`: `00` for kernel code/data segment, ring 0
+ `P`: `1` for present
+ `limit(16-19)`: `0x0`
+ `AVL`: `0`, available for system software
+ `D/B`: `1` for 32-bit segment
+ `G`: `1` for granularity 4KB, the limit then leads to `[0x07FF(FFF)+1] Bytes = 8MB`
+ `base address(24-31)`: `0x00`

The third descriptor is similar to the second one, except the `Type` is `1001` for data segment, read/write.

The `IDT(Interrupt Descriptor Table)` is also stored in the memory, can be accessed by the `IDTR(Interrupt Descriptor Table Register)`:
```text
 47                                                  15
┌────────────────────────────────────────────────────┬────────────────────────┐
│        Offset(liner address of IDT)                │   table size           │
└────────────────────────────────────────────────────┴────────────────────────┘
```
The `setup.s` declares an empty `IDT` with 256 entries and sets the `IDTR` to point to it:
```assembly
lidt idt_48
idt_48:
    .word   0
    .word   0,0
```
It's ok to have an empty `IDT` at the beginning because the interrupt is not enabled yet. 

### 3.2. Turning on A20 Address Line
The `8086` CPU can address up to `0xFFFFF`, which is `1MB`. Turning on the `A20` line allows the CPU to address up to `0xFFFFFFFF`, which is `4GB`. The more detail of `A20` line can be found in [this article](https://en.wikipedia.org/wiki/A20_line).
### 3.3 Switch to Protected Mode
The `setup.s` switches to the protected mode by setting the `CR0` register:
```assembly
mov ax,#0x0001  ; protected mode (PE) bit
lmsw ax      ; This is it;
jmpi 0,8     ; jmp offset 0 of segment 8 (cs)
```
The `lmsw` instruction loads the `ax` to the `CR0` register, and the `jmpi` instruction jumps to the `0x8:0x0` address. The index of segment descriptor can be extract from the segment selector: `0x8>>3=0x1`, which is the index of the second descriptor in the `GDT`. The `0x0` is the offset of the segment. As the section 3.1, the `base address` of the second descriptor is `0x0000`, so the `0x8:0x0` address is `0x0000`, which points to the `head.s` loaded before.

## 3.5 Executing head.s
The `head.s` starts with:
```assembly
_pg_dir:
_startup_32:
    mov eax,0x10
    mov ds,ax
    mov es,ax
    mov fs,ax
    mov gs,ax
    lss esp,_stack_start
```
The segment registers are set to `0x10 (0x10>>3=2)`, which is the index of the third descriptor in the `GDT`, the data segment. The `lss esp,_stack_start` changes the stack pointer(which is currently `0x9FF00`).
```c
stack_start = {&user_stack[PAGE_SIZE >> 2], 0x10};
```
> lss is `Load Segment Instruction`, which loads the lower word of the given value in memory to the specified register(here is `esp`), and the upper word to the stack segment register(`ss`). Here, `0x10` is loaded to `ss`, and the `user_stack` address is loaded to `esp`.

It then call `setup_idt` to setup the `IDT`:
```assembly
setup_idt:
    lea edx,ignore_int
    mov eax,00080000h
    mov ax,dx
    mov dx,8E00h
    lea edi,_idt
    mov ecx,256
rp_sidt:
    mov [edi],eax
    mov [edi+4],edx
    add edi,8
    dec ecx
    jne rp_sidt
    lidt fword ptr idt_descr
    ret

idt_descr:
    dw 256*8-1
    dd _idt

_idt:
    DQ 256 dup(0)
```
It initializes a 256-entry `IDT(Interrupt Descriptor Table)` with the default `ignore_int` handler. An `ID(Interruption Descriptor)` is represented by 8 bytes:
```text
 31                                        15 14   12 11        7   4
┌──────────────────────────────────────────┬──┬────┬──┬─────────┬─────┬─────────┐
│                Offset(31-16)             │P │ DPL│0 │  Type   │0 0 0│ Unused  │
└──────────────────────────────────────────┴──┴────┴──┴─────────┴─────┴─────────┘
┌──────────────────────────────────────────┬────────────────────────────────────┐
│                Selector                  │            Offset(15-0)            │
└──────────────────────────────────────────┴────────────────────────────────────┘
```
The `0x8e00` is set to the `32-47` bits of the descriptor, which means:
+ `P`: `1` for present
+ `DPL`: `00` for kernel code/data segment, ring 0
+ `Type`: `1110` for interrupt gate, which means the interrupt is handled by the `interrupt handler` in the `IDT` entry.
> A gate (call, interrupt, task or trap) is used to transfer control of execution across segments. Privilege level checking is done differently depending on the type of destination and instruction used.
-- [stackoverflow](https://stackoverflow.com/questions/3425085/the-difference-between-call-gate-interrupt-gate-trap-gate)

The `GPT` is then rebuilt since the older one was built to address the `header.s` code after entrying the protected mode, and it's not safe anymore(may be overwritten). The new one is just like the old one:
```assembly
_gdt:
    DQ 0000000000000000h    ;/* NULL descriptor */
    DQ 00c09a0000000fffh    ;/* 16Mb */
    DQ 00c0920000000fffh    ;/* 16Mb */
    DQ 0000000000000000h    ;/* TEMPORARY - don't use */
    DQ 252 dup(0)
```
> Note: In litle endian, the `0x00c09a0000000fffh` leads to the `limit=0x0fff`, which means the segment can address `0x0ffffff+1=16M` bytes.

### 3.6. Enabling Paging
The `header.s` is going to do the last work before jumping to the `main` function: enabling paging. 

When enabling paging, the linear address, `SegmentSelector:Offset`, is translated to the physical address by the following steps:
+ 1. The `SegmentSelector` is used to index the `GDT` to get the `SegmentDescriptor`, which contains the `base address` of the segment.
+ 2. The `Offset` is added to the `base address` to get the `linear address`.
+ 3. The linear address is translated to the physical address by the `MMU(Memory Management Unit)` according to the `Page Table` in the `Page Directory`.
```text
            11         21
┌──────────┬──────────┬────────────┐
│0000000011│0100000000│000000000100│
└───┬──────┴─┬────────┴───┬────────┘
    │        │            │
    │        │            │
    │        │            │
    │        │            │      ┌────────┐
    │        │            │      │        │
    │        │            │      │        │
    │        │            │      │        │
    │        │            │      │        │
    │        │            └─────►│        │
    │        │      3. Add the   ├────────┤◄───┐
    │        │         offset    │        │    │
    │        │                   │        │    │
    │        │                   │        │    │
    │        │                   │        │    │
    │        │                   │        │    │
    │        │                   │        │    │
    │        │                   │        │    │
    │        │                   ├────────┤    │
    │        │                   │ PT 3   │    │
    │        └──────────────────►│        ├────┘
    │          2.Lookup for 256th├────────┤ 0x4000◄──┐
    │            entry in PT 3   │ PT 2   │          │
    │                            │        │          │
    │                            ├────────┤ 0x3000   │
    │                            │ PT 1   │          │
    │                            │        │          │
    │                            ├────────┤ 0x2000   │
    │                            │ PT 0   │          │
    │                            │        │          │
    │                            ├────────┤ 0x1000   │
    └───────────────────────────►│ PD     ├──────────┘
        1.Lookup for the 3rd     │        │
          entry in PD            └────────┘ 0
                                   Memory
```

To enable paging, it first executes the following code:
```assembly
jmp after_page_tables
...
after_page_tables:
    push 0
    push 0
    push 0
    push L6
    push _main
    jmp setup_paging
L6:
    jmp L6
```
it push 3 `0`s to the stack, which are the parameters to `main`. Then it pushes the `L6` label address to the stack, will endlessly transfer the program's execution flow to the `L6` until the program is interrupted or stopped.

It then jumps to `setup_paging` to create the `Page Directory` and `Page Tables`:
```assembly
setup_paging:
    mov ecx,1024*5
    xor eax,eax
    xor edi,edi
    pushf
    cld
    rep stosd
    mov eax,_pg_dir
    mov [eax],pg0+7
    mov [eax+4],pg1+7
    mov [eax+8],pg2+7
    mov [eax+12],pg3+7
    mov edi,pg3+4092
    mov eax,00fff007h
    std
L3: stosd
    sub eax,00001000h
    jge L3
    popf
```
The `PDE/PTE` is a 32-bit value:
```text
 31                                                    11    9    
┌─────────────────────────────────────────────────────┬─────┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┐
│     Page Table Address (20 bits) PDE/               │     │ │ │ │ │ │P│P│U│R│ │
│                                                     │AVL  │G│0│D│A│G│C│W│ │ │P│
│     Page Address (20 bits) PTE                      │     │ │ │ │ │ │D│T│S│W│ │
└─────────────────────────────────────────────────────┴─────┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┘
```
After building PDEs and PTEs, it sets the `CR3` register to the `Page Directory`'s physical address:
```assembly
xor eax,eax
mov cr3,eax
```
which is `0` since the `Page Directory` is at the beginning of the memory.
It then enable the paging by setting the `CR0.PG` bit to `1`:
```assembly
mov eax,cr0
or  eax,80000000h
mov cr0,eax
```
The paging is enabled now, the final step is to jump to the `main` function:
```assembly
ret
```
which pops the value from the stack to the `EIP` register, which is just `main`'s address. And the `CS` register holds `0x8`(after switching to protected mode, `jmpi 0,8` is called), which points to the `GDT`'s second entry, code segment descriptor.

## Summary
This post records how to switch to the protected mode and enable paging. Now that the protected mode is enabled, the `GDT` is built to make it possible to use the `SegmentSelector:Offset` to address the memory in linear address space. The `Page Directory` and `Page Tables` are built to translate the linear address to the physical address. 