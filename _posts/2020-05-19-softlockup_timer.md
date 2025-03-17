---
title: "[Softlockup Crash Analysis] A deadlock analysis related to a timer"
date: 2020-05-19
---
设备crash，提示 PANIC: "Kernel panic - not syncing: softlockup: hung tasks" 。遇到这个要先确认是真死锁还是软中断执行时间过长被误判死锁。

有个简单的办法区分，就是如果异常栈crash在锁等待，那么就是真死锁；如果crash在其他地方，大概率就是执行时间过长被误判死锁。

查看异常栈：

```c
PID: 0      TASK: ffffffff819c5460  CPU: 0   COMMAND: "swapper/0"
 #0 [ffff88007fc03978] machine_kexec at ffffffff81059beb
 #1 [ffff88007fc039d8] __crash_kexec at ffffffff81105822
 #2 [ffff88007fc03aa8] panic at ffffffff81680541
 #3 [ffff88007fc03b28] watchdog_timer_fn at ffffffff8112f454
 #4 [ffff88007fc03b60] __hrtimer_run_queues at ffffffff810b4d72
 #5 [ffff88007fc03bb8] hrtimer_interrupt at ffffffff810b5310
 #6 [ffff88007fc03c08] local_apic_timer_interrupt at ffffffff81051037
 #7 [ffff88007fc03c20] smp_apic_timer_interrupt at ffffffff81699f0f
 #8 [ffff88007fc03c38] apic_timer_interrupt at ffffffff8169845d
 #9 [ffff88007fc03cd8] Test_StateUpdate at ffffffffa04552f6 [testmod]	//上锁失败spin_lock(&testMod->Lock);
#10 [ffff88007fc03dc0] Test_StateTimerFn at ffffffffa04568a9 [testmod]
#11 [ffff88007fc03df8] _Test_TimerFunc at ffffffffa04a7f99 [testmod]
#12 [ffff88007fc03e28] call_timer_fn at ffffffff81095eb6
#13 [ffff88007fc03e60] run_timer_softirq at ffffffff81098ba7  //spin_lock_irq(&base->lock);
#14 [ffff88007fc03ed8] __do_softirq at ffffffff8108f63f
#15 [ffff88007fc03f48] call_softirq at ffffffff8169929c
#16 [ffff88007fc03f60] do_softirq at ffffffff8102d365
#17 [ffff88007fc03f80] irq_exit at ffffffff8108f9d5
#18 [ffff88007fc03f98] smp_apic_timer_interrupt at ffffffff81699f15
#19 [ffff88007fc03fb0] apic_timer_interrupt at ffffffff8169845d

PID: 27921  TASK: ffff880069d32f10  CPU: 1   COMMAND: "rmmod"
 #0 [ffff88007fd05e48] crash_nmi_callback at ffffffff8104d342
 #1 [ffff88007fd05e58] nmi_handle at ffffffff816901b7
 #2 [ffff88007fd05eb0] do_nmi at ffffffff816903c3
 #3 [ffff88007fd05ef0] end_repeat_nmi at ffffffff8168f5d3
    [exception RIP: lock_timer_base+77]		//上锁失败spin_lock_irqsave(&base->lock, *flags);
    RIP: ffffffff8109702d  RSP: ffff88006256ba58  RFLAGS: 00000246
    RAX: 0000000000000000  RBX: 0000000000000000  RCX: 0000000000000006
    RDX: 0000000000800000  RSI: ffff88006256ba88  RDI: ffff8800684b72b8
    RBP: ffff88006256ba78   R8: 0000000000000086   R9: 000000000002afe6
    R10: 4e4f43574c3e6f66  R11: 3a307b6e6e6f434e  R12: 0000000000000000
    R13: ffff8800684b72b8  R14: ffff88006256ba88  R15: 0000000000000004
    ORIG_RAX: ffffffffffffffff  CS: 0010  SS: 0018
--- <NMI exception stack> ---
 #4 [ffff88006256ba58] lock_timer_base at ffffffff8109702d
 #5 [ffff88006256ba80] try_to_del_timer_sync at ffffffff8109858f
 #6 [ffff88006256bab0] del_timer_sync at ffffffff81098642
 #7 [ffff88006256bac8] Test_TimerDelSync at ffffffffa04a81c2 [testmod]
 #8 [ffff88006256bad8] _Test_StateNotify at ffffffffa0455d16 [testmod]	//spin_lock_bh(&testMod->Lock);
 #9 [ffff88006256bb78] Test_StateNotify at ffffffffa04564f3 [testmod]
 //...省略无关栈
#24 [ffff88006256bee8] cleanup_module at ffffffffa042c8eb [testmod]
#25 [ffff88006256bef8] sys_delete_module at ffffffff810fe3db
#26 [ffff88006256bf80] system_call_fastpath at ffffffff81697809
```

这是一个典型的 X（Hold A）等待B ，Y（Hold B）等待A 问题。
CPU0 获取了 timer的 base->lock，然后获取 testMod->Lock失败，cpu忙等。
CPU1 获取了 testMod->Lock，然后获取base->lock失败，cpu忙等。
谁也无法获取到锁，于是形成死锁。

对于这类死锁定位，需要先找到每个cpu在等待的锁。然后走查调用栈，查看是否有调用已经持有该锁，如果存在则就是死锁原因。
