---
title: "[softlockup Crash Analysis] Deadloop Caused by Business Logic Anomaly"
date: 2020-08-12
---
公司产品在线上运行中出现了softlockup导致的crash。

```bash
crash> bt
PID: 23     TASK: ffff8801694a4e70  CPU: 3   COMMAND: "ksoftirqd/3"
 #0 [ffff88102d2c3cf0] machine_kexec at ffffffff81059beb
 #1 [ffff88102d2c3d50] __crash_kexec at ffffffff81105822
 #2 [ffff88102d2c3e20] panic at ffffffff81680541
 #3 [ffff88102d2c3ea0] watchdog_timer_fn at ffffffff8112f454
 #4 [ffff88102d2c3ed8] __hrtimer_run_queues at ffffffff810b4d72
 #5 [ffff88102d2c3f30] hrtimer_interrupt at ffffffff810b5310
 #6 [ffff88102d2c3f80] local_apic_timer_interrupt at ffffffff81051037
 #7 [ffff88102d2c3f98] smp_apic_timer_interrupt at ffffffff81699f0f
 #8 [ffff88102d2c3fb0] apic_timer_interrupt at ffffffff8169845d
--- <IRQ stack> ---
 #9 [ffff8801694cf758] apic_timer_interrupt at ffffffff8169845d
    [exception RIP: BSN_ReleasePkt]
    RIP: ffffffffa04dca10  RSP: ffff8801694cf800  RFLAGS: 00000246
    RAX: ffffc9006fcb4008  RBX: ffff880e8cb61090  RCX: 0000000070cd6caa
    RDX: 0000000000000000  RSI: ffffea003a32d840  RDI: ffffc9006ff2cae0
    RBP: ffff8801694cf820   R8: ffff880e8cb61090   R9: 000000010038001f
    R10: 000000008cb61301  R11: ffffea003a32d840  R12: 000000008cb61301
    R13: ffffea003a32d840  R14: ffffffff811dcedb  R15: ffff8801694cf7f8
    ORIG_RAX: ffffffffffffff10  CS: 0010  SS: 0018
#10 [ffff8801694cf800] _BSN_ResetTcp at ffffffffa04e2bd4 [business]
#11 [ffff8801694cf828] BSN_OnDestroyFlow at ffffffffa04ec250 [business]
#12 [ffff8801694cf8e8] _BSN_DestroyFlow at ffffffffa04d745d [business]
//...省略部分
```

看起来不像是锁竞争导致的死锁，那就是中断中执行时间过长导致的假死锁问题。

一般执行时间过长都是循环处理中出现的。于是查找 _BSN_ResetTcp 调用 BSN_ReleasePkt的位置附近是不是存在循环处理。

```bash
crash> dis  ffffffffa04e2bd4
0xffffffffa04e2bd4 <_APX_ETcpResetTcpLink+436>: mov    0xc0(%rbx),%rdi
crash> dis _APX_ETcpResetTcpLink
//...省略部分
//循环体内容
0xffffffffa04e2b78 <_BSN_ResetTcp+344>: mov    0x68(%rdi),%rdx	//得到BSN_PACKET->Prev
0xffffffffa04e2b7c <_BSN_ResetTcp+348>: mov    %rdx,0xc0(%rbx)	//BSN_CONN->Tail = 
0xffffffffa04e2b83 <_BSN_ResetTcp+355>: mov    0x48(%rdi),%rdx
0xffffffffa04e2b87 <_BSN_ResetTcp+359>: test   %rdx,%rdx
0xffffffffa04e2b8a <_BSN_ResetTcp+362>: je     0xffffffffa04e2ba7 <_BSN_ResetTcp+391>
0xffffffffa04e2b8c <_BSN_ResetTcp+364>: mov    0x50(%rdi),%rcx
0xffffffffa04e2b90 <_BSN_ResetTcp+368>: mov    %rdx,(%rcx)
0xffffffffa04e2b93 <_BSN_ResetTcp+371>: mov    %rcx,0x8(%rdx)
0xffffffffa04e2b97 <_BSN_ResetTcp+375>: movq   $0x0,0x48(%rdi)
0xffffffffa04e2b9f <_BSN_ResetTcp+383>: movq   $0x0,0x50(%rdi)
0xffffffffa04e2ba7 <_BSN_ResetTcp+391>: mov    0x38(%rdi),%rdx
0xffffffffa04e2bab <_BSN_ResetTcp+395>: test   %rdx,%rdx
0xffffffffa04e2bae <_BSN_ResetTcp+398>: je     0xffffffffa04e2bcf <_BSN_ResetTcp+431>
0xffffffffa04e2bb0 <_BSN_ResetTcp+400>: mov    0x40(%rdi),%rcx
0xffffffffa04e2bb4 <_BSN_ResetTcp+404>: mov    %rdx,(%rcx)
0xffffffffa04e2bb7 <_BSN_ResetTcp+407>: mov    %rcx,0x8(%rdx)
0xffffffffa04e2bbb <_BSN_ResetTcp+411>: andb   $0xf7,0x2f(%rdi)
0xffffffffa04e2bbf <_BSN_ResetTcp+415>: movq   $0x0,0x38(%rdi)
0xffffffffa04e2bc7 <_BSN_ResetTcp+423>: movq   $0x0,0x40(%rdi)
0xffffffffa04e2bcf <_BSN_ResetTcp+431>: callq  0xffffffffa04dca10 <BSN_ReleasePkt>	
0xffffffffa04e2bd4 <_BSN_ResetTcp+436>: mov    0xc0(%rbx),%rdi	//BSN_PACKET = BSN_CONN->Tail
0xffffffffa04e2bdb <_BSN_ResetTcp+443>: test   %rdi,%rdi
0xffffffffa04e2bde <_BSN_ResetTcp+446>: jne    0xffffffffa04e2b78 <_BSN_ResetTcp+344>
//...省略部分

```

从汇编上看，这是BSN_CONN的Tail取到BSN_PACKET，并将BSN_PACKET的Prev赋予BSN_CONN的Tail。释放BSN_PACKET后，再从BSN_CONN的Tail取到BSN_PACKET。如此循环，直到BSN_CONN的Tail为NULL，即BSN_PACKET的Prev为NULL时跳出循环。

```bash
//BSN_PACKET = rdi = ffffc9006ff2cae0
crash> BSN_PACKET ffffc9006ff2cae0
struct BSN_PACKET {
//...省略无关 
  Next = 0xffffc9006ff2cae0, 
  Prev = 0xffffc9006ff2cae0	//Prev指向自己，导致每次循环都是在处理该PACKET
}
```

查看BSN_PACKET发现，该packet的Prev指向自己，导致在循环过程中，每次都获取到自己，产生了死循环。最后导致系统认定softlockup。

