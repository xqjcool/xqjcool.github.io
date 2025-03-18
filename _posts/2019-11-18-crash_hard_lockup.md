---
title: "[crash analysis]Kernel panic - not syncing: Hard LOCKUP"
date: 2019-11-18
---

客户上报设备crash问题，取得coredump文件后，用crash工具打开，分析异常调用栈。

```c
      KERNEL: /usr/lib/debug/lib/modules/3.10.0-514.26.2.el7.x86_64/vmlinux
    DUMPFILE: vmcore  [PARTIAL DUMP]
        CPUS: 2
        DATE: Sun Nov 17 05:56:07 2019
      UPTIME: 1 days, 00:22:05
LOAD AVERAGE: 0.00, 0.01, 0.05
       TASKS: 120
    NODENAME: cpe
     RELEASE: 3.10.0-514.26.2.el7.x86_64
     VERSION: #1 SMP Tue Jul 4 15:04:05 UTC 2017
     MACHINE: x86_64  (1800 Mhz)
      MEMORY: 2 GB
       PANIC: "BUG: unable to handle kernel "
         PID: 0
     COMMAND: "swapper/1"
        TASK: ffff88007b889f60  (1 of 2)  [THREAD_INFO: ffff88007b89c000]
         CPU: 1
       STATE: TASK_RUNNING (PANIC)
 
crash> bt
PID: 0      TASK: ffff88007b889f60  CPU: 1   COMMAND: "swapper/1"
 #0 [ffff88007f2859f0] machine_kexec at ffffffff81059beb
 #1 [ffff88007f285a50] __crash_kexec at ffffffff81105822
 #2 [ffff88007f285b20] panic at ffffffff81680541
 #3 [ffff88007f285ba0] nmi_panic at ffffffff81085abf
 #4 [ffff88007f285bb0] watchdog_overflow_callback at ffffffff8112f879
 #5 [ffff88007f285bc8] __perf_event_overflow at ffffffff81174d2e
 #6 [ffff88007f285c00] perf_event_overflow at ffffffff81175974
 #7 [ffff88007f285c10] intel_pmu_handle_irq at ffffffff81009d88
 #8 [ffff88007f285e38] perf_event_nmi_handler at ffffffff8168ed6b
 #9 [ffff88007f285e58] nmi_handle at ffffffff816901b7
#10 [ffff88007f285eb0] do_nmi at ffffffff81690463
#11 [ffff88007f285ef0] end_repeat_nmi at ffffffff8168f5d3
    [exception RIP: _raw_spin_lock_irqsave+71]
    RIP: ffffffff8168eb47  RSP: ffff88007f2839a8  RFLAGS: 00000002
    RAX: 000000000000193f  RBX: ffff88007b90c000  RCX: 000000000000f6c4
    RDX: 000000000000f6c2  RSI: 000000000000f6c4  RDI: ffff88007b90c000
    RBP: ffff88007f2839a8   R8: 0000000000000086   R9: 0000000000000000
    R10: 0000000000000000  R11: ffff88007f2839fe  R12: ffff88007b90c000
    R13: ffff8800364228f0  R14: ffff88007f2839e8  R15: 0000000000000e20
    ORIG_RAX: ffffffffffffffff  CS: 0010  SS: 0018
--- <NMI exception stack> ---
#12 [ffff88007f2839a8] _raw_spin_lock_irqsave at ffffffff8168eb47
#13 [ffff88007f2839b0] lock_timer_base at ffffffff8109700b
#14 [ffff88007f2839e0] mod_timer at ffffffff81099234
#15 [ffff88007f283a28] add_timer at ffffffff810993e8
#16 [ffff88007f283a38] fbcon_add_cursor_timer at ffffffff81381a69
#17 [ffff88007f283a60] fbcon_cursor at ffffffff81384c2a
#18 [ffff88007f283ab0] hide_cursor at ffffffff813f70b8
#19 [ffff88007f283ac8] vt_console_print at ffffffff813f8ae8
#20 [ffff88007f283b30] call_console_drivers.constprop.17 at ffffffff81087011
#21 [ffff88007f283b58] console_unlock at ffffffff810888b8
#22 [ffff88007f283b98] vprintk_emit at ffffffff81088df4
#23 [ffff88007f283c08] vprintk_default at ffffffff81089169
#24 [ffff88007f283c18] printk at ffffffff816806db
#25 [ffff88007f283c78] no_context at ffffffff8167fbbe
#26 [ffff88007f283cc8] __bad_area_nosemaphore at ffffffff8167fd2c
#27 [ffff88007f283d10] bad_area_nosemaphore at ffffffff8167fe96
#28 [ffff88007f283d20] __do_page_fault at ffffffff81692e4e
#29 [ffff88007f283d80] do_page_fault at ffffffff81692ff5
#30 [ffff88007f283db0] page_fault at ffffffff8168f208
    [exception RIP: run_timer_softirq+248]
    RIP: ffffffff81098a68  RSP: ffff88007f283e68  RFLAGS: 00010082
    RAX: ffff88007b90cbf8  RBX: ffff88007b90c000  RCX: ffffffffa88b53b0
    RDX: ffff88007b90cbd0  RSI: ffff88007f283e90  RDI: ffff88007b90c000
    RBP: ffff88007f283ed0   R8: 00004fc93c9ddf00   R9: ffff88007f283d98
    R10: 0000000000000002  R11: ffff88007f283da0  R12: 00000000000000bd
    R13: 00004fc93c9643c1  R14: ffffffff819b70c8  R15: 0000000000000041
    ORIG_RAX: ffffffffffffffff  CS: 0010  SS: 0018
#31 [ffff88007f283e60] run_timer_softirq at ffffffff810989b3
#32 [ffff88007f283ed8] __do_softirq at ffffffff8108f63f
#33 [ffff88007f283f48] call_softirq at ffffffff8169929c
#34 [ffff88007f283f60] do_softirq at ffffffff8102d365
#35 [ffff88007f283f80] irq_exit at ffffffff8108f9d5
#36 [ffff88007f283f98] smp_apic_timer_interrupt at ffffffff81699f15
#37 [ffff88007f283fb0] apic_timer_interrupt at ffffffff8169845d
--- <IRQ stack> ---
#38 [ffff88007b89fda8] apic_timer_interrupt at ffffffff8169845d
    [exception RIP: cpuidle_enter_state+82]
    RIP: ffffffff81514eb2  RSP: ffff88007b89fe50  RFLAGS: 00000206
    RAX: 00004fc93c9de33f  RBX: ffff880077e61fc8  RCX: 0000000000000019
    RDX: 0000000225c17d03  RSI: 74cf4e9d5c1a5c1a  RDI: 00004fc93c9de33f
    RBP: ffff88007b89fe78   R8: 00000000000002e6   R9: 0000000000000018
    R10: 0000000000000373  R11: 0000000000000000  R12: 00004fc8f34d8f80
    R13: ffffffff810d065c  R14: ffff88007b89fdf8  R15: ffffffff810c9bf8
    ORIG_RAX: ffffffffffffff10  CS: 0010  SS: 0018
#39 [ffff88007b89fe80] cpuidle_idle_call at ffffffff81514ff9
#40 [ffff88007b89fec0] arch_cpu_idle at ffffffff8103512e
#41 [ffff88007b89fed0] cpu_startup_entry at ffffffff810e82f5
#42 [ffff88007b89ff28] start_secondary at ffffffff8104f0da
```

