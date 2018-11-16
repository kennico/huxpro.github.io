---
layout: post
title: "Segment registers"
subtitle: "段寄存器的前世今生"
date: 2018-11-12 21:42
tags: 
    - Assembly
---

# Contents

一如其名，段寄存器是若干个存储与数据段相关信息的寄存器。特别地，段寄存器长度为 16 bits。段寄存器的作用是存储对应数据段的起始位置。比如，为了获取程序启动时的命令行参数个数，调用 CRT 函数 `__p___argc()` 可能会产生以下指令（用助记符表示，Windows 10, VS2017）：


```assembly
call dword ptr ds:[<&__p___argc>]
```

将代码编译为`compiled-snippet.exe`。调试代码到这条指令处：

![__p___argc()]({{ "img/SR-call-crt-function.png" | absolute_url }})

该指令位于`0x00F61820`，对应机器码为 `FF15 80B1F600`(小端)，长度为6字节。有意思的是 `call` 的参数 `0x00F6B180` 并不是 `__p___argc()` 地址。接着查看在地址 `0x00F6B180` 处的数据：

![__p___argc()]({{ "img/SR-crt-function-address.png" | absolute_url }})

连续四个字节的内容为 `E0EA370F`。这四个字节表示另一个地址`0x0F37EAE0`；这个地址才是函数 `__p___argc()` 第一条指令的地址。

这类`call`指令不使用直接地址，也许是出于共享内存的考虑。而至于为什么`call`指令要带上段寄存器`ds`，可能是编译器认为这样做可以产生长度更短的程序。


此外，当前指令和函数 `__p___argc()` 的代码并不属于同一个数据段：前者位于 `compiled-snippet.exe` 的 `.text` 段，`0x00F6B180` 属于 `.idata` 段；而函数 `__p___argc()` 位于 `ucrtbase.dll` 的 `.text` 段。但在这里 `ds` 的值并不对应 `ucrtbase.dll` 的 `.text` 段，而应该对应 `compiled-snippet.exe` 的 `.idata`段。


然而这个程序将 4GB 内存看作一个大数组，已经不需要按照“段寄存器+段内位移”去产生目标地址。段内位移`0x00F6B180`已经是一个4字节 32-bit 虚拟地址。段寄存器的另一个作用体现为向后兼容性(backward compatibility)，即为旧硬件编写的程序仍然可以运行新硬件上。了解相关硬件的 [发展历史](https://en.wikipedia.org/wiki/X86_memory_segmentation) 对理解这种向后兼容性有所帮助。

## Real mode in Intel 8086

现在大众所说的“x86”指令集架构正是源于 Intel 8086。准确的来讲，“x86”并不能和“32位”画上等号。Intel 8086 是一个 16-bit 的处理器，寄存器的长度都是16位。特别地， 8086 包含了四个 **段寄存器** `cs`(code segment register), `ds`(data segment register), `es`(extra segment register) 和 `ss`(stack segment register)。

然而8086的外部却有 20 条地址总线。因此 8086 有特别的寻址技巧：

1. 通过分段来管理内存，段的起始地址按照 16B 对齐，大小不固定。不同的段可以重叠；
2. 物理地址(address)由两个16-bit的整数计算得到，一个是段的起始地址(segment)，另一个是段内位移(offset)。它们之间的关系如下：

$$\text{address}=\text{segment}\times16+\text{offset}$$

在寻址过程中，**段寄存器** 的值被解释为 **段的起始地址**。整个计算过程由CPU的分段单元(Segmentation Unit)完成，应用程序只需要修改段寄存器的值即可访问内存中任何位置。

虽然 8086 是一个 16-bit 处理器，但上述寻址方式使得 8086 的寻址空间超过了 64KB 而扩大到 1MB。这种寻址方式在 80286 中被重命名为"实模式(Real mode)"，而 80286 又引入了新的寻址方式“保护模式(Protected mode)”。

## Protected mode in Intel 80286

Intel 80286 的仍然是 16-bit 的处理器，但是其外部有24条地址总线，寻址能力扩大到16MB。80286 的保护模式引入了 **（段描述符表）Segmentation Descriptor Table**，该表的表项包含两个整数，一个是长度为24位的段的起始地址，另一个表示段的大小。而在寻址过程中，**段寄存器** 的值被解释为 **段在表中的索引**(13-bit)、请求的特权等级(2-bit)和表指示器(1-bit)的组合。每次寻址都要比较请求的特权等级和对应的表项，倘若不满足特权条件将引起[一般保护中断(general protection fault)](https://en.wikipedia.org/wiki/General_protection_fault)。具体的寻址方式参见[wiki](https://en.wikipedia.org/wiki/X86_memory_segmentation#Detailed_Segmentation_Unit_Workflow)。

为了向后兼容性，80286 以实模式启动，应用程序仍然能够通过“段寄存器+段内位移”的方式访问内存；一旦切换到保护模式，除了触发硬件重置 CPU 以外，不能切换到实模式。

## Intel 80386

80386 是一个32-bit 的处理器，直到现在 80386 架构指令集仍然占据主流。现在有的应用程序按照不同指令集架构分发不同版本，其中某些版本带有的 i386 字样后缀即表示 Intel 80386。

在 80386 段描述符表中，表项的段起始地址的长度增加到32bits，段内位移也增加到32bits。直到 80386 之前，应用程序都能直接按照物理地址访问内存。但是 80386 增加了分页单元，将其作为分段单元和地址总线之间的中间人。页和段不同：页的大小是固定的而且对程序员透明。如果启用分页单元，段内的地址（段的起始地址和段内位移）只能是虚拟地址；最终由分页单元将虚拟地址翻译为物理地址。此外 80386 还引入另外两个通用的 16-bit 段寄存器 FS 和 GS，在 Linux 和 Windows 下的用途不同。

到了现在，段寄存器仍然可以在指令中显式地指定，但在更多的情况下是被 **隐式** 地使用的。比如

1. 所有指令都由 CPU 从 `cs` 所指定的代码段中取得；
2. 大部分内存引用都源自 `ds` 指定的数段；
3. 栈的操作(e.g. `push` and `pop` 指令和 `(e)sp` and `(e)bp` 寄存器)
4. [字符串指令](http://www.plantation-productions.com/Webster/www.artofasm.com/Linux/HTML/StringInstructions.html)

现在我们所熟知的操作系统，比如 Linux，实现了 [平面存储器模式(Flat memory model)](https://en.wikipedia.org/wiki/X86_memory_segmentation#Practices)，将 `cs` 和 `ds` 设置为 0；由段内位移直接产生虚拟地址。