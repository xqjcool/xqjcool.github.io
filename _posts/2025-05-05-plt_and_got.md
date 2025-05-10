---
title: "Procedure Linkage Table and Global Offset Table"
date: 2025-05-05
---

# 过程链接表(PLT)和全局偏移表(GOT)

在linux进程调试中，经常会看到你所调用的函数多了一个@plt的后缀，例如printf@plt, puts@plt等等。相信你当时一定有些疑问，plt是什么？ printf@plt与printf有什么关系？...
下面让我们慢慢揭开他们的面纱。

## 1. 认识PLT
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

## 2. PLT(Procedure Linkage Table)

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

## 3. GOT(Global Offset Table)

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

## 4. 继续调试

回到 `printf@plt`的反汇编代码，逐步解读。

```bash
   0x0000555555555030 <+0>:     jmp    *0x2fca(%rip)        # 0x555555558000 <printf@got.plt>
   0x0000555555555036 <+6>:     push   $0x0
   0x000055555555503b <+11>:    jmp    0x555555555020
```

### jmp    *0x2fca(%rip)

0x2fca(%rip) 也就是 0x555555558000， 去这个地址中找到指令的地址，跳转执行。

```bash
gdb) x/gx 0x555555558000
0x555555558000 <printf@got.plt>:        0x0000555555555036	//其实就是printf@plt的第二条指令的地址
```

也就是说实际上 `printf@plt`的第一条jmp指令，跳转到 `printf@plt`的第二条指令。是不是有点多此一举？不要着急，很快你就会明白的。

### push   $0x0

这条指令将printf的符号索引压入栈中。每个需要重定位的的符号都会在 rela.plt表中有个entry。0 表示 printf在rela.plt表中的索引为0。

```bash
# readelf -r ./test

Relocation section '.rela.dyn' at offset 0x540 contains 8 entries:
  Offset          Info           Type           Sym. Value    Sym. Name + Addend
000000003dd0  000000000008 R_X86_64_RELATIVE                    1130
000000003dd8  000000000008 R_X86_64_RELATIVE                    10f0
000000004010  000000000008 R_X86_64_RELATIVE                    4010
000000003fc0  000100000006 R_X86_64_GLOB_DAT 0000000000000000 __libc_start_main@GLIBC_2.34 + 0
000000003fc8  000200000006 R_X86_64_GLOB_DAT 0000000000000000 _ITM_deregisterTM[...] + 0
000000003fd0  000400000006 R_X86_64_GLOB_DAT 0000000000000000 __gmon_start__ + 0
000000003fd8  000500000006 R_X86_64_GLOB_DAT 0000000000000000 _ITM_registerTMCl[...] + 0
000000003fe0  000600000006 R_X86_64_GLOB_DAT 0000000000000000 __cxa_finalize@GLIBC_2.2.5 + 0

Relocation section '.rela.plt' at offset 0x600 contains 1 entry:
  Offset          Info           Type           Sym. Value    Sym. Name + Addend
000000004000  000300000007 R_X86_64_JUMP_SLO 0000000000000000 printf@GLIBC_2.2.5 + 0
```

- 这个entry的Offset保存了 printf@got.plt地址。 这里的0x4000是原始地址，需要加上程序基地址0x555555554000。

0x555555554000 + 0x4000 = 0x555555558000 正好就是 

```bash
//查看printf@got.plt地址
(gdb) info address printf@got.plt
Symbol "printf@got.plt" is at 0x555555558000 in a file compiled without debugging.

//通过link_map获取基地址，也就是可执行文件中符号地址与内存中符号地址的差异量
(gdb) x/gx 0x555555557ff0
0x555555557ff0:	0x00007ffff7ffe2e0	//link_map地址
(gdb) x/gx 0x00007ffff7ffe2e0
0x7ffff7ffe2e0:	0x0000555555554000	//link_map.l_addr就是基地址

//通过进程映射获取基地址
(gdb) info proc mappings	//这里可以看到
process 14245
Mapped address spaces:

          Start Addr           End Addr       Size     Offset  Perms  objfile
      0x555555554000     0x555555555000     0x1000        0x0  r--p   /var/log/gui_upload/test	//基地址
      0x555555555000     0x555555556000     0x1000     0x1000  r-xp   /var/log/gui_upload/test
```

