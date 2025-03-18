---
title: "[Dead Loop] [Mac OS] Program Infinite Loop Issue Caused by Clang Compiler Optimization"
date: 2020-03-13
---

mac客户端之前调试都运行正常，最近正是编译后，运行正常，但是无法停止。
经过定位（此种曲折），发现是一个while ((c = a - b) <= 0) {} 逻辑进入死循环。
示例代码如下：

```c
#include <stdio.h>
#include <stdint.h>
#include <time.h>

int32_t __attribute__((noinline)) time_positive()
{
    static int32_t base_time = 0;
    if (base_time == 0) base_time = time(NULL) - 16;
    return (int32_t)time(NULL) - base_time;
}

int32_t __attribute__((noinline)) inc_7fffffff(int32_t v)
{
    return v + 0x7fffffff;
}

int main()
{
    int32_t current_time = time_positive();
    int32_t next_time = current_time;
    int32_t diff_time = 0;

    printf("before while-loop\n");
    while ((diff_time = next_time - current_time) <= 0)
    {
        current_time = time_positive();
        next_time = inc_7fffffff(current_time);
        printf("n=%d, c=%d, d=%d\n", next_time, current_time, next_time - current_time);
    }
    printf("after while-loop (diff_time=%d)\n", diff_time);
    return 0;
}
```
编译执行结果如下：

```powershell
$ clang test-int-ring.c -O1
$ ./a.out 
before while-loop
n=-2147483633, c=16, d=2147483647
n=-2147483633, c=16, d=2147483647
n=-2147483633, c=16, d=2147483647
n=-2147483633, c=16, d=2147483647
n=-2147483633, c=16, d=2147483647
n=-2147483633, c=16, d=2147483647
n=-2147483633, c=16, d=2147483647
n=-2147483633, c=16, d=2147483647
n=-2147483633, c=16, d=2147483647
n=-2147483633, c=16, d=2147483647
n=-2147483633, c=16, d=2147483647
...//无限循环，省略
```
明明diff_time = next_time - current_time 是2147483647 ，却无法跳出循环。
分析代码反汇编，我们找到判断逻辑部分：

```powershell
100000e90: 55  pushq %rbp
100000e91: 48 89 e5  movq %rsp, %rbp
100000e94: 41 57  pushq %r15
100000e96: 41 56  pushq %r14
100000e98: 53  pushq %rbx
100000e99: 50  pushq %rax
100000e9a: e8 b1 ff ff ff  callq -79 <_time_positive>
100000e9f: 48 8d 3d ea 00 00 00  leaq 234(%rip), %rdi
100000ea6: e8 63 00 00 00  callq 99 <dyld_stub_binder+0x100000f0e>
100000eab: 4c 8d 35 9e 00 00 00  leaq 158(%rip), %r14
100000eb2: 66 2e 0f 1f 84 00 00 00 00 00  nopw %cs:(%rax,%rax)
100000ebc: 0f 1f 40 00  nopl (%rax)
// 循环开始
100000ec0: e8 8b ff ff ff  callq -117 <_time_positive>
100000ec5: 41 89 c7  movl %eax, %r15d       //r15d 存放 current_time
100000ec8: 89 c7  movl %eax, %edi
100000eca: e8 b1 ff ff ff  callq -79 <_inc_7fffffff>
100000ecf: 89 c3  movl %eax, %ebx           //ebx 存放 next_time
100000ed1: 89 c1  movl %eax, %ecx
100000ed3: 44 29 f9  subl %r15d, %ecx
100000ed6: 31 c0  xorl %eax, %eax
100000ed8: 4c 89 f7  movq %r14, %rdi
100000edb: 89 de  movl %ebx, %esi
100000edd: 44 89 fa  movl %r15d, %edx
100000ee0: e8 23 00 00 00  callq 35 <dyld_stub_binder+0x100000f08>
100000ee5: 44 29 fb  subl %r15d, %ebx      //next_time - current_time
100000ee8: 7e d6  jle -42 <_main+0x30>     //跳转指令，注意这里
// 循环结束
100000eea: 48 8d 3d 71 00 00 00  leaq 113(%rip), %rdi
100000ef1: 31 c0  xorl %eax, %eax
100000ef3: 89 de  movl %ebx, %esi
100000ef5: e8 0e 00 00 00  callq 14 <dyld_stub_binder+0x100000f08>
100000efa: 31 c0  xorl %eax, %eax
100000efc: 48 83 c4 08  addq $8, %rsp
100000f00: 5b  popq %rbx
100000f01: 41 5e  popq %r14
100000f03: 41 5f  popq %r15
100000f05: 5d  popq %rbp
100000f06: c3  retq
```
观察循环体部分，主要指令我已经加注释，配合代码很容易理解。注意观察循环体最后两条指令。
next_time(-2147483633) - current_time(16)，有符号减法导致溢出，OF=1；符号位为0，SF=0；差值不为0，ZF=0。
jle的判断条件是 (SF ^ OF) | ZF，于是判断为真，继续循环下去。

=======================
开头我说过，调试的时候是OK的，后来知道是因为开了-g选项，禁止了编译优化。我们看一下不优化时的表现。

