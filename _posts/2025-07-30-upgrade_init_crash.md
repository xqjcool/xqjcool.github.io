---
title: "Kernel panic triggered by unexpected exit of the init process"
date: 2025-07-30
---

# init进程处理异常导致的系统crash

## 1. 问题现象

QA在升级版本image时，出现系统crash。异常信息如下：

```bash
[82062.787587] init[1]: segfault at 3ab8 ip 00007f8315dbc1e2 sp 00007ffde990f410 error 4 in libmbase.so[7f8315dbb000+17000] likely on CPU 1 (core 1, socket 0)
[82062.787916] Code: 0f 29 44 24 70 0f 29 84 24 80 00 00 00 e8 a6 f3 ff ff 85 c0 0f 88 1e 01 00 00 48 8b 2d cf bd 01 00 e8 72 f6 ff ff 48 8b 45 00 <f6> 80 b8 3a 00 00 02 75 2d 48 8b 84 24 18 01 00 00 64 48 2b 04 25
[82062.814934] Kernel panic - not syncing: Attempted to kill init! exitcode=0x00000000
[82062.815227] CPU: 1 PID: 1 Comm: init Kdump: loaded Tainted: G           O       6.1 #1
[82062.815398] Hardware name: VMware, Inc. VMware Virtual Platform/440BX Desktop Reference Platform, BIOS 6.00 09/21/2015
[82062.815611] Call Trace:
[82062.815678]  <TASK>
[82062.815764]  panic+0x13a/0x401
[82062.817381]  do_exit+0xaa6/0xb30
[82062.817464]  do_group_exit+0x97/0xa0
[82062.817560]  __x64_sys_exit_group+0x17/0x20
[82062.817706]  do_syscall_64+0x49/0xa0
[82062.817798]  ? exc_page_fault+0x6b/0xc0
[82062.817889]  entry_SYSCALL_64_after_hwframe+0x64/0xce
[82062.818001] RIP: 0033:0x7f8315af6295
[82062.818086] Code: 02 ba ff ff ff ff 89 d0 c3 66 2e 0f 1f 84 00 00 00 00 00 66 90 48 8b 35 71 db 0f 00 ba e7 00 00 00 eb 03 66 90 f4 89 d0 0f 05 <48> 3d 00 f0 ff ff 76 f3 f7 d8 64 89 06 eb ec 66 2e 0f 1f 84 00 00
[82062.818477] RSP: 002b:00007ffde990ef38 EFLAGS: 00000202 ORIG_RAX: 00000000000000e7
[82062.818651] RAX: ffffffffffffffda RBX: 00007f8315bf6188 RCX: 00007f8315af6295
[82062.818793] RDX: 00000000000000e7 RSI: ffffffffffffff78 RDI: 0000000000000000
[82062.818980] RBP: 0000000000000000 R08: 00007ffde990eed8 R09: 0000000000000007
[82062.819172] R10: 00007f8314c5c008 R11: 0000000000000202 R12: 0000000000000014
[82062.819361] R13: 0000000000000000 R14: 00007f8315bf4860 R15: 00007f8315bf61a0
[82062.819559]  </TASK>
```

## 2. 初步分析

只看异常栈没有什么有效信息，就是init进程异常，收到SIGSEGV信号。信号处理是 系统调用exit(0), 最后系统panic。
但前面一行日志非常重要。它告诉我们是init进程在执行 libmbase.so[7f8315dbb000+17000] 中 指令ip 00007f8315dbc1e2时，访问错误地址 3ab8，导致段错误。

offset = 0x11e2 = 0x7f8315dbc1e2 - 0x7f8315dbb000

用objdump反汇编 libmbase.so后，基地址时0x7000，偏移0x11e2 对应 0x81e2

```bash
0000000000008160 <print_err_msg>:
    8160:       41 57                   push   %r15
    8162:       66 0f ef c0             pxor   %xmm0,%xmm0
    8166:       41 56                   push   %r14
    8168:       41 55                   push   %r13
    816a:       41 54                   push   %r12
    816c:       55                      push   %rbp
    816d:       53                      push   %rbx
//省略无关
    81d2:       48 8b 2d cf bd 01 00    mov    0x1bdcf(%rip),%rbp        # 23fa8 <g_dbg_zone@Base>
    81d9:       e8 72 f6 ff ff          callq  7850 <config_init@plt>
    81de:       48 8b 45 00             mov    0x0(%rbp),%rax
    81e2:       f6 80 b8 3a 00 00 02    testb  $0x2,0x3ab8(%rax)
    81e9:       75 2d                   jne    8218 <print_err_msg+0xb8>
//省略无关
```

访问基于rax寄存器保存地址偏移0x3ab8后指向的内容。因为出问题时的地址就是0x3ab8， 所以可以得知rax寄存器当时是0。
也就是说当时全局变量 g_gdb_zone 指针为NULL。

## 3. 继续分析

对比代码，发现是在一个flag判断时。

```c
if (g_gdb_zone->flags & 2) {
  //some error log
}

```
因为 g_gdb_zone 这个全局变量，在启动时就初始化了，整个生命周期都是OK的。除了退出时，会主动释放相关内存，并置 g_gdb_zone = NULL。

而升级image在最后会退出，释放内存。导致这里存在g_gdb_zone为NULL的风险。