这是卸载模块产生的crash，最近分析crash遇到了好几次类似现象。分析完后，总结了一些规律。

对于这种问题从vmcore-dmesg上看是 Hard LOCKUP，也就是在硬中断中出现死锁了。

通过分析异常栈，我们看到在软中断中执行FRAME#31 run_timer_softirq 过程中已经锁住了&base->lock（注：通过spin_lock_irq(&base->lock)）。然后在FRAME#13 lock_timer_base 中再次对  &base->lock 上锁，导致死锁发生，最终crash。

但这只是表面现象。

我们看到FRAME#31 run_timer_softirq 虽然禁止硬中断并上锁，但是还是被page_fault 打断了。我们查看导致问题的指令

0xffffffff81098a68 <run_timer_softirq+248>:     mov    %rsi,0x8(%rcx)

查看 rcx的值，提示非法内核虚拟地址。也就是说run_timer_softirq执行过程中访问了非法的内核虚拟地址。

crash> rd ffffffffa88b53b0
rd: invalid kernel virtual address: ffffffffa88b53b0  type: "64-bit KVADDR"



run_timer_softirq 执行一系列timer 处理，而timer都是 内核本身和模块注册的。加载模块，使用的就是内核虚拟地址，此地址很可能是问题模块的虚拟地址，因为模块释放了。等到模块注册的timer执行时，访问模块地址非法，导致问题发生。

配合dmesg，定位到卸载模块时其中一个timer没有delete，最终模块释放了，timer执行时出错，系统crash。

====================我是分割线========================

出现这种问题的表现：

run_timer_softirq 处理过程中出现page_fault，且在手动执行卸载模块操作时出现。

导致这种问题出现的原因：

1. 卸载模块没有delete 相关timer， 这个属于初级错误一般不会犯。

2. 卸载模块delete timer后，被其他操作又添加了。 这个比较隐蔽，需要检查 delete timer后的操作有没有可能再将timer加进去。例如软中断。

3. 某个系列操作中间失败了，没有回退全部已执行操作。

例如，某个操作添加 timer， 执行B，结果执行B失败了，直接退出。 但是timer没有删除。导致卸载模块时，timer依然在运行中。
