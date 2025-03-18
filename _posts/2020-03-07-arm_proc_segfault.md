---
title: "[crash analysis][arm]Program Segmentation fault"
date: 2020-03-07
---
定制业务程序prog，在客户设备上运行后出现Segmentation fault。
为了定位问题，编译了未strip的版本，并修改设备core file size为unlimited。重新运行，产生了core_prog_25647。
使用toolchain gdb工具打开，开始调试。

```powershell
...//省略无关
Program terminated with signal 11, Segmentation fault.
#0  0x0000bb88 in nfq_update ()
(gdb) bt
#0  0x0000bb88 in nfq_update ()
#1  0x0000c72c in nfq_create ()
#2  0x0000ade0 in main ()
```
从调用栈看是cransh在nfq_update函数中，我们反汇编nfq_update,找到出错指令位置。

```powershell
(gdb) disass
Dump of assembler code for function nfq_update:
0x0000bb44 <nfq_update+0>:        push    {r4, r5, r6, r7, r8, r9, r10, r11, lr}
0x0000bb48 <nfq_update+4>:        subs    r4, r1, #0      ; 0x0
0x0000bb4c <nfq_update+8>:        sub     sp, sp, #28     ; 0x1c
0x0000bb50 <nfq_update+12>:       str     r0, [sp, #4]		//5. [sp + 4] = r0, 第一个入参
0x0000bb54 <nfq_update+16>:       beq     0xbb64 <nfq_update+32>
0x0000bb58 <nfq_update+20>:       ldr     r3, [r4]
0x0000bb5c <nfq_update+24>:       cmp     r3, #0  ; 0x0
0x0000bb60 <nfq_update+28>:       bne     0xbf98 <nfq_update+1108>
0x0000bb64 <nfq_update+32>:       mov     r10, #0 ; 0x0
0x0000bb68 <nfq_update+36>:       ldr     r9, [sp, #4]		//4. r9 = [sp + 4]
0x0000bb6c <nfq_update+40>:       mov     r11, #0 ; 0x0
0x0000bb70 <nfq_update+44>:       ldr     r1, [r9, #28]!	//3. r1 = [r9 + 28]，感叹号表示r9会更新为r9 + 28
0x0000bb74 <nfq_update+48>:       str     r1, [sp, #8]
0x0000bb78 <nfq_update+52>:       mov     r2, r1		//2. r2 = r1
0x0000bb7c <nfq_update+56>:       cmp     r9, r2
0x0000bb80 <nfq_update+60>:       beq     0xbbf0 <nfq_update+172>
0x0000bb84 <nfq_update+64>:       mov     r8, r2
0x0000bb88 <nfq_update+68>:       ldr     r2, [r2]		//1. 出错指令
0x0000bb8c <nfq_update+72>:       cmp     r10, #0 ; 0x0
0x0000bb90 <nfq_update+76>:       str     r2, [sp, #8]
0x0000bb94 <nfq_update+80>:       beq     0xbe10 <nfq_update+716>
...//省略无关
```
从反汇编看，出错指令是加载内存地址为r2的数据，并存入r2中。（小tips：大多数情况下这种操作是链表的x = x->next操作）。查看寄存器可以看到r2是0空地址，访问空地址导致段错误发生。
我们一步一步反推。先是找到 r2是从r1得到。继续分析可知r1是r9指向内存地址偏移28个字节的数据。而r9是sp栈指针指向内存地址偏移4个字节的数据。最终可知这个位置存放的是第一个入参nfq指针地址。

```powershell
(gdb) info reg
r0             0x891d8  561624
r1             0x0      0
r2             0x0      0
r3             0x100004 1048580
r4             0x0      0
r5             0x891d8  561624
r6             0xbefffbcc       3204447180
r7             0x3e7    999
r8             0x0      0
r9             0x891f4  561652
r10            0x0      0
r11            0x0      0
r12            0x863c4  549828
sp             0xbefff830       0xbefff830
lr             0xc72c   50988
pc             0xbb88   0xbb88 <nfq_update+68>
fps            0x0      0
cpsr           0x20000010       536870928
```
至此脉络基本清晰，顺着推下来， nfq结构偏移 28个字节指向的结构变量就是出错的变量。对照代码可知是
list_head List；因为在Create过程中异常进入clean up逻辑，nfq的List并未初始化，所以List的next和prev都是NULL。而在nfq_update中node = List->next, 然后在访问下一个节点时使用了node = node->next。产生了空指针引用，导致进程crash。
对于环形链表，我们最好在创建的同时就对其初始化，避免遗漏或者中间处理异常跳出，导致链表未初始化，在遍历的时候产生异常。