- 这个entry的 Sym. Name + Addend 保存了 符号的字符串名称， 后续在so中找。

也就是说后续要在 libc.so中查找 printf@GLIBC_2.2.5 对应符号的地址。

### jmp    0x555555555020

这一步是跳转0x555555555020后继续执行指令。

```bash
(gdb) x/3i 0x555555555020
   0x555555555020:      push   0x2fca(%rip)        # 0x555555557ff0	//0x555555557ff0 中存放了link_map的地址
   0x555555555026:      jmp    *0x2fcc(%rip)        # 0x555555557ff8
   0x55555555502c:      nopl   0x0(%rax)
```

这里第一条指令是 将 line_map地址 压入栈中。然后跳转到 0x555555557ff8中存放的指令地址去执行。

```bash
(gdb) x/gx 0x555555557ff0
0x555555557ff0:	0x00007ffff7ffe2e0	//link_map地址

(gdb) x/gx 0x555555557ff8
0x555555557ff8:	0x00007ffff7fdf720	//跳转指令地址
(gdb) x/64i 0x00007ffff7fdf720	//实际上是<_dl_runtime_resolve_fxsave>
   0x7ffff7fdf720:	push   %rbx
   0x7ffff7fdf721:	mov    %rsp,%rbx
   0x7ffff7fdf724:	and    $0xffffffffffffffc0,%rsp
   0x7ffff7fdf728:	sub    0x1d561(%rip),%rsp        # 0x7ffff7ffcc90 <_rtld_global_ro+464>
   0x7ffff7fdf72f:	mov    %rax,(%rsp)
   0x7ffff7fdf733:	mov    %rcx,0x8(%rsp)
   0x7ffff7fdf738:	mov    %rdx,0x10(%rsp)
   0x7ffff7fdf73d:	mov    %rsi,0x18(%rsp)
   0x7ffff7fdf742:	mov    %rdi,0x20(%rsp)
   0x7ffff7fdf747:	mov    %r8,0x28(%rsp)
   0x7ffff7fdf74c:	mov    %r9,0x30(%rsp)
   0x7ffff7fdf751:	mov    $0xee,%eax
   0x7ffff7fdf756:	xor    %edx,%edx
   0x7ffff7fdf758:	mov    %rdx,0x240(%rsp)
   0x7ffff7fdf760:	mov    %rdx,0x248(%rsp)
   0x7ffff7fdf768:	mov    %rdx,0x250(%rsp)
   0x7ffff7fdf770:	mov    %rdx,0x258(%rsp)
   0x7ffff7fdf778:	mov    %rdx,0x260(%rsp)
   0x7ffff7fdf780:	mov    %rdx,0x268(%rsp)
   0x7ffff7fdf788:	mov    %rdx,0x270(%rsp)
   0x7ffff7fdf790:	mov    %rdx,0x278(%rsp)
   0x7ffff7fdf798:	xsave  0x40(%rsp)
   0x7ffff7fdf79d:	mov    0x10(%rbx),%rsi	//printf的符号索引 0
   0x7ffff7fdf7a1:	mov    0x8(%rbx),%rdi	//link_map地址 0x00007ffff7ffe2e0
   0x7ffff7fdf7a5:	call   0x7ffff7fddad6	//<_dl_fixup>
   0x7ffff7fdf7aa:	mov    %rax,%r11	//返回printf的真正地址，保存的r11寄存器
   0x7ffff7fdf7ad:	mov    $0xee,%eax
   0x7ffff7fdf7b2:	xor    %edx,%edx
   0x7ffff7fdf7b4:	xrstor 0x40(%rsp)
   0x7ffff7fdf7b9:	mov    0x30(%rsp),%r9
   0x7ffff7fdf7be:	mov    0x28(%rsp),%r8
   0x7ffff7fdf7c3:	mov    0x20(%rsp),%rdi
   0x7ffff7fdf7c8:	mov    0x18(%rsp),%rsi
   0x7ffff7fdf7cd:	mov    0x10(%rsp),%rdx
   0x7ffff7fdf7d2:	mov    0x8(%rsp),%rcx
   0x7ffff7fdf7d7:	mov    (%rsp),%rax
   0x7ffff7fdf7db:	mov    %rbx,%rsp
   0x7ffff7fdf7de:	mov    (%rsp),%rbx
   0x7ffff7fdf7e2:	add    $0x18,%rsp
   0x7ffff7fdf7e6:	jmp    *%r11		//跳转到printf去执行

```