```powershell
$ clang test-int-ring.c 
$ ./a.out 
before while-loop
n=-2147483633, c=16, d=2147483647
after while-loop (diff_time=2147483647)
```
可以正常退出循环。那么区别在哪里呢？我们看一下反汇编代码：

```powershell
100000ea0: 55  pushq %rbp
100000ea1: 48 89 e5  movq %rsp, %rbp
100000ea4: 48 83 ec 20  subq $32, %rsp
100000ea8: c7 45 fc 00 00 00 00  movl $0, -4(%rbp)
100000eaf: e8 8c ff ff ff  callq -116 <_time_positive>
100000eb4: 89 45 f8  movl %eax, -8(%rbp)          // -8(%rbp) 存放 current_time
100000eb7: 8b 45 f8  movl -8(%rbp), %eax
100000eba: 89 45 f4  movl %eax, -12(%rbp)         // -12(%rbp) 存放 next_time
00000ebd: c7 45 f0 00 00 00 00  movl $0, -16(%rbp)
100000ec4: 48 8d 3d a1 00 00 00  leaq 161(%rip), %rdi
100000ecb: b0 00  movb $0, %al
100000ecd: e8 6a 00 00 00  callq 106 <dyld_stub_binder+0x100000f3c>
100000ed2: 89 45 ec  movl %eax, -20(%rbp)
//循环开始
100000ed5: 8b 45 f4  movl -12(%rbp), %eax
100000ed8: 2b 45 f8  subl -8(%rbp), %eax          // diff_time = next_time - current_time
100000edb: 89 45 f0  movl %eax, -16(%rbp)         // -16(%rbp) 存放 diff_time
100000ede: 83 f8 00  cmpl $0, %eax                // 比较 diff_time 和 0
100000ee1: 0f 8f 37 00 00 00  jg 55 <_main+0x7e>  // 大于0 则跳出循环
100000ee7: e8 54 ff ff ff  callq -172 <_time_positive>
100000eec: 89 45 f8  movl %eax, -8(%rbp)
100000eef: 8b 7d f8  movl -8(%rbp), %edi
100000ef2: e8 89 ff ff ff  callq -119 <_inc_7fffffff>
100000ef7: 89 45 f4  movl %eax, -12(%rbp)
100000efa: 8b 75 f4  movl -12(%rbp), %esi
100000efd: 8b 55 f8  movl -8(%rbp), %edx
100000f00: 8b 45 f4  movl -12(%rbp), %eax
100000f03: 2b 45 f8  subl -8(%rbp), %eax
100000f06: 48 8d 3d 72 00 00 00  leaq 114(%rip), %rdi
100000f0d: 89 c1  movl %eax, %ecx
100000f0f: b0 00  movb $0, %al
100000f11: e8 26 00 00 00  callq 38 <dyld_stub_binder+0x100000f3c>
100000f16: 89 45 e8  movl %eax, -24(%rbp)
100000f19: e9 b7 ff ff ff  jmp -73 <_main+0x35>
//循环结束
100000f1e: 8b 75 f0  movl -16(%rbp), %esi
100000f21: 48 8d 3d 69 00 00 00  leaq 105(%rip), %rdi
100000f28: b0 00  movb $0, %al
100000f2a: e8 0d 00 00 00  callq 13 <dyld_stub_binder+0x100000f3c>
100000f2f: 31 f6  xorl %esi, %esi
100000f31: 89 45 e4  movl %eax, -28(%rbp)
100000f34: 89 f0  movl %esi, %eax
100000f36: 48 83 c4 20  addq $32, %rsp
100000f3a: 5d  popq %rbp
100000f3b: c3  retq
```
从汇编代码上可以看出，未优化的汇编更庞大，更接近c语言代码的逻辑格式，自然也更易于理解。
从循环开头4行我们可以看出，这次的判断是cmp $0, %eax，第一次判断时 diff_time = 0，cmp结果，不满足跳出判断jg，所以进入循环；第二次判断时diff_time = 2147483647，满足循环跳出判断jg，所以跳出循环，正常退出。

=================================
现在我们知道代码优化会影响存在溢出回绕的算术运算，如果不优化，整体代码运行性能又会大大下降。如何解决这种两难选择呢？
答案就是使用 -fwrapv， 使用这个编译项后，编译器会假设有符号的算数运算可能溢出，所以不会对这部分进行优化。其余部分仍然正常优化。
关于这个选项，GNU有如下解释：

```powershell
-ftrapv
This option generates traps for signed overflow on addition, subtraction, multiplication operations. 
-fwrapv
This option instructs the compiler to assume that signed arithmetic overflow of addition, subtraction and multiplication wraps around using twos-complement representation. This flag enables some optimizations and disables others.
```
简单来说：
1. 这两个编译项都会影响到编译器的正常优化。除非你有特殊需求，否则尽量不要使用这两个选项。
2. -ftrapv  当有符号算术运算导致溢出时，通过trap异常，使程序异常退出。主要用于调试目的。
3. -fwrapv 假设有符号算术运算的溢出会导致回绕，开启该选项后，不对相关运算进行优化，避免文章上面所描述的问题出现。
