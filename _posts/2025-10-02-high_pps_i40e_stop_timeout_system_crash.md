---
title: "System crash caused by i40e NIC stop timeout under high PPS traffic"
date: 2025-10-02
---

# 高pps流量下i40e网卡stop超时导致的系统crash

## 1. 问题现象

最近QA在做性能测试，没有关闭流量的情况下，对测试设备进行了重启。刚重启没多久就出现了crash，重启后又crash。直到停止流量后，系统才正常启动。

当系统正常启动后，再开启流量测试，则工作正常。或者流量不够大，系统启动也不会crash。

也就是说问题的触发满足两个条件。
1. 高PPS流量
2. 系统刚启动

## 2. 初步分析

取coredump分析

```bash
       PANIC: "general protection fault: 0000 [#1] SMP NOPTI"
         PID: 2815
     COMMAND: "ifconfig"
        TASK: ffffa15a928af000  [THREAD_INFO: ffffa15a928af000]
         CPU: 5
       STATE: TASK_RUNNING (PANIC)

```

"general protection fault: 0000 [#1] SMP NOPTI" 这种错误一般都是 异常地址访问触发访问保护。我们查看对应的异常地址

```bash
crash> bt
PID: 2815     TASK: ffffa15a928af000  CPU: 5    COMMAND: "ifconfig"
 #0 [ffffa44b5bc4fba8] machine_kexec at ffffffff9743e54c
 #1 [ffffa44b5bc4fbf8] __crash_kexec at ffffffff974cbd3b
 #2 [ffffa44b5bc4fcd0] oops_end at ffffffff9741c009
 #3 [ffffa44b5bc4fcf8] die at ffffffff9741c083
 #4 [ffffa44b5bc4fd28] do_general_protection at ffffffff9741a45e
 #5 [ffffa44b5bc4fd50] general_protection at ffffffff97e01325
    [exception RIP: acct_collect+92]
    RIP: ffffffff974cb68c  RSP: ffffa44b5bc4fe08  RFLAGS: 00010202
    RAX: 004500087d7dca0b  RBX: ffffa15a92b28400  RCX: 0000000000000000
    RDX: 90003f1058ecf300  RSI: 0000000000000001  RDI: ffffa15a75a90d40
    RBP: ffffa44b5bc4fe28   R8: 0000000000000000   R9: 00007f8e150a58d8
    R10: 0000000000000000  R11: 0000000000000000  R12: 0000000000000000
    R13: 0000000000000000  R14: ffffa15a928af000  R15: 0000000000000000
    ORIG_RAX: ffffffffffffffff  CS: 0010  SS: 0018
 #6 [ffffa44b5bc4fe30] do_exit at ffffffff97456837
 #7 [ffffa44b5bc4fea0] do_group_exit at ffffffff974571db
 #8 [ffffa44b5bc4fed0] sys_exit_group at ffffffff97457264
 #9 [ffffa44b5bc4fee0] do_syscall_64 at ffffffff974016d5
#10 [ffffa44b5bc4ff50] entry_SYSCALL_64_after_hwframe at ffffffff97e0008d

crash> dis -l acct_collect+92
/root/linux-4.14.336/kernel/acct.c: 547
0xffffffff974cb68c <acct_collect+92>:   add    0x8(%rax),%rdx    //访问rax寄存器

```

因为 rax = 004500087d7dca0b 不是一个正常的地址，所以导致 "general protection fault"。
那么rax是什么，从哪里来？

<img width="289" height="155" alt="image" src="https://github.com/user-attachments/assets/ecb57906-de2a-4b22-8c57-fbfd0147e116" />

```bash
crash> dis -r acct_collect+92
0xffffffff974cb630 <acct_collect>:      nopl   0x0(%rax,%rax,1) [FTRACE NOP]
0xffffffff974cb635 <acct_collect+5>:    push   %rbp
0xffffffff974cb636 <acct_collect+6>:    mov    %rsp,%rbp
0xffffffff974cb639 <acct_collect+9>:    push   %r14
0xffffffff974cb63b <acct_collect+11>:   push   %r13
0xffffffff974cb63d <acct_collect+13>:   mov    %gs:0x19dc0,%r14    //current指向的当前task_struct
0xffffffff974cb646 <acct_collect+22>:   push   %r12
0xffffffff974cb648 <acct_collect+24>:   mov    %rdi,%r12
0xffffffff974cb64b <acct_collect+27>:   push   %rbx
0xffffffff974cb64c <acct_collect+28>:   mov    0x650(%r14),%rbx
0xffffffff974cb653 <acct_collect+35>:   test   %esi,%esi
0xffffffff974cb655 <acct_collect+37>:   je     0xffffffff974cb7a2 <acct_collect+370>
0xffffffff974cb65b <acct_collect+43>:   xor    %r13d,%r13d
0xffffffff974cb65e <acct_collect+46>:   cmpq   $0x0,0x3a0(%r14)
0xffffffff974cb666 <acct_collect+54>:   je     0xffffffff974cb6bc <acct_collect+140>
0xffffffff974cb668 <acct_collect+56>:   mov    0x3a0(%r14),%rax
0xffffffff974cb66f <acct_collect+63>:   lea    0x80(%rax),%rdi
0xffffffff974cb676 <acct_collect+70>:   call   0xffffffff97cde3e0 <down_read>
0xffffffff974cb67b <acct_collect+75>:   mov    0x3a0(%r14),%rax  //task_struct.mm
0xffffffff974cb682 <acct_collect+82>:   mov    (%rax),%rax
0xffffffff974cb685 <acct_collect+85>:   test   %rax,%rax
0xffffffff974cb688 <acct_collect+88>:   je     0xffffffff974cb6a3 <acct_collect+115>
0xffffffff974cb68a <acct_collect+90>:   xor    %edx,%edx
0xffffffff974cb68c <acct_collect+92>:   add    0x8(%rax),%rdx

crash> task_struct.mm -ox
struct task_struct {
   [0x3a0] struct mm_struct *mm;
}
crash> mm_struct.mmap -ox
struct mm_struct {
    [0x0] struct vm_area_struct *mmap;
}

```

配合代码和反汇编，我们知道 rax是 vm_area_struct *vma指针。

```bash
crash> task_struct.mm -x ffffa15a928af000
  mm = 0xffffa15a75a90cc0,
crash> mm_struct.mmap -x 0xffffa15a75a90cc0
  mmap = 0xffffa15a4533c2e0
crash> vm_area_struct.vm_next -ox
struct vm_area_struct {
  [0x10] struct vm_area_struct *vm_next;
}
crash> list -o 0x10 0xffffa15a4533c2e0  //循环查看所有vm_area_struct
ffffa15a4533c2e0
ffffa15a4533c228
ffffa15a4533c170
ffffa15a4533c0b8
4500087d7dca0b      //出问题的vm_area_struct
list: invalid kernel virtual address: 4500087d7dca1b  type: "list entry"

```

可以看到其中 vm_area_struct ffffa15a4533c0b8 的 vm_next是个异常的值， 导致while (vma) 遍历操作访问 vma->end时出错。

```bash

```
