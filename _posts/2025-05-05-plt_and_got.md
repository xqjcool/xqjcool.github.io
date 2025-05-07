---
title: "Procedure Linkage Table and Global Offset Table"
date: 2025-05-05
---

# 过程链接表(PLT)和全局偏移表(GOT)

在linux进程调试中，经常会看到你所调用的函数多了一个@plt的后缀，例如printf@plt, puts@plt等等。相信你当时一定有些疑问，plt是什么？ printf@plt与printf有什么关系？...
下面让我们慢慢揭开他们的面纱。

## 认识PLT
为了帮助我们更好的理解，我们理论实践相结合。边调试边学习。

### 示例程序

写一个简单的printf调用程序,然后用gdb调试。

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

int main(int argc, char *argv[])
{
	printf("plt test\n");
	return 0;
}
```

```bash
(gdb) start
Temporary breakpoint 1 at 0x1139
Starting program: /var/log/test 
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib64/libthread_db.so.1".

Temporary breakpoint 1, 0x0000555555555139 in main ()
(gdb) disass main
Dump of assembler code for function main:
   0x0000555555555135 <+0>:     push   %rbp
   0x0000555555555136 <+1>:     mov    %rsp,%rbp
=> 0x0000555555555139 <+4>:     sub    $0x10,%rsp
   0x000055555555513d <+8>:     mov    %edi,-0x4(%rbp)
   0x0000555555555140 <+11>:    mov    %rsi,-0x10(%rbp)
   0x0000555555555144 <+15>:    mov    -0x10(%rbp),%rax
   0x0000555555555148 <+19>:    mov    (%rax),%rax
   0x000055555555514b <+22>:    mov    %rax,%rsi
   0x000055555555514e <+25>:    lea    0xeaf(%rip),%rdi        # 0x555555556004
   0x0000555555555155 <+32>:    mov    $0x0,%eax
   0x000055555555515a <+37>:    call   0x555555555030 <printf@plt>  //printf@plt 本尊
   0x000055555555515f <+42>:    mov    $0x0,%eax
   0x0000555555555164 <+47>:    leave
   0x0000555555555165 <+48>:    ret
End of assembler dump.
```

从示例中，我们看到了printf@plt， 那么为什么要用printf@plt, 而不是直接用printf呢？ 此时就引出了PLT的概念。

## PLT(Procedure Linkage Table)

### 什么是PLT

定义：PLT，全称Procedure Linkage Table，是ELF可执行文件中用于支持动态链接的一种技术机制。
作用：它为程序提供一种间接调用共享库函数的方式，即在程序运行时，根据动态链接器的解析结果调用正确的函数地址。

### 为什么需要PLT？

- 动态链接的需要：程序在编译时不知道最终函数（尤其是共享库函数）的实际地址，需要一种机制在运行时动态绑定。
- 支持Lazy Binding：只有真正调用到某个函数时，才解析其地址，加快程序启动速度。
- 统一跳转入口：对动态库函数的调用统一经过PLT，方便动态链接器管理。
- 位置无关性（PIE、共享库都需要）。

以上都是需要PLT的原因，本质问题是：程序在编译/链接时不知道外部函数最终的内存地址（比如printf在libc里的位置）。
没有PLT的话，程序根本没办法在运行时正确跳到动态库的函数。所以必须有一个机制在程序运行时动态绑定符号地址，这就是PLT最初被设计出来的根本目的。

参考上面的示例程序，在编译时不知道程序运行时，libc.so动态加载后的地址，自然也无法获取到so中printf函数的地址。
为了能够正确调用libc.so中的printf函数，PLT机制就应运而生了。

### function@plt

继续调试示例程序，查看printf@plt。

```bash
(gdb) disass 0x555555555030
Dump of assembler code for function printf@plt:
   0x0000555555555030 <+0>:     jmp    *0x2fca(%rip)        # 0x555555558000 <printf@got.plt>  //这个地址很关键
   0x0000555555555036 <+6>:     push   $0x0
   0x000055555555503b <+11>:    jmp    0x555555555020
```

这个函数相当简单，只有3条指令。第一条指令就是一个jmp，读取`printf@got.plt`中的值，跳转到这个值指向的内存。

这时你的脑中又会多出一个疑问：这个`printf@got.plt`又是什么？ 要说清这个，我们先了解下GOT。

## GOT(Global Offset Table)

### 什么是GOT

GOT，全称Global Offset Table，是ELF格式（Executable and Linkable Format）中用于支持动态链接的重要数据结构。
它在程序运行时保存了需要动态解析的符号（如函数、全局变量）的实际地址。GOT是实现位置无关代码（PIC, Position Independent Code）的关键技术之一。
它的每一项（每一个GOT表entry）里，保存的是一个地址（64位机器上是8字节）。

### 为什么需要GOT

- 支持动态链接：GOT为动态库中的函数和变量提供一种间接访问的机制，程序不需要在编译时确定符号的真实地址。
- 实现位置无关性：GOT允许程序即使被加载到任意内存地址也能正确访问外部资源。
- 支持符号重绑定（symbol rebinding）：通过修改GOT表项，可以在程序运行期间改变函数指针或全局变量指针（比如LD_PRELOAD）。

以上都是需要GOT的原因，最根本的原因是支持动态链接，程序在编译时不知道符号（函数/变量）的最终地址，必须通过GOT表在运行时动态确定地址。

### got.plt 表保存了什么

got.plt表保存的是所有通过PLT调用的外部函数的间接跳转地址。每个外部函数（比如printf()、malloc()）都有一个对应的got.plt表项。程序在运行时，通过访问 got.plt 的表项，跳到外部函数的真实地址。

### 什么需要 got.plt？

- 普通 got 段是给全局变量、静态数据用的。
- got.plt 段是专门给PLT函数调用用的。