_dl_fixup 就不再展开了，简而言之， 它通过 符号索引和link_map，找到对应的rela.plt(见上面), 从而得到最终需要写入的 printf@got.plt地址， 然后通过 printf@GLIBC_2.2.5 字串，遍历link_map，找到对应符号的真实地址。最后写入到 printf@got.plt中。再跳转到真正的printf地址去执行。

这里有两个关键点：

- 将真正printf的符号地址 写入 printf@got.plt， 供后续使用
- 跳转 printf的符号地址，继续本次调用的执行。

### 更新后的printf@got.plt

我们在main退出前设置断点， 观察printf第一次调用后的变化。

```bash
(gdb) b *0x0000555555555164
Breakpoint 2 at 0x555555555164: file test.c, line 9.
(gdb) continue
Continuing.
[/var/log/test] plt test

Breakpoint 2, main (argc=1, argv=0x7fffffffed88) at test.c:9
9	in test.c

(gdb) disass 0x555555555030
Dump of assembler code for function printf@plt:
   0x0000555555555030 <+0>:	jmp    *0x2fca(%rip)        # 0x555555558000 <printf@got.plt>
   0x0000555555555036 <+6>:	push   $0x0
   0x000055555555503b <+11>:	jmp    0x555555555020
End of assembler dump.
(gdb) x/gx 0x555555558000
0x555555558000 <printf@got.plt>:	0x00007ffff7e809ef	//此时已经更新成printf的真正地址了，后续直接jmp到printf执行。
(gdb) info address printf
Symbol "printf" is at 0x7ffff7e809ef in a file compiled without debugging.

```

对比 printf@got.plt中的内容

- 调用前： 0x0000555555555036 ，printf@plt的第二条压栈质量，后续调用_dl_runtime_resolve_fxsave 去解析并保存符号真正地址。
- 调用后： 0x00007ffff7e809ef ，printf的真正地址，再次调用printf@plt时，直接jmp到printf去执行。

## 5. 初次调用和后续执行路径

下面是初次调用和后续执行路径的示意图。
可见PLT的核心在于 printf@got.plt 这个地址，而其他代码都没有改变。
这个地址默认保存 printf@plt的下一条指令，最终去解析和保存真正的符号地址。
一旦保存后，这里的jmp就直接跳转到真正的符号地址去执行。

```text
+---------------------+
| 调用 printf()       |
+---------------------+
           |
           v
+------------------------------+
| printf@plt (PLT entry)       |
| jmp *printf@got.plt          |
+------------------------------+
                          |
                          v
                     (是否已绑定？)
                      /        \
                     / 否       \ 是
                    /            \
                   v              v
+-------------------------+      +------------------------+
| printf@got.plt 指向：    |      | printf@got.plt 指向：   |
| _dl_runtime_resolve     |      | 真正的printf地址        |
+-------------------------+      +------------------------+
           |                                |
           v                                |
+-------------------------------+           |
| _dl_runtime_resolve[_rxsave]  |           |
| - 保存寄存器状态                |           |
| - call _dl_fixup              |           |
+-------------------------------+           |
           |                                |
           v                                |
+--------------------------+                |
| _dl_fixup                |                |
| - 找printf符号            |                |
| - 确定真实地址             |                |
| - 写入GOT[printf]         |                |
+--------------------------+                |
           |                                |
           v                                |
+------------------------------+            |
| 返回 printf 的真实地址，存入%11 |            |
+------------------------------+            |
           |                                |
           v                                |
+-----------+        +------------+         |
| jmp *%r11 |------> | 执行printf  | <-------+
+-----------+        +------------+

```
